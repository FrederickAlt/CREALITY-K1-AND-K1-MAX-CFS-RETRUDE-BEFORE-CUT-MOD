# Box serial communication interface

## Scope

This document covers only the observed application-level serial frame format,
box command ids, payload bit fields, response states, and low-level G-code
macros that directly map to serial commands. It intentionally excludes
material-change workflows, motion/cleaning flows, auto-refill, and recovery
logic; those belong in separate documents.

This document describes the observed application-level serial interface used to
communicate with the material box over the configured serial transport. For
detailed command layouts, response shapes, state/error bytes, and timeout values,
see [Protocol reference](protocol-reference.md) and [Command timeouts](command-timeouts.md).

The wrapper sends byte packets through the configured `serial_485` transport and
waits for a response using a per-command timeout.

The wrapper constructs the packet body shown below. Any lower-level
UART framing, checksum, start byte, or retries are handled outside this module by
`serial_485`.

## Request frame

```text
+------+--------+-------+-----+-------------------+
| ADDR | LENGTH | STATE | CMD | DATA ...          |
+------+--------+-------+-----+-------------------+
| 1 B  | 1 B    | 1 B   | 1 B | 0 or more bytes   |
+------+--------+-------+-----+-------------------+
```

| Field | Size | Meaning |
|---|---:|---|
| `ADDR` | 1 byte | Physical box address, normally `0x01`..`0x04`. |
| `LENGTH` | 1 byte | `len(CMD) + 2 + len(DATA)`. Since `CMD` is one byte, this is normally `3 + len(DATA)`. |
| `STATE` | 1 byte | Host request state, normally `0xff`. |
| `CMD` | 1 byte | Box command id. |
| `DATA` | variable | Command-specific payload. |

## Response frame

Responses are parsed with these byte positions:

```text
+------+-------+--------+-------+-----+-------------------+
| HEAD | ADDR  | LENGTH | STATE | CMD | DATA ...          |
+------+-------+--------+-------+-----+-------------------+
| 0    | 1     | 2      | 3     | 4   | 5 ...             |
+------+-------+--------+-------+-----+-------------------+
```

| Position | Field | Meaning |
|---:|---|---|
| `0` | `HEAD` | Present in replies, but not interpreted by this layer. |
| `1` | `ADDR` | Box address. |
| `2` | `LENGTH` | Reply length byte. |
| `3` | `STATE` | Response state/result code. |
| `4` | `CMD` | Command id echoed by the box. |
| `5...` | `DATA` | Command-specific response payload. |

## Command ids

| Command | Byte | Request data |
|---|---:|---|
| `CREATE_CONNECT` | `0x00` | none |
| `GET_RFID` | `0x01` | slot mask, default `0x0f` |
| `GET_REMAIN_LEN` | `0x02` | slot mask, default `0x0f` |
| `SET_BOX_MODE` | `0x03` | slot byte + mode byte |
| `GET_BUFFER_STATE` | `0x04` | none |
| `CTRL_MATERIAL_MOTOR_ACTION` | `0x05` | reserved / not documented for normal operation |
| `CTRL_CONNECTION_MOTOR_ACTION` | `0x06` | action byte |
| `GET_FILAMENT_SENSOR_STATE` | `0x07` | sensor-bank byte |
| `SET_MOTOR_SPEED` | `0x08` | reserved / not documented for normal operation |
| `GET_BOX_STATE` | `0x09` | none |
| `SET_PRE_LOADING` | `0x0a` | slot mask + action byte |
| `MEASURING_WHEEL` | `0x0b` | action byte |
| `TIGHTEN_UP_ENABLE` | `0x0c` | enable byte |
| `EXTRUDE_PROCESS` | `0x0d` | slot byte + stage byte + extrude byte |
| `RETRUDE_PROCESS` | `0x0e` | slot byte + trigger byte |
| `EXTRUDE_PROCESS_MODEL2` | `0x0f` | slot byte + trigger byte |
| `GET_VERSION_SN` | `0x10` | none |
| `GET_HARDWARE_STATUS` | `0x11` | reserved / not documented for normal operation |
| `COMMUNICATION_TEST` | `0x55` | one test byte |

## Low-level Klipper/G-code macros

These are the low-level macros that most directly translate to one serial
command. They still parse G-code parameters and use the shared `send_data`
response/error handling, but they do not perform material-change, motion,
retry, heating, cutter, or cleaning workflows.

| G-code macro | Serial command | Notes |
|---|---|---|
| `BOX_CREATE_CONNECT ADDR=<1..4>` | `CREATE_CONNECT` | No data bytes. Ignored while printing. |
| `BOX_GET_BOX_STATE ADDR=<1..4>` | `GET_BOX_STATE` | No data bytes. Timeout handling is special: a timeout returns `None`. |
| `BOX_GET_VERSION_SN ADDR=<1..4>` | `GET_VERSION_SN` | No data bytes. |
| `BOX_GET_RFID ADDR=<1..4> NUM=<0..15>` | `GET_RFID` | `NUM` is masked with `0x0f` and sent as one slot-mask byte. |
| `BOX_GET_REMAIN_LEN ADDR=<1..4> NUM=<0..15>` | `GET_REMAIN_LEN` | `NUM` is masked with `0x0f` and sent as one slot-mask byte. |
| `BOX_GET_BUFFER_STATE ADDR=<1..4>` | `GET_BUFFER_STATE` | No data bytes. |
| `BOX_GET_FILAMENT_SENSOR_STATE ADDR=<1..4> POSITION=MATERIAL\|CONNECTIONS` | `GET_FILAMENT_SENSOR_STATE` | Sends one sensor-bank byte, then prints the decoded mask. |
| `BOX_SET_BOX_MODE ADDR=<1..4> MODE=PRINT\|IDLE [NUM=A\|B\|C\|D\|0]` | `SET_BOX_MODE` | Sends slot byte + mode byte. Missing `NUM` defaults to `0`. |
| `BOX_CTRL_CONNECTION_MOTOR_ACTION ADDR=<1..4> ACTION=STOP\|EXTRUDE\|RETRUDE` | `CTRL_CONNECTION_MOTOR_ACTION` | Sends one action byte. |
| `BOX_MEASURING_WHEEL NUM=<1..4> ACTION=GET\|CLEAN` | `MEASURING_WHEEL` | `NUM` is the box address. Sends one action byte. |
| `BOX_TIGHTEN_UP_ENABLE ADDR=<1..4> ENABLE=ENABLE\|DISABLE` | `TIGHTEN_UP_ENABLE` | Sends one enable byte. |
| `BOX_EXTRUDE_PROCESS ADDR=<1..9> NUM=A\|B\|C\|D [VELOCITY=<1..30>]` | `EXTRUDE_PROCESS` | For supported versions documented here, this macro sends stage `0x02`; the parsed `VELOCITY` is not placed in the packet. |
| `BOX_EXTRUDE_2_PROCESS ADDR=<1..4> NUM=A\|B\|C\|D TRIGGER=CONNECTION\|BUFFER` | `EXTRUDE_PROCESS_MODEL2` | Sends slot byte + trigger byte. |
| `BOX_RETRUDE_PROCESS ADDR=<1..4> NUM=A\|B\|C\|D\|0 TRIGGER=BUFFER\|MATERIAL` | `RETRUDE_PROCESS` | Sends slot byte + trigger byte. |
| `BOX_SEND_DATA ADDR=<1..4> NUM=<0..32> STATE=<0..255> TIMEOUT=<1..120> [DATA=<digits>]` | raw `send_data` wrapper | Builds a normal request frame with custom command byte `NUM`. It is not a full raw-frame sender; see below. |
| `BOX_COMMUNICATION_TEST ADDR=<1..4> NUM=<0..255> [COUNT] [TIMEOUT] [INTERVAL]` | `COMMUNICATION_TEST` | Diagnostic loop. Sends the same test frame repeatedly and expects returned data `(NUM + 1) & 0xff`. |

### Macro safety guards

A few low-level macros still have guards or wrapper-side state effects:

- `BOX_CREATE_CONNECT` is ignored while printing.
- `BOX_SET_PRE_LOADING` may be skipped while printing or paused unless an
  wrapper-managed forced path is used.
- `BOX_SET_PRE_LOADING ACTION=OPEN|CLOSE` changes the wrapper's preloading flag.
- `BOX_SET_PRE_LOADING POWER_ON=ENABLE|DISABLE` changes the wrapper's power-on
  preloading flag.

### Raw-ish sender: `BOX_SEND_DATA`

`BOX_SEND_DATA` is the closest public macro to a raw serial sender. It still
constructs the frame using the normal request format:

```text
ADDR | LENGTH | STATE | NUM-as-CMD | DATA...
```

Important limitations:

| Limitation | Detail |
|---|---|
| Command range | `NUM` is limited by the macro to `0..32`, so it cannot send `COMMUNICATION_TEST` (`0x55`). |
| Length byte | Computed automatically; cannot be supplied manually. |
| Head/checksum/trailer | Not supplied by this macro; any such framing belongs to `serial_485`. |
| Data parsing | `DATA` is parsed character-by-character as decimal digits. `DATA=0123` becomes bytes `00 01 02 03`. |
| Bytes >= `0x0a` in `DATA` | Not directly representable through the public digit parser. For example `DATA=15` becomes `01 05`, not `0x0f`. |
| Response handling | Still uses `send_data`, so non-OK states can trigger normal error handling. |

Example raw-ish macro that sends `GET_FILAMENT_SENSOR_STATE/MATERIAL`:

```gcode
BOX_SEND_DATA ADDR=1 NUM=7 STATE=255 TIMEOUT=2 DATA=0
```

Resulting application-level request bytes:

```text
01 04 ff 07 00
```

## Slot bit masks

Material slots use a 4-bit mask. The same bit layout is used by slot selectors,
material sensors, connection sensors, and several response payloads.

| Slot | Bit index | Mask byte |
|---|---:|---:|
| `A` | 0 | `0x01` |
| `B` | 1 | `0x02` |
| `C` | 2 | `0x04` |
| `D` | 3 | `0x08` |
| all slots | 0..3 | `0x0f` |
| no slot / special `0` | none | `0x00` |

```text
bit:   7 6 5 4 3 2 1 0
       - - - - D C B A
mask:          8 4 2 1
```

## Command payload enums

### `SET_BOX_MODE` data

Frame data:

```text
+------+------+
| SLOT | MODE |
+------+------+
```

| Value | Slot byte |
|---|---:|
| `A` | `0x01` |
| `B` | `0x02` |
| `C` | `0x04` |
| `D` | `0x08` |
| `0` | `0x00` |

| Mode | Byte |
|---|---:|
| `PRINT` | `0x00` |
| `IDLE` | `0x01` |

### `GET_FILAMENT_SENSOR_STATE` data

| Sensor bank | Byte | Meaning |
|---|---:|---|
| `MATERIAL` | `0x00` | Reads the material-present sensor mask. |
| `CONNECTIONS` | `0x01` | Reads the connection/five-way sensor mask. |

The response data byte is a slot bit mask using the A/B/C/D layout above.

### `SET_PRE_LOADING` data

Frame data:

```text
+-----------+--------+
| SLOT_MASK | ACTION |
+-----------+--------+
```

| Action | Byte |
|---|---:|
| `CLOSE` | `0x00` |
| `OPEN` | `0x01` |
| `RUN` | `0x02` |
| `TIGHT` | `0x03` |

### `CTRL_CONNECTION_MOTOR_ACTION` data

| Action | Byte |
|---|---:|
| `STOP` | `0x00` |
| `EXTRUDE` | `0x01` |
| `RETRUDE` | `0x02` |

### `MEASURING_WHEEL` data

| Action | Byte |
|---|---:|
| `CLEAN` | `0x00` |
| `GET` | `0x01` |

### `TIGHTEN_UP_ENABLE` data

| Value | Byte |
|---|---:|
| `ENABLE` | `0x00` |
| `DISABLE` | `0x01` |

### `RETRUDE_PROCESS` data

Frame data:

```text
+------+---------+
| SLOT | TRIGGER |
+------+---------+
```

| Trigger | Byte |
|---|---:|
| `BUFFER` | `0x00` |
| `MATERIAL` | `0x01` |

### `EXTRUDE_PROCESS_MODEL2` data

Frame data:

```text
+------+---------+
| SLOT | TRIGGER |
+------+---------+
```

| Trigger | Byte |
|---|---:|
| `CONNECTION` | `0x00` |
| `BUFFER` | `0x01` |

### `EXTRUDE_PROCESS` data

For supported box versions documented here (`version <= 1`), frame data is:

```text
+------+-------+---------+
| SLOT | STAGE | EXTRUDE |
+------+-------+---------+
```

| Field | Meaning |
|---|---|
| `SLOT` | A/B/C/D slot byte. |
| `STAGE` | Process stage byte. |
| `EXTRUDE` | Optional extrusion amount byte; `0x00` when not supplied. |

Observed stage values:

| Stage | Observed use |
|---:|---|
| `0` | Start / connection stage. |
| `3` | Retry stage used by load-retry handling. |
| `4` | Begin material extrusion part. |
| `5` | Poll extrusion progress. |
| `6` | Advance/recover during a stage-7 related loading event. |
| `7` | Final extrusion toward buffer-full detection. |

## Concrete request examples

All examples below show the application-level bytes before any lower-level serial framing.

### `GET_RFID ADDR=1 NUM=15`

```text
+------+------+-------+------+------+
| ADDR | LEN  | STATE | CMD  | DATA |
+------+------+-------+------+------+
| 01   | 04   | ff    | 01   | 0f   |
+------+------+-------+------+------+
```

```text
01 04 ff 01 0f
```

### `GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL`

```text
+------+------+-------+------+------+
| ADDR | LEN  | STATE | CMD  | DATA |
+------+------+-------+------+------+
| 01   | 04   | ff    | 07   | 00   |
+------+------+-------+------+------+
```

```text
01 04 ff 07 00
```

For `POSITION=CONNECTIONS`, the final byte is `01`:

```text
01 04 ff 07 01
```

### `SET_BOX_MODE ADDR=1 NUM=A MODE=PRINT`

```text
+------+------+-------+------+----------+------+
| ADDR | LEN  | STATE | CMD  | SLOT=A   | MODE |
+------+------+-------+------+----------+------+
| 01   | 05   | ff    | 03   | 01       | 00   |
+------+------+-------+------+----------+------+
```

```text
01 05 ff 03 01 00
```

### `SET_BOX_MODE ADDR=1 NUM=0 MODE=IDLE`

```text
01 05 ff 03 00 01
```

### Staged `EXTRUDE_PROCESS ADDR=1 NUM=A STAGE=0`

A staged load-start request with stage `0` has this application-level shape:

```text
+------+------+-------+------+----------+-------+---------+
| ADDR | LEN  | STATE | CMD  | SLOT=A   | STAGE | EXTRUDE |
+------+------+-------+------+----------+-------+---------+
| 01   | 06   | ff    | 0d   | 01       | 00    | 00      |
+------+------+-------+------+----------+-------+---------+
```

```text
01 06 ff 0d 01 00 00
```

The public `BOX_EXTRUDE_PROCESS` macro does not expose `STAGE`; for supported
versions documented here it sends stage `0x02`:

```gcode
BOX_EXTRUDE_PROCESS ADDR=1 NUM=A VELOCITY=2
```

```text
01 06 ff 0d 01 02 00
```

### `RETRUDE_PROCESS ADDR=1 NUM=A TRIGGER=MATERIAL`

```text
+------+------+-------+------+----------+----------+
| ADDR | LEN  | STATE | CMD  | SLOT=A   | TRIGGER  |
+------+------+-------+------+----------+----------+
| 01   | 05   | ff    | 0e   | 01       | 01       |
+------+------+-------+------+----------+----------+
```

```text
01 05 ff 0e 01 01
```

### `SET_PRE_LOADING ADDR=1 NUM=15 ACTION=OPEN`

The public macro has state side effects and guards, so it is not listed as a
pure direct macro above. `ADDR=0` or an omitted address applies the action to all
connected boxes. `POWER_ON=ENABLE|DISABLE` changes the wrapper's saved power-on
preloading behavior. The send may be skipped while printing or paused unless the
the wrapper uses a forced path. The actual serial payload shape is still
simple:

```text
01 05 ff 0a 0f 01
```

## Response state codes

The reply `STATE` byte maps to these symbolic results:

| State | Byte |
|---|---:|
| `OK` | `0x00` |
| `PARAMS_ERR` | `0x01` |
| `CRC_ERR` | `0x02` |
| `STATE_ERR` | `0x03` |
| `LENGTH_ERR` | `0x04` |
| `EXTRUDE_ERR1` | `0x05` |
| `EXTRUDE_ERR4` | `0x08` |
| `EXTRUDE_ERR5` | `0x09` |
| `EXTRUDE_ERR6` | `0x0a` |
| `EXTRUDE_ERR7` | `0x0b` |
| `EXTRUDE_ERR8` | `0x0c` |
| `EXTRUDE_ERR10` | `0x0d` |
| `EXTRUDE_ERR9` | `0x0e` |
| `RETRUDE_ERR1` | `0x13` |
| `RETRUDE_ERR2` | `0x14` |
| `RETRUDE_ERR3` | `0x15` |
| `RETRUDE_ERR4` | `0x16` |
| `RETRUDE_ERR5` | `0x17` |
| `RETRUDE_ERR6` | `0x19` |
| `RETRUDE_ERR7` | `0x1a` |
| `MOTOR_LOAD_ERR` | `0x22` |
| `UPDATE_STATE` | `0x30` |
| `FILAMENT_ERR` | `0x50` |
| `SPEED_ERR` | `0x51` |
| `ENWIND_ERR` | `0x52` |

## Decoded `OK` response data

| Command | Response data handling |
|---|---|
| `GET_RFID` | Parses text fragments like `A:<value>;`; valid slot keys are `A`..`D`. |
| `GET_REMAIN_LEN` | Reads four remain-length bytes and maps selected slots by request mask. |
| `MEASURING_WHEEL` | Public command actions are `CLEAN=0x00` and `GET=0x01`; measuring-wheel response decoding remains hardware validation needed. |
| `GET_FILAMENT_SENSOR_STATE` | Stores `DATA[0]` as either the `material` or `connections` slot mask. |
| `GET_VERSION_SN` | First three data bytes are version; remaining bytes up to the trailer are serial number. |
| `GET_BUFFER_STATE` | `0` means middle/not full; `>=1` means full. |
| `GET_BOX_STATE` | Reads dry/humidity bytes and reports humidity error when humidity is `0`. |

## Box mode values

The observed status model uses this mode map when reading box mode from state storage.
These are stored/comparison string values in observed state, not confirmed wire
bytes. The exact `GET_BOX_STATE` payload field that sets this value should be
validated on hardware:

| Stored mode value | Meaning |
|---:|---|
| `0` | `IDLE` |
| `1` | `PRELOADING` |
| `2` | `PRINTING` |
| `3` | `WRAPPERING` |
| `4` | `ERR` / `ERROR` |
| `5` | `TEST` |

## Slot update-state values

The `UPDATE_STATE` response carries four bytes, one per slot A/B/C/D.

```text
+---+---+---+---+
| A | B | C | D |
+---+---+---+---+
```

| Value | Meaning |
|---:|---|
| `0` | unchanged |
| `1` | insert |
| `2` | extract |
| `3` | completed |

## Stored sensor/box state surfaces

The serial layer feeds these higher-level state fields:

| State surface | Shape | Meaning |
|---|---|---|
| Box connection state | per box `T1`..`T4` | `connect`, `disconnect`, or literal string `None`. |
| Aggregate box state | single value | `connect`, `disconnect`, or literal string `None`. |
| Box mode | per box | `IDLE`, `PRELOADING`, `PRINTING`, `WRAPPERING`, `ERR`, `TEST`. |
| Material sensor mask | per box, 4-bit mask | Which slots have material present. |
| Connection sensor mask | per box, 4-bit mask | Which slots are detected at the connection/five-way sensor. |
| Buffer state | per box/global read | `0` = middle/not full, `>=1` = full. |
| RFID/vendor | per slot | `none`, `unknown`, or a 40-character RFID string. |
| Remain length | per slot | Stored as string/byte-derived remaining length. |
| Color/material type | per slot | Derived from RFID or saved state. |

## Timeout, retry, and version behavior

| Behavior | Detail |
|---|---|
| CRC retry | `CRC_ERR` causes `send_data` to resend. Most commands get up to 5 retry attempts. |
| Timeout | A timeout on `GET_BOX_STATE` returns `None`; other command timeouts call `timeout_process(addr)` and return `False`. |
| `COMMUNICATION_TEST` | Bypasses normal response-state parsing and returns the response data byte directly, or `None` on timeout. |
| `OK` | Dispatches command-specific response parsing, then returns `True`. |
| `UPDATE_STATE` | Handled as a response state and updates four slot states, then returns `True`. |
| Supported `BoxCfg.version` | Config constrains `version` to `0..1`. |
| `EXTRUDE_PROCESS` for `version <= 1` | Payload is `slot + stage + extrude`. If `extrude` is absent/falsey, the third byte is `0x00`. |
| Older or unknown protocol variants | Outside this reference unless validated on hardware. |

## Validation checklist

Useful things to confirm with a serial sniffer or the lower-level serial transport:

- Whether `serial_485` prepends a request `HEAD` byte.
- Whether `serial_485` appends CRC/checksum/trailer bytes.
- Exact checksum/trailer algorithm, if present.
- Exact `GET_BOX_STATE` response payload layout.
- Which `GET_BOX_STATE` payload byte updates box mode.
- Whether `GET_BUFFER_STATE` response data is stored as the later-read `buffer`
  state, or only logged.
- Exact measuring-wheel request/response subcommand indexing.
- Whether the public `BOX_EXTRUDE_PROCESS` macro intentionally ignores
  `VELOCITY` in the packet for `version <= 1`.

## Known uncertainties

These parts should be validated if you are not using the existing serial transport:

| Area | Confidence | Notes |
|---|---|---|
| Complete wire frame | Medium | The application layer uses `ADDR/LENGTH/STATE/CMD/DATA`, while replies include a leading `HEAD`. The lower layer may add/remove head/checksum bytes. |
| Response trailer/checksum | Medium-low | Some response handling suggests a trailing byte exists. Its exact checksum/end-marker meaning should be validated if you do not use the existing serial transport. |
| `GET_BOX_STATE` payload layout | Low | The visible mode map is known, but the exact `GET_BOX_STATE` response byte that updates mode should be validated. |
| `GET_BUFFER_STATE` storage | Low | The response value is `0` = middle and `>=1` = full. Cache/status update behavior should be validated. |
| `MEASURING_WHEEL` response decode condition | Low | Public commands use `CLEAN=0x00`, `GET=0x01`; float response decoding should be validated on hardware. |
| `EXTRUDE_PROCESS` stage meanings | Medium | Stage numbers are directly observed; human-readable meanings are compatibility labels based on surrounding behavior. |
| Reserved commands | Medium | `CTRL_MATERIAL_MOTOR_ACTION`, `SET_MOTOR_SPEED`, and `GET_HARDWARE_STATUS` are present in the command table but are not documented for normal operation. |
