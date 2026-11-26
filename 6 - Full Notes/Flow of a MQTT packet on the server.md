[[Rhygen Server]]

Here's the path one `logger/<ID>/log` message takes, from the broker to S3, with the file for each step.

## The flow

```
Logger ──MQTT QoS1──▶ Mosquitto (:1883)
                         │  (v1 loggers: old broker ──bridge──▶ same topics)
                         ▼
   ingester  on_message()                       __main__.py
      │
      ├─1─▶ Spool: append raw frame + fsync     spool.py, rhyb.py
      │        (only now is the PUBACK sent)
      │
      ├─2─▶ Processor.handle()                  processor.py
      │       ├─ wire.decode()  header/records  decoder/wire.py
      │       ├─ DeviceStream.process()          decoder/stream.py
      │       │    dedup, reboot, timestamps
      │       └─ client_row()  scaling → 25 cols decoder/scaling.py
      │
      ├─3─▶ LiveWriter → Valkey (right away)    live.py
      │
      └─4─▶ every 10 min: segment closes → Uploader → S3 staging   service.py, store.py
                 nightly: rhygen-compact → one file per device per day   compact.py
```

## Step by step

### 0. Receive

[**main**.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/__main__.py#L33) subscribes to `logger/+/log` and `logger/+/status` at QoS 1. It uses a persistent session (`clean_session=False`) and **manual acks**. [on_message](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/__main__.py#L122-L132) calls `ing.ingest(topic, payload)` and only then calls `client.ack()`. If writing to disk fails, the process stops without acking, so the broker keeps the message and sends it again later.

### 1. Spool to disk first, before any decoding

In [Ingester.ingest](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/service.py#L207-L221), your payload is wrapped in a `Frame(received_at_ms, topic, payload)`. [Spool.append](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/spool.py#L74-L82) writes it to disk and fsyncs it. The frame format is [.rhyb](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/rhyb.py): a 14-byte head (`<QHI>`: receive time in ms, topic length, payload length), then the topic, then **your bytes, unchanged**.

- There is one spool file per 10-minute window: `<window_ms>-<opened_ms>.rhyb`.
- Next to it is a `.state.json` checkpoint holding the decoder state (dedup position, clock offset, reboot count). After a crash, the ingester replays the spool and rebuilds exactly the same output.

The raw bytes are stored before anything tries to parse them. A malformed packet is never lost.

### 2. Decode ([processor.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/processor.py#L149-L209))

**a. Topic routing.** The topic must look like `logger/<8 hex>/log|status`. A `status` payload is parsed as JSON; the ingester keeps the top-level scalar fields and picks up `fw_version`, `firmware` or `fw` from them. Any other topic goes to dead letters.

**b. Wire parse** ([wire.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/decoder/src/rhygen_decoder/wire.py#L105-L128)). The header is `<IBBHII>` = `device_id, fmt_ver, rec_size, n_recs, first_seq, dropped`. The decoder rejects the packet if any of these hold:

- `fmt_ver` is not 1 or 2
- `rec_size` ≠ 64
- `n_recs` == 0
- `len(payload)` ≠ `16 + n_recs*64` exactly

**20 records is not required.** Any `n_recs ≥ 1` is accepted, as long as the length matches. A rejected packet becomes a dead letter, and its raw bytes are still kept.

Each record is unpacked with `<I8fB3xBII3H7B2x>`. The field offsets are listed in [RECORD_FIELDS](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/decoder/src/rhygen_decoder/wire.py#L27-L51). Note that byte 36 (`link_err_dup`), bytes 37–39 (padding) and byte 40 (`link_err`) follow your struct's padding. Only byte 40 is used.

**c. Device check.** The device ID comes from the **header**, not the topic. The topic's ID is checked against the header's only for devices marked `vnext` in `devices.yaml`. A mismatch there goes to dead letters.

**d. Per-device stream** ([stream.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/decoder/src/rhygen_decoder/stream.py#L83-L124)). This is ported from the client's script:

- **Dedup by `first_seq`:** if every record in the batch has already been seen (`first_seq + n ≤ next_seq`), the batch is dropped. If only some are new, the old ones are skipped. Each record's `seq` is `first_seq + i`.
- **Reboot detection:** `first_seq` goes backwards **and** the first record's `j1939_ts` is more than 120 s behind the last tick seen. The device state is then reset and `boot_count` goes up by one. The server counts reboots itself (`0x80000000 + reboots`), because the wire carries no boot counter.
- **Timestamps:**
    - **Format 2 with RTC synced** (bit 31 clear, not `0xFFFFFFFF`): `elec_ts` is ms since IST midnight. The date comes from the receive time; of the three candidate days around it, the closest one wins, which handles midnight.
    - **Otherwise** (format 1, or the RTC not synced yet): `ts = j1939_ts + offset`, where `offset = receive_time − last record's j1939_ts`. The offset can rise by at most 2 % of the elapsed ticks but can fall at once. This **assumes you publish right after the last record is queued.** Any queueing delay before the publish makes timestamps later than the truth until a faster delivery pulls the offset down.

**e. Scaling** ([scaling.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/decoder/src/rhygen_decoder/scaling.py#L54-L85)). `client_row()` builds the client's 25 columns with the same scale factors and rounding as the client's script (RPM ×0.125, coolant −40, oil temp ×0.03125−273, and so on). Then the processor adds `device_id, boot_count, seq, time_source, received_at_ms, source, fw_version, decoder_version`.

The `dropped` header field is parsed, but nothing downstream uses it at the moment.

### 3. Live data in Valkey, sent right away ([live.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/live.py))

- `dev:{id}:latest` (hash): the newest record.
- `dev:{id}:recent` (stream): the last 500 rows in arrival order.
- `dev:{id}:1s` (stream): 1-second averages, 15 minutes' worth.
- `dev:{id}:status` (hash): the latest status message.
- `fleet:last_seen` (sorted set): last receive time per device.

Valkey is only a cache: if it's down, the ingester logs it and keeps going. The dashboard reads this only through the live API.

### 4. S3, every 10 minutes and nightly ([store.py](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/store.py))

When a 10-minute window ends, the segment closes and the uploader writes:

```
staging/raw/device_id=<ID>/date=<IST date>/<HHMM-opened>.rhyb.zst     raw frames, zstd
staging/parquet/device_id=<ID>/date=<IST date>/<HHMM-opened>.parquet  decoded rows
deadletter/date=<d>/<chunk>.jsonl                                      bad packets as hex + reason
```

The spool file is deleted only after S3 accepts the upload; failed uploads are retried with backoff. Every night, [rhygen-compact](vscode-webview://1glco2hr15sn7qmgdd3bl6dfg7g6itjd7rqb66pjtcb6v0067ghj/ingester/src/rhygen_ingester/compact.py) merges each day's chunks into one `raw.rhyb.zst` (in the raw bucket) and one `parquet/.../data.parquet` per device. It checks the frame and row counts, then deletes the staging chunks. The partition date is the **receive** date in IST.

The raw files are never changed. If a decoding bug turns up, the Parquet files are rebuilt from the raw files.

## What matters on the firmware side

- `first_seq` must count **records**, not batches, and must keep increasing within a boot. Dedup and reboot detection depend on it.
- `j1939_ts` must be ms since boot in both formats; it's used as the boot tick.
- Publish soon after the last record is queued, otherwise format 1 and unsynced format 2 timestamps come out late.
- The header device ID decides where data is filed. The topic ID is cross-checked only for `vnext` devices.