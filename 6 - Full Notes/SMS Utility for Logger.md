[[RhyGen]]

![[SMS_Utility_Architecture.excalidraw]]

So, here's the deal.
We need a reliable method to update any configuration parameters in our logger's firmware, if there is ever a problem, or simply an update to any of them. And I am of the opinion that doing it over SMS is a better way than doing it via the EC2 instance itself - simply because SMS comms have a relatively simpler connection path to the MCU and the 4G Module than the EC2 uplink. So, what needs to be figured out, is - 

-> the parameters that we need to let us or the clients configure
-> the process of safely reconnecting via the new params
-> the format of the SMS 
-> implementing the parser for the SMS that shall be received, with genuine input validation for each field 

In that order

We have already configured the SMS to be routed straight to the MCU's UART instead of being stored in the SIM storage. We are also able to send an SMS, but are facing issues with the reception of the same. The last time I tried working on the same, I also made the mistake of sending RTF messages and not SMS messages. This is a common theme in most modern phones, and we'll do well to integrate support for that. I did later configure my SMS app to send actual SMS messages but it still didn't work. 

Also, I have a suspicion, that I was testing the wrong number, but that's unlikely - since I was able to send SMS messages from the MCU (via Quectel ) to my mobile number, and was replying to the same conversation (but was not able to receive anything on my MCU). 
## What needs to be configured, and what to do once we have new values
1. DataAPN - for when the user could change the SIM card that's inserted in the logger. The way this'll work is that they shall first send the new dataAPN, and shall then swap out the SIMs. There's a ton of things that can go wrong with this - the user being a dumbass and sending the updated params AFTER they swap out the SIM, and a bunch more thins, so we'll need solid fallback strategies for if and when that happens. 
2. MQTT Broker IP / Domain - This is to reconfigure the MQTT connection that is in place. For now, we are connecting to an EC2 instance, but maybe we need to switch brokers, or a new instance with as different EIP all together. 
3. Topic At which we are publishing - For now, it is logger/<device_id>/log. And it works well, but just in case that we need to change it, provisions need to be in place for the same. 

## What I need help with
1. Figuring out how to get SMS reception working on the MCU
2. Figuring out a format for the Configuration SMS
3. Figuring out how to have a parser for the received SMS
4. Figuring out how to proceed once we have new values for the config params. 
---
# Claude's Plan
## Plan: changing the logger's network config by SMS

Branch: `feat/Conf_via_SMS`, created by you off the current work.
## Context

The APN (`airtelgprs.com`) and broker (`52.66.89.140:1883`) are hardcoded string literals in [ec200_driver.c](App/common/Src/ec200_driver.c#L388) and [ec200_driver.c:432](App/common/Src/ec200_driver.c#L432). The topic `logger/<id>/log` is built into a 32-byte buffer in [app_main.c:264](App/dg/Src/app_main.c#L264). Changing any of them today means reflashing the firmware.

SMS is a better config channel than the MQTT uplink for two reasons:
- It needs only network registration, not a working PDP context or broker.
- That makes it the one channel that still works when the config itself is wrong, for example a wrong APN after a SIM swap.

**Where things stand:**
- Sending SMS works.
- Receiving does not. An FTDI sniff of the modem's TX line showed no `+CMT` at all. **The modem never emits it, so the parser in `ec200_uart_rx_cb` is not the cause yet.**

Each phase below has its own checkpoint, so you can stop after any one of them and still have something working.

---

## Phase 0 — Get SMS reception working (bench work, no code yet)

Use the FTDI and pass AT commands straight to the modem. You can uncomment `poll_debug_cli()` temporarily, or drive the modem directly. Work through these in order; each step narrows down where the SMS is lost.

1. **Is the SMS reaching the modem at all?** Send `AT+CNMI=2,1,0,0,0`, which stores incoming SMS instead of routing them. Send an SMS from your phone, then run `AT+CPMS?` and `AT+CMGL="ALL"`.
   - **Message listed:** the network delivers fine and the problem is routing. Go to step 2.
   - **Nothing listed:** the problem is the network or SIM. Go to step 3.
2. **Routing.** Run `AT+QURCCFG="urcport"`. If it reports `usbat`, `usbmodem` or anything other than the UART, unsolicited `+CMT` messages are going out on USB. Set `AT+QURCCFG="urcport","uart1"`; check the exact port name for your EC200 variant with `AT+QURCCFG=?`. Also check `AT+CSMS?`: if it returns service `1`, every `+CMT` must be acknowledged with `AT+CNMA`, so set `AT+CSMS=0`.
3. **Network or SIM.**
   - `AT+CNUM` and `AT+CREG?`: is the modem registered on the circuit-switched side? Many Indian networks deliver SMS over LTE through that path.
   - `AT+CEREG?` should show 1 or 5. Then run `AT+QCFG="ims"`.
   - **The likeliest culprit is the SIM itself.** IoT/M2M data SIMs often block incoming person-to-person SMS. Put the SIM in a phone and confirm it can receive a text from your phone.
   - If the SIM is Jio, it delivers SMS over IMS only. Enable IMS on the modem (`AT+QCFG="ims",1` plus the right MBN profile) and test again.
4. **RCS (what you called "RTF").** RCS chat (Google Messages "Chat") and iMessage travel over the internet and never reach a cellular modem, so there is nothing the MCU can support for them.
   - **What we can handle:** real SMS sent in Unicode (UCS-2) encoding. Phones switch to it as soon as one emoji or curly quote “ ” appears. Phase 3 decodes it.
   - **Operator guidance:** "turn off chat features" (Android) or "Send as Text Message" (iPhone).

**Checkpoint:** `+CMT:` plus the message body shows up on the FTDI at 115200 baud. Then make whichever AT setting fixed it permanent in the `EC200_SMS_Setup` state of [ec200_driver.c:320](App/common/Src/ec200_driver.c#L320).

**A debug output channel.** printf is stubbed out and its UART is taken. Two options:
- Retarget `_write` in [app_main.c:37](App/dg/Src/app_main.c#L37) to SWO/ITM (`ITM_SendChar`). The ST-Link shows it in the SWV console, and it costs no UART.
- Or keep the FTDI's RX pin only on the modem's TX line as a passive sniffer.

---

## Phase 1 — Parameters and persistent storage

### Parameters

| Key | Field | Default (today's value) | Notes |
|---|---|---|---|
| `APN` | `char apn[64]` | `airtelgprs.com` | The special value `AUTO` means detect the APN from the operator (see Phase 2). |
| `BRK` | `char host[64]` + `uint16_t port` | `52.66.89.140`, `1883` | Accepts an IP or a domain, with an optional `:port`. |
| `TOP` | `char topic[64]` | `logger/{ID}/log` | `{ID}` is replaced by `%08lX` of `g_device_id` when used. |
| `ADM` | `char admins[3][16]` | compiled-in list | The whitelist of phone numbers allowed to send config. |

### Storage format and location

Store it as a `net_cfg_record_t`:
`{ magic, layout_ver, seq, net_cfg_t active, net_cfg_t lkg, char pending_apn[64], char last_iccid[21], admins, crc32 }`.
`lkg` is the "last known good" config, the one we fall back to.

- **Location:** two flash sectors used as A/B copies: sector 6 (`0x08040000`) and sector 7 (`0x08060000`), 128 KB each.
- **Reserving them:** shrink `FLASH LENGTH` in [STM32F446XX_FLASH.ld](STM32F446XX_FLASH.ld) (the uppercase file is the one CMake uses) to 256K. The app is currently about 80 KB.
- **Writing:** always write to the inactive copy, newest valid `seq` + CRC wins. A power cut mid-write can never leave the device without a valid config.
- **If neither copy is valid:** use the compiled-in defaults, which are the current hardcoded values.
- **CRC:** reuse `crc32_update` from [app_main.c:51](App/dg/Src/app_main.c#L51). Move it to a common helper.
- **Erase timing:** erasing a 128 KB sector stalls the CPU for about 1–2 s on this single-bank part. Only erase while the MQTT link is already torn down (Phase 2 arranges this), never during a publish.

New files: `App/common/Inc/net_cfg.h` and `App/common/Src/net_cfg.c` (types, defaults, load/save). Add them to [CMakeLists.txt](CMakeLists.txt#L67).

---

## Phase 2 — Applying new values safely and falling back

### Driver changes ([ec200_driver.c](App/common/Src/ec200_driver.c), [ec200_driver.h](App/common/Inc/ec200_driver.h))

- Add `apn[64]`, `broker_host[64]` and `broker_port` to `ec200_driver_t`. `EC200_DataAPN` and `EC200_MQTT_Sock` build their AT commands from these with `snprintf` instead of string literals.
- Add `ec200_set_net_params(dev, apn, host, port)` and `ec200_request_reconnect(dev)`.
  - The reconnect request is acted on only in `EC200_Idle_Online`, so it never cuts into a publish.
  - It routes into the existing but currently unreachable `EC200_Net_Cleanup` state. Add `AT+QMTDISC=0` before `QMTCLOSE` there.
- **Read the SIM's identity:**
  - Add `AT+QCCID` (the SIM's serial number, ICCID) and `AT+COPS?` (operator name, after `AT+COPS=3,0`).
  - Parse them in the RX callback into `iccid[21]` and `operator_name[24]`. The best place to issue them is the `EC200_SIM_State` → `NW_Reg` path.
- **Expose connection health** so `net_cfg` can judge whether a trial worked:
  - `ec200_is_ready()` already exists.
  - Add a counter that increments each time the FSM falls back to `NW_Reg`/`DataAPN` after a PDP or MQTT failure.
- **Topic:** move it into the 64-byte config. Grow `mqtt_topic` in [app_main.c:88](App/dg/Src/app_main.c#L88) from 32 to 64 bytes, and rebuild it whenever the config changes.

### Trial-and-revert (in `net_cfg.c`, ticked from `app_run`)

```
STABLE --(valid CFG SMS)--> save {active=new, lkg=old} --> TRIAL
TRIAL : success = MQTT READY + first PUBACK (EC200_PUB_OK) within 3 min
        -> lkg = active, save, reply "OK"           -> STABLE
        failure = timeout or 3 reconnect cycles
        -> active = lkg, save, reconnect, reply "ERR ... reverted" -> STABLE
```

The record holds both `active` and `lkg`. So if power is lost during a trial, the next boot sees an unconfirmed trial and starts from `lkg`.

### The APN / SIM-swap problem (the "user does it in the wrong order" cases)

- **One rule covers both orders.** When `APN=x` arrives, try it now under the trial rules.
  - If the trial fails, revert, but **keep `x` as `pending_apn`** and reply: `APN x failed on current SIM; will try it on next SIM`.
  - Sent before the swap: the new APN fails on the old SIM, gets staged, and is used after the swap.
  - Sent after the swap: it works right away.
- **Detecting a swap at boot.** If `iccid != last_iccid`, try these APNs in order, two PDP attempts each. The first that reaches MQTT READY becomes `active` (and `lkg`), and the admins get an SMS.
  1. `pending_apn`
  2. the operator table: substring match on the `COPS` name. airtel → `airtelgprs.com`, jio → `jionet`, vi/vodafone/idea → `www`, bsnl → `bsnlnet`
  3. the current `active`
  4. empty APN (many networks hand out a default one)
- **If every candidate fails:** the device stays reachable by SMS, because SMS needs no APN. Send the admins one rate-limited (once every 6 hours) SMS:
  `NO DATA. ICCID ...., OP <name>. Send: CFG APN=<apn>`.
  This is the safety net for every other failure case.

---

## Phase 3 — SMS format

Design rules:
- One message of at most 160 plain-text (GSM-7) characters.
- Only characters you can type easily, with **no quotes or commas**. These would get turned into curly quotes and force Unicode encoding, and they are dangerous inside AT command strings.
- The command word and keys are case-insensitive, so the keyboard's automatic capital ("Cfg") does not matter.
- Tokens can be separated by spaces, `;` or newlines.

```
CFG APN=jionet
CFG BRK=mqtt.example.com:8883 TOP=fleet/{ID}/log
CFG APN=airtelgprs.com BRK=13.233.10.5
CFG GET                  -> replies with the current config + ICCID + state
CFG STATUS               -> state, signal strength (CSQ), operator, last PUBACK age, failure count
CFG RESET                -> back to the compiled-in defaults, goes through a trial like any change
```

- Every SMS gets an SMS reply to the sender: `OK <what changed>` or `ERR <KEY>: <reason>`.
- A message is **all or nothing**: any invalid field rejects the whole message, so nothing is applied partially.
- Messages that don't start with `CFG` are ignored silently.
- Messages from numbers not on the whitelist are dropped silently, with no reply, so the device doesn't reveal itself.

---

## Phase 4 — Receiving SMS and the parser

### Driver-side reception ([ec200_driver.c](App/common/Src/ec200_driver.c))

- Add `AT+CSDH=1` and `AT+CSCS="GSM"` to `EC200_SMS_Setup`. With `CSDH=1` the `+CMT` header carries `<fo>`, `<dcs>` and `<length>`, which tell us the encoding and whether the message is one part of a longer one.
- Grow `rx_line` from 256 to 384 bytes. A 70-character Unicode SMS arrives as 280 hex characters and would be cut off today.
- **Replace the single slot with a small queue.** Today `rx_sms_sender`/`rx_sms_payload` are one slot that the next SMS overwrites.
  - The ISR (interrupt handler) only copies the raw header line and raw body line into a 2-entry queue `ec200_sms_raw_t {char hdr[96]; char body[320];}`.
  - Add `bool ec200_take_sms(dev, ec200_sms_raw_t *out)`, following the `ec200_take_pub_result` pattern. Remove the print in `Idle_Online`.
- **Fix:** `prompt_received` is set by any `>` byte, including one inside an SMS body. Only set it when `>` is the first byte on a fresh line (`rx_idx == 0`).

### Parser module (new: `App/common/Inc/sms_cfg.h`, `App/common/Src/sms_cfg.c`)

It is pure C with no HAL, so it can be unit-tested on your PC. The stages:

1. **`sms_parse_header(hdr, &sender, &dcs, &fo)`** walks the comma-separated fields while respecting quotes. The sender is field 1, `<fo>` is field 5 and `<dcs>` is field 7.
   - If `fo & 0x40` is set, the message is one part of a longer SMS. Reject it with the reply `ERR too long`.
2. **`sms_decode_body(body, dcs, out, outlen)`** turns the body into plain ASCII.
   - If `(dcs & 0x0C) == 0x08` the body is Unicode: hex-decode the UTF-16BE text.
     - Map curly quotes ‘ ’ “ ” to ' and ", the non-breaking space to a space, and dashes – — to -.
     - Reject any character above 0x7E that remains.
   - Otherwise the body is GSM-7, which text mode already delivers as ASCII.
3. **`sms_sender_allowed(sender, admins)`** normalizes both sides to their last 10 digits, so that +91, 0 and bare numbers all match.
4. **`sms_cfg_parse(text, &cmd, &cfg_delta, err, errlen)`** splits the text into tokens and checks:
   - the command is `CFG`
   - every key is known and appears at most once
   - each value passes its validator (below)
   - it returns a structure saying which fields are set.

Validators. Each returns an error string that goes straight into the reply SMS.

- **APN:** length 1–63, or the word `AUTO`. Allowed characters: `[A-Za-z0-9.-]`. No empty labels; it must not start or end with `.` or `-`.
- **BRK host:**
  - **IPv4:** exactly 4 numbers, each 0–255, with no leading zeros. Reject `0.0.0.0`, `255.255.255.255` and `127.x`.
  - **Hostname:** total length ≤ 63, 1–63 characters per label, characters `[A-Za-z0-9-]`, no `-` at either end of a label. It must contain at least one dot, and the last label cannot be all digits.
- **BRK port:** optional `:` followed by only digits, 1–65535. The default is 1883.
- **TOP:**
  - Length ≤ 63 after `{ID}` is expanded. Allowed characters: `[A-Za-z0-9/_.-]` plus the `{ID}` token.
  - Rejected: `+`, `#`, a leading `$`, a leading or trailing `/`, and `//`.
  - **It must contain `{ID}`,** otherwise every logger would publish to the same topic.
- **Injection safety:** these character sets leave out `"`, `,`, `\r`, `\n` and `;`, so a value can never break out of the AT command strings built from it.

### Wiring (in [app_main.c](App/dg/Src/app_main.c))

`app_run` → `ec200_take_sms()` → `sms_cfg_*` → `net_cfg_apply()` → reply with `ec200_send_sms()`.

The reply has to wait for `Idle_Online`, because `ec200_send_sms` currently replaces `fsm_state` without checking it. Add a guard there, or a one-slot outgoing queue.

---

## Critical files

- [App/common/Src/ec200_driver.c](App/common/Src/ec200_driver.c) and [App/common/Inc/ec200_driver.h](App/common/Inc/ec200_driver.h): configurable APN/broker, reconnect request, ICCID/COPS, CSDH/CSCS setup, SMS queue, `>` fix, `send_sms` guard.
- New: `App/common/{Inc,Src}/net_cfg.{h,c}` (config store, trial/revert, SIM-swap APN search) and `App/common/{Inc,Src}/sms_cfg.{h,c}` (header parse, Unicode decode, whitelist, parser, validators).
- [App/dg/Src/app_main.c](App/dg/Src/app_main.c): topic from config (64 bytes), SMS wiring, shared CRC helper, optional SWO `_write`.
- [STM32F446XX_FLASH.ld](STM32F446XX_FLASH.ld): reserve sectors 6–7.
- [CMakeLists.txt](CMakeLists.txt): add the new sources.
- New: `tests/test_sms_cfg.c`, a host-side test built with plain `gcc`, not part of the firmware build.

## Verification

1. **Phase 0:** `+CMT` appears on the FTDI or SWO for (a) a plain SMS, (b) an SMS containing an emoji, so it arrives as Unicode, (c) an SMS sent as the first message of a new conversation, not a reply.
2. **Parser on your PC:** `gcc -I App/common/Inc App/common/Src/sms_cfg.c tests/test_sms_cfg.c && ./a.out`. Cover:
   - valid and invalid examples for each validator: IP octet 256, port 0, port 70000, `+` or `#` in the topic, no `{ID}`, a 64-character APN, `"` injection
   - Unicode with curly quotes
   - lowercase `cfg`
   - duplicate keys
   - a message from a number that isn't whitelisted
3. **On the device:**
   - Change `TOP=` and confirm on the EC2 broker that `mosquitto_sub -t 'logger/#' -v` sees the new topic.
   - Send `BRK=` with a dead IP; the device should revert within about 3 minutes and reply with `ERR`.
   - Pull power during a trial; the device should boot on `lkg`.
4. **SIM swap, both orders:** send `APN` and then swap SIMs, and swap SIMs and then send `APN`. Also swap with no SMS at all; the operator table or the empty APN should connect, and the admins should get the `NO DATA` SMS if nothing does.
5. **Robustness:** after a full SMS-driven reconnect cycle, the RAM queue (`rq_dropped`) and the SD logging keep running. During the flash erase stall, CAN and Modbus lose at most about 2 s of samples.
