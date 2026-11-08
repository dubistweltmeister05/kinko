[[RhyGen]]
**Scope:** This covers the DG application firmware (`App/dg`) on branch `feat/4G-integration`. It is the only firmware variant that uses the EC200 modem; `bibtya_1.2` sends nothing to the server.
## 1. Connection

| Item | Value |
|---|---|
| Transport | Cellular (Quectel EC200), APN `airtelgprs.com` |
| Protocol | MQTT over plain TCP, using the modem's built-in MQTT client |
| Broker | `52.66.89.140:1883`, the EC2 Elastic IP, hard-coded in firmware |
| Security | **None.** There is no TLS and no username or password. |
| Client ID | `LOG_<DEVICE_ID>`, for example `LOG_4B010EA4` |
| MQTT session options | The firmware sends no `AT+QMTCFG`, so keep-alive, clean session and protocol version are the modem defaults. |

`DEVICE_ID` is the CRC32 of the STM32's 96-bit factory UID, printed as 8 uppercase hex digits. It is stable per board and survives reflashing.

## 2. What Is Sent

- **Direction:** upstream only. The device subscribes to no topics, so the server cannot send it anything over MQTT.
- **One topic per device:** `logger/<DEVICE_ID>/log`
- **QoS 1, retain off.** Each publish gets a PUBACK from the broker.
- **Payload:** binary, little-endian, packed. Each message is a 16-byte header followed by N 64-byte records. The records are byte-for-byte what the device also writes to the SD card.
  
The J1939 values are sent raw, without scaling. The server applies the scaling, for example RPM = raw × 0.125.

## 3. Rate and Volume

| Metric                      | Value                                                         |
| --------------------------- | ------------------------------------------------------------- |
| Record rate                 | 10 Hz (one record every 100 ms)                               |
| Batch size                  | 20 records, 1,296 B payload                                   |
| Publish rate (steady state) | One publish every 2 s                                         |
| Partial-batch trigger       | Oldest queued record is 5 s old                               |
| Payload throughput          | ~650 B/s ≈ 2.3 MB/hour ≈ 56 MB/day per device                 |
| MQTT overhead per publish   | ~26 B up (header, 19-char topic, packet ID) + 4 B PUBACK down |

These figures cover the payload and MQTT framing only; TCP/IP and cellular overhead come on top. Because the device publishes every 2 s, keep-alive PINGs should rarely be sent.

## 4. Delivery Behaviour

- **Store and forward:** records stay in a 1,024-record RAM queue (about 100 s of data) until the broker PUBACKs them. After an outage the backlog is sent in back-to-back 20-record batches.
- **Data loss:** if an outage lasts longer than about 100 s, the oldest records are overwritten. They remain on the SD card. The loss shows up on the server as gaps in `first_seq` and as an increase in `dropped`.
- **Duplicates are possible.** If a PUBACK is lost, or the 60 s publish timeout fires after the broker already got the message, the same batch is resent. The server should deduplicate on `(device_id, seq)`.
- **Reconnects:** when the modem reports the link as lost, the device deactivates and reactivates the PDP context, then reopens the socket and reconnects with the same client ID. It does not send an MQTT DISCONNECT or close the old client first. The broker may therefore hold a stale session until keep-alive expires or the new CONNECT with the same client ID replaces it. If the socket open or CONNECT gets no response, the device retries after 15 s or 10 s respectively.
- **Connection refusals are not detected.** The firmware treats any successful CONNECT exchange as connected and ignores the CONNACK return code. If the broker starts refusing the device (for example after authentication is enabled), the device will keep trying to publish instead of reconnecting.
- **Startup:** the modem starts 20 s after boot, so the first data arrives on the order of 20–60 s after power-up.

## 5. Notes and Known Limitations

1. **There is no absolute time.** Timestamps are milliseconds since boot, and sequence numbers reset on every reboot. The server has to apply its own receive time to anchor the data.
2. **Electrical and mechanical data refresh at different rates.** Electrical values come from the multifunction meter over Modbus RTU, polled every 160 ms. Mechanical values come from J1939 broadcasts as they arrive. Records are taken every 100 ms, so neighbouring records can repeat the same electrical values.
3. **The connection is unencrypted and unauthenticated** on a public IP. Anyone who can reach port 1883 can publish to or read `logger/#`.
4. **SMS is not used for server traffic.** The driver can send and receive SMS, but nothing in the current build sends one.

## 6. Server-Side Software

Three Python scripts run on the server. Only the logger runs continuously; the other two are started by hand when we want to look at the data.

| Script                  | Role                                         | When it runs            |
| ----------------------- | -------------------------------------------- | ----------------------- |
| `mqtt_logger_sqlite.py` | Receives, decodes and stores all device data | Always (primary ingest) |
| `app_sqlite.py`         | Live dashboard of incoming data              | On demand               |
| `data_reader.py`        | Browsing, plotting and export of stored data | On demand               |

### 6.1 `mqtt_logger_sqlite.py` (receiver and decoder)

The logger subscribes to `logger/+/log` at QoS 1 through `mosquitto_sub`, so a single process receives data from every device. For each message it:

1. **Validates the batch.** It drops any message whose format version, record size or total length don't match the header.
2. **Removes duplicates.** It keeps track of the next expected sequence number for each device and discards records it has already stored. This handles the resends described in Section 4. If the sequence number and the device's clock both jump backwards, it treats that as a device reboot and starts tracking that device again.
3. **Converts timestamps to real time.** It works out an offset between the device's milliseconds-since-boot clock and the server's clock from when each batch arrives. That gives every record a wall-clock time even though the device has none. The offset is allowed to rise only by a small drift allowance, so batches that arrive late from a backlog don't skew it.
4. **Decodes and scales.** It unpacks each 64-byte record and applies the J1939 scaling (RPM, temperatures, pressures, fuel rate, engine hours) to produce engineering units.
5. **Stores the data.** It appends rows to a SQLite database for the current run. When that database reaches 50,000 rows (about 83 minutes per device at 10 Hz), it moves the rows into a Parquet file and empties the database.

The data is organised in partitioned folders:

```
mode=mqtt/device_id=<ID>/date=<YYYY-MM-DD>/run_id=<logger start time>/
    active_log.db          ← current rows (SQLite)
    telemetry/part-*.parquet ← archived rows
    _run_meta.json
```

The `run_id` comes from when the logger was started, so each restart of the logger begins a new run folder. The `date` is the server's receive date.

**Operational caveats**

- **Data sent while the logger is down is lost on the server.** The broker acknowledges the device's publish whether or not the logger is subscribed, and the subscription doesn't use a persistent session. Anything published while the logger isn't running isn't queued for it. That data is still on the device's SD card.
- **Duplicate tracking is held in memory,** so it resets when the logger restarts.

### 6.2 `app_sqlite.py` (live dashboard)

A Streamlit page that finds devices from the folder structure and shows the most recent run for one device at a time. It shows the newest 500 rows from `active_log.db`, refreshed every 2 s. It can also open archived Parquet files, archive the active database to Parquet on demand, and download the latest archive. It only reads the stored files; it doesn't connect to the broker.

### 6.3 `data_reader.py` (analysis tool)

A Streamlit tool for the archived Parquet data. It lets us choose a device, date and run, filter by time and column, plot channels against time, export any view to CSV, and optionally run SQL over the Parquet files through DuckDB. Like the dashboard, it reads only what the logger has already stored.

# Server Setup

## Compute
We have a t3.micro instance running in the ap-south-1 (Mumbai) region to serve our compute needs. It runs Ubuntu 26.04 LTS (Resolute) on an x86_64 architecture. The instance has been assigned an Elastic IP (52.66.89.140), which is hardcoded into the logger firmware as the publish target (making this configurable is a pending improvement).

The security group (launch-wizard-1) currently allows all inbound TCP traffic on all ports (0–65535) from 0.0.0.0/0, which covers port 1883 (MQTT) for logger communication. SSH access on port 22 is also open to the internet. All outbound traffic is unrestricted. Certificate-based authentication for the MQTT broker is not yet in place for the logger firmware.

## Storage
There are two EBS volumes attached to the instance: 
- Root volume (/dev/sda1): An 8 GB gp3 volume used as the OS disk. This is set to delete on instance termination.
- Data volume (/dev/sdf): A 50 GB gp3 volume that serves as hot storage for live, decoded logs. This volume is retained on instance termination, which is the correct configuration for persistent data.

Both volumes are currently unencrypted. To ease the strain on the t3.micro's limited RAM, a 4 GB swap partition is configured on the server, which helps when running visualization scripts.

We plan to eventually move data that is not immediately needed to an S3 bucket for cold/archival storage.
