[[Rhygen Firmware]]

Welcome to hell. This is about to be fun.
## Goal
We have an I2C interface with the BNO055, and I need to figure out which commands to send, and in what format to do that for me to get the data from all 3 of it's sensors.

I better do it in the dummy repo or a separate branch of the logger 0,2 code, so that the board config does not beat my ass when I get to it.

Also, thank GOD for bosch and it's datasheets.

First thoughts are that I'll run it in IMU (fusion) mode mode, and get the accel and gyro data. But for just that, I don't think I will be using the sensor to it's full capacity. Also, this is primarily going to be used in vehicles, so I need to find out how is an IMU actually used there.

## It works. Here's how

We have an I2C pal that runs it's own FSM, for keeping track of each transaction that happens on the bus. We are saved a little by our constraint of having a single master and a single device to talk to. We kinda don't have to care about the bus arbitration or anything else that concerns the multi-slave or even multi-master scenarios with the PAL.

The three layers, and who calls whom:

```
APPLICATION (app_main.c)          main loop, every pass
   │  i2c->process()  ──────────────────────────────┐
   │  bno055_process()                              │
   │  bno055_get_sample()                           │
   ▼                                                ▼
BNO055 DRIVER (bno055.c)  ──submit()──►  I2C PAL (pal_i2c_stm32.c)
   ▲                                                │ HAL_I2C_Master_Seq_*_IT()
   └──────────── done() callback ◄──────────────────┤
                 (from inside i2c->process())       ▼
                                             STM32 HAL + I2C ISRs
                                             (TxCplt / RxCplt / Error callbacks
                                              only move the PAL's state forward)
```

Two contexts exist:
- **Main context**: `submit()`, `process()`, the `done()` callback, everything in the BNO driver and the app.
- **ISR context**: the three HAL callbacks. They only write the PAL's `state` / `result` and never call user code.

### Generic I2C device interfacing with the PAL
![[Random_I2C_Comms_Trace.excalidraw]]
Let's trace a random device that I wish to talk to via the I2C pal.  Say that the device is at ID `0x30`, and I wish to request 4 bytes of data from that device. I want the complete flow of this process, how shall I construct the transaction descriptor, which functions shall be called, when will the HAL callbacks fire, all end to end.

Assume that there are no timeouts or bus errors that happen along the way.

#### 1. Build a transfer descriptor
```C
static uint8_t tx[1] = { 0x10 };   // register to read from
static uint8_t rx[4];

const pal_i2c_xfer_t x = {
    .addr       = 0x30,            // 7-bit address; the PAL shifts it left by 1 for the HAL
    .tx         = tx,
    .tx_len     = 1,
    .rx         = rx,
    .rx_len     = 4,
    .timeout_ms = 100,             // 0 = use PAL_I2C_DEFAULT_TIMEOUT_MS
    .done       = my_done,         // called once with the final status
    .ctx        = my_device,       // opaque; handed back to done() untouched
};
```

- `ctx` is a `void*` the PAL never looks at. It just gives it back to `done()` so the callback knows which device the transfer was for. The BNO driver passes a pointer to its own `bno055_t` instance (its run-time state: FSM state, buffers, stats). The register macros and config constants are *not* in there; they are `#define`s in `bno055.c`.
- The descriptor itself is **copied** into the PAL's queue, so it can live on the stack. The `tx` / `rx` buffers it points to are **not** copied. They must stay valid until `done()` fires. That's why the BNO driver uses `d->tx` and `d->rx` inside its instance struct instead of locals.

#### 2. `submit()`
`submit()` validates the descriptor (address ≤ 0x7F, at least one of tx/rx non-empty, non-NULL buffers for non-zero lengths). If that fails it returns `PAL_I2C_ERR_PARAM`. Then it pushes a copy into the ringbuffer (`PAL_I2C_ERR_FULL` if full) and calls `start_next()`.

Return value: `PAL_I2C_OK` only means "accepted and queued". Anything that goes wrong after that is reported through `done()`. Every accepted transfer gets **exactly one** `done()` call. A rejected one gets **none**, so the caller must handle a non-OK return itself.

#### 3. `start_next()`
Called from both `submit()` and the end of `process()`.
- If the PAL is not `XFER_IDLE` (another transfer is in flight), it does nothing. The transfer waits in the queue and a later `process()` starts it.
- Otherwise it pops the queue into `active`, fills in the default timeout if `timeout_ms == 0`, records `started_ms = HAL_GetTick()`, moves to `XFER_WAIT_BUS`, and **immediately calls `try_start()`**.

So in the normal case the hardware transfer starts **inside the `submit()` call**. It doesn't wait for the next `process()`.

#### 4. `try_start()`
Checks the peripheral's hardware `BUSY` flag. This is a line-level flag: SDA/SCL are being held or a START was seen without a STOP. It is not "our own transfer is running", because that is already guarded by `state != IDLE`.
- `BUSY` clear → `start_hw()`.
- `BUSY` set → stay in `XFER_WAIT_BUS`. Each `process()` call first checks the timeout and then calls `try_start()` again. With a single master this mostly happens when the bus is stuck, and it ends in `ERR_TIMEOUT` plus bus recovery.

#### 5. `start_hw()`
Sets `result = OK`, then sets the state **before** calling the HAL, because the transfer can finish in an ISR before the HAL call even returns.
- `tx_len > 0` → state `XFER_WRITE`, `HAL_I2C_Master_Seq_Transmit_IT(...)` with:
  - `rx_len == 0` → `I2C_FIRST_AND_LAST_FRAME`. Pure write: START, address+W, bytes, **STOP**. Used for writing a config register, e.g. every BNO init step.
  - `rx_len > 0` → `I2C_FIRST_FRAME`. Write **without a STOP**, so the bus stays ours and the read phase can follow with a **repeated START**. Used for "write a register address, then read from it", e.g. reading the BNO's CHIP_ID or the sample burst.
- `tx_len == 0` → state `XFER_READ`, `HAL_I2C_Master_Seq_Receive_IT(..., I2C_FIRST_AND_LAST_FRAME)`. Pure read: START, address+R, bytes, STOP. The BNO driver never uses this.
- If the HAL call itself returns non-OK, no interrupts were enabled. `start_hw()` stores `hal_error`, sets `result = ERR_BUS` and `state = XFER_COMPLETE`, and the next `process()` reports it.

For our 4-byte example this is `FIRST_FRAME` with 1 tx byte.

#### 6. HAL callbacks (ISR context)
- `HAL_I2C_MasterTxCpltCallback`: ignored unless state is `XFER_WRITE`.
  - `rx_len == 0` → `XFER_COMPLETE`. Done.
  - `rx_len > 0` → state `XFER_READ`, then `HAL_I2C_Master_Seq_Receive_IT(..., I2C_LAST_FRAME)`. `LAST_FRAME` in the other direction makes the HAL issue the repeated START, read the bytes, NACK the last one and send STOP. **Only if that call fails** is `ERR_BUS` recorded and the state moved to `XFER_COMPLETE`. On success the state stays `XFER_READ` until the next callback.
- `HAL_I2C_MasterRxCpltCallback`: ignored unless state is `XFER_READ`. → `XFER_COMPLETE`.
- `HAL_I2C_ErrorCallback` (not in the happy path, but the BNO hits it all the time while booting): only acts in `XFER_WRITE` / `XFER_READ`. A pure acknowledge failure (`HAL_I2C_ERROR_AF`) → `ERR_NACK`; the HAL already sent STOP, so the bus is fine. Anything else (BERR, ARLO, OVR) → `ERR_BUS`. Either way → `XFER_COMPLETE`.

Bus picture for our example:
```
S | 0x30+W | A | 0x10 | A | Sr | 0x30+R | A | D0 A | D1 A | D2 A | D3 N | P
└──── Seq_Transmit_IT FIRST_FRAME ───┘ └──── Seq_Receive_IT LAST_FRAME ─────┘
                        TxCplt fires here ┘                RxCplt fires here ┘
```

#### 7. `process()` finishes it (main context)
On the next `process()` call:
1. If the state is still `WAIT_BUS` / `WRITE` / `READ` and the timeout has run out → `abort_active()` (disable the I2C interrupts first so no ISR can race, then `ERR_TIMEOUT`). Otherwise, in `WAIT_BUS`, retry `try_start()`.
2. If `XFER_COMPLETE`: copy `active` and `result` into locals first, because `done()` may call `submit()` and reuse `active`. Run `bus_recover()` only for `ERR_TIMEOUT` / `ERR_BUS` (**not** for NACK). Set `XFER_IDLE`, then call `done(self, status, ctx)` if it isn't NULL.
3. `start_next()` → launch the next queued transfer, if any.

So `done()` always runs **inside `i2c->process()`**, never in an ISR. A finished transfer is seen on the first `process()` call after the ISR marked it complete.

---
### How does the BNO driver actually use this PAL

#### App side
All the app does (`App/dg/Src/app_main.c`):
```C
static bno055_t imu;

// once, in init: no I/O happens here
bno055_init(&imu, hardware.imu_i2c, BNO055_ADDR_DEFAULT, IMU_SAMPLE_PERIOD_MS /* 20 ms = 50 Hz */);

// every main-loop pass
hardware.imu_i2c->process(hardware.imu_i2c);   // 1. drive the bus, fire done()
bno055_process(&imu, current_time);            // 2. advance the BNO FSM
if (bno055_get_sample(&imu, &sample)) { ... }  // 3. true once per new sample
```
The order matters: running `i2c->process()` first means a transfer that finished since the last pass gets its `done()` call, and then `bno055_process()` acts on it in the **same** pass instead of one pass later.

`bno055_get_state()`, `bno055_get_stats()` and `bno055_state_str()` are for monitoring only. The app prints state transitions and periodic stats.

Nothing blocks. The driver uses no HAL calls and no `HAL_GetTick()`; time comes in as `now_ms`.

#### The "op": one register access
The driver never has more than **one** transfer in flight (`op_in_flight`). Its unit of work is an *op* = one register read or write:

| Op                   | `tx`           | `tx_len` | `rx_len` | PAL framing                  | On the bus                          |
| -------------------- | -------------- | -------- | -------- | ---------------------------- | ----------------------------------- |
| Register write       | `{reg, value}` | 2        | 0        | `FIRST_AND_LAST_FRAME`       | `S 0x28+W reg val P`                |
| Register read        | `{reg}`        | 1        | N        | `FIRST_FRAME` → `LAST_FRAME` | `S 0x28+W reg Sr 0x28+R d0..dN-1 P` |
| Page select (prefix) | `{0x07, 0x00}` | 2        | 0        | `FIRST_AND_LAST_FRAME`       | `S 0x28+W 0x07 0x00 P`              |

The BNO055 register map has two pages, and everything we use is on page 0. The driver caches the current page in `d->page` (`-1` = unknown). 

While the page is unknown (at start-up and after **any** failed transfer), the op first writes `PAGE_ID = 0`. Only after that write's `done()` reports OK does it submit the real access.

The two transfers are issued **one after the other, never queued together**. If they were both queued, a failed page select would still be followed by the real access, which could then hit the wrong page.

Call chain to start an op:
```
start_action()            picks the op for the current FSM state
 └ op_read() / op_write() store reg / len / value in d->op_*
    └ op_kick()            page unknown? → PAGE_ID write (op_is_page = true)
       │                   else          → op_issue_main()
       └ submit_xfer()     builds the descriptor (ctx = d, done = xfer_done),
                           sets op_in_flight = true, op_done = false,
                           calls i2c->submit()
```
If `submit()` refuses (`ERR_FULL` / `ERR_PARAM`), `done()` will never come. `submit_xfer()` therefore completes the op as failed right away (`op_done = true`, `op_status = st`, `stats.xfer_rejected++`), and the normal failure path takes over.

`xfer_done()` (the PAL's `done` callback) only does two things: it stores `op_status` and sets `op_done = true`.

#### `bno055_process()`
```
if an op is in flight:
    not done yet → return
    done:
        op_in_flight = false, count the status in stats
        error            → page = -1 (re-select next time), on_op_done(error)
        OK, was PAGE_ID  → page = 0, op_issue_main(), return   // the real access, same op
        OK, real access  → on_op_done(OK)

if no op in flight and now >= next_ms → start_action()
```
`next_ms` is how all delays work: probe period, retry delay, init-step settle times, the sample period. The FSM never sleeps; it sets `next_ms` and the next `bno055_process()` past that time starts the next op.

#### Driver FSM
```
PROBING ──CHIP_ID == 0xA0──► CONFIGURING ──all 5 init writes OK──► RUNNING
   ▲                                │                                 │
   └── > MAX_RETRIES (3) consecutive failures, or SYS_STATUS != 5 ────┘
       (fusion running) for >= 1500 ms
```

**PROBING**: read 1 byte from `CHIP_ID` (0x00) every `PROBE_PERIOD_MS` (20 ms).
- The BNO needs ~650 ms to boot and NACKs its address until then. That shows up as `ERR_NACK` in the stats and is expected.
- It retries forever; failures here don't count toward `retries`.
- On `0xA0` → CONFIGURING with `step = 0`.

**CONFIGURING**: one register write per step, waiting `wait_ms` after each before the next:

| Step | Register | Value | Wait | Why |
|---|---|---|---|---|
| 0 | `OPR_MODE` 0x3D | 0x00 CONFIG | 25 ms | Most registers are only writable in CONFIG mode; any→CONFIG takes 19 ms |
| 1 | `PWR_MODE` 0x3E | 0x00 NORMAL | 10 ms | |
| 2 | `SYS_TRIGGER` 0x3F | 0x00 (0x80 = external 32 kHz crystal) | 10 ms | Clock source |
| 3 | `UNIT_SEL` 0x3B | 0x81 | 1 ms | Degrees, mg, °C, Android orientation |
| 4 | `OPR_MODE` 0x3D | 0x0C NDOF | 20 ms | 9-DOF fusion (accel + gyro + mag); CONFIG→any takes 7 ms |

A failed step is retried after `RETRY_DELAY_MS` (5 ms) on the same step. After more than `MAX_RETRIES` (3) consecutive failures the driver falls back to PROBING (`enter_probing()` resets step, retries, page, and bumps `stats.reinit`).

**RUNNING**: one 50-byte burst read from 0x08 to 0x39 each `period_ms`:

| Offset in `rx` | Register | Data | Scale |
|---|---|---|---|
| 0 | 0x08 | ACC x/y/z (gravity included) | 1 LSB = 1 mg |
| 6 | 0x0E | MAG x/y/z | read, not published |
| 12 | 0x14 | GYR x/y/z | read, not published |
| 18 | 0x1A | Euler heading/roll/pitch | 16 LSB = 1° |
| 24 | 0x20 | Quaternion w/x/y/z | 16384 LSB = 1.0 |
| 32 | 0x28 | Linear accel x/y/z (gravity removed) | 1 LSB = 1 mg |
| 38 | 0x2E | Gravity vector x/y/z | 1 LSB = 1 mg |
| 44 | 0x34 | Temperature | 1 °C |
| 45 | 0x35 | CALIB_STAT: sys/gyr/acc/mag, 2 bits each | 0..3 |
| 49 | 0x39 | SYS_STATUS | 5 = fusion running |

All 16-bit values are little-endian signed (`le16()`).

On every successful read the driver checks `SYS_STATUS`:
- `== 5` → `publish_sample()` decodes into `d->sample`, stamps `timestamp_ms = now`, sets `sample_new`.
- `!= 5` → the sample is dropped (`stats.not_fusing++`). That's normal for a moment after entering NDOF. If it lasts `NOT_FUSING_LIMIT_MS` (1500 ms), the chip has most likely reset itself back into CONFIG mode, so the driver goes back to PROBING and reconfigures it.

Sampling is fixed-rate: `next_ms += period_ms`. If the driver fell behind (the new `next_ms` is already in the past), it sets `next_ms = now` instead of firing a burst of reads to catch up.

#### End to end: one sample in RUNNING (page already 0)
```
Pass N   (now >= next_ms)
  app: i2c->process()          nothing to do
  app: bno055_process()
         start_action → op_read(0x08, 50) → op_kick → op_issue_main
         → submit_xfer → i2c->submit → push → start_next → try_start → start_hw
         → HAL_I2C_Master_Seq_Transmit_IT(0x50, {0x08}, 1, FIRST_FRAME)   state WRITE

ISR      TxCplt → HAL_I2C_Master_Seq_Receive_IT(0x50, d->rx, 50, LAST_FRAME)  state READ
ISR      RxCplt                                                             state COMPLETE
         (~5 ms at 100 kHz, ~1.3 ms at 400 kHz)

Pass N+k (first pass after RxCplt)
  app: i2c->process()          COMPLETE → IDLE → xfer_done(): op_status = OK, op_done = true
  app: bno055_process()        op done → on_op_done(RUNNING) → SYS_STATUS == 5
                               → publish_sample → schedule next_ms
  app: bno055_get_sample()     returns true with the new sample
```
After any failure the same sample takes one extra round trip: `page = -1`, so the next op first writes PAGE_ID, and the 50-byte read is submitted from inside `bno055_process()` once that write's `done()` reports OK.

#### Stats (`bno055_get_stats()`)
| Counter                                                   | Incremented when                                                    |
| --------------------------------------------------------- | ------------------------------------------------------------------- |
| `xfer_ok` / `xfer_nack` / `xfer_timeout` / `xfer_bus_err` | Each completed transfer, by `done()` status (page selects included) |
| `xfer_rejected`                                           | `submit()` refused the descriptor                                   |
| `reinit`                                                  | Driver fell back to PROBING                                         |
| `samples`                                                 | A valid sample was published                                        |
| `not_fusing`                                              | A sample was dropped because SYS_STATUS != 5                        |

A healthy run: `nack` climbs during the first ~650 ms of boot, then only `ok` and `samples` grow, at roughly `1000 / period_ms` per second.
