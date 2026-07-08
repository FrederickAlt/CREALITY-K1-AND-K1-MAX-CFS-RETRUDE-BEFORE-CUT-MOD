# Material box serial protocol reference

## Scope and confidence

This is a protocol-focused interoperability reference for the material-box
interface. It describes the application-level frame body passed to the configured
serial transport, observed command identifiers, request payload byte layouts,
parsed response shapes, and known response state meanings.

It does **not** document material-change workflows, motion workflows, UI state,
or wrapper policy except where those details affect interoperability.

Confidence levels used below:

| Confidence | Meaning |
|---|---|
| High | Directly exercised by a public command or observed response handling. |
| Medium | Command id or enum is known, but response or behavior needs hardware validation. |
| Low | Inferred from observable behavior and should be validated before use. |

## Application-level request frame

The wrapper builds this frame body before passing it to the lower serial layer:

```text
ADDR | LENGTH | REQUEST_STATE | CMD | PAYLOAD...
```

| Field | Size | Meaning | Confidence |
|---|---:|---|---|
| `ADDR` | 1 byte | Box address. Normal public range is `0x01`..`0x04`; one debug extrude macro accepts `0x01`..`0x09`. | High |
| `LENGTH` | 1 byte | `3 + payload-byte-count`: one command byte plus state/command overhead used by this application layer. | High |
| `REQUEST_STATE` | 1 byte | Normally `0xff`. The raw-ish debug macro can supply another value. | High |
| `CMD` | 1 byte | Command id from the table below. | High |
| `PAYLOAD` | variable | Command-specific bytes. | High |

Open items for hardware validation when not using the existing serial transport:

- Whether the lower serial layer adds a physical head byte before this body.
- Whether the lower serial layer adds a checksum, CRC, or trailer after this body.
- Whether the response trailer byte is checksum, terminator, or another
  lower-layer artifact.

## Application-level response frame

Responses are interpreted with these byte positions:

```text
HEAD | ADDR | LENGTH | RESPONSE_STATE | CMD | DATA... | optional/trailing byte?
```

| Position | Field | Meaning | Confidence |
|---:|---|---|---|
| 0 | `HEAD` | Present in replies; not interpreted by the wrapper. | Medium |
| 1 | `ADDR` | Box address. | High |
| 2 | `LENGTH` | Response length byte used to bound variable fields. | High |
| 3 | `RESPONSE_STATE` | Result/error/update code. | High |
| 4 | `CMD` | Echoed command id; used to choose response handling when state is `OK`. | High |
| 5... | `DATA` | Command-specific response data. Some response handling stops before the last byte. | High for parsed fields, medium for trailer meaning |

Pseudocode for normal handling:

```text
send request frame with command timeout
if no response:
  GET_BOX_STATE returns no data
  other commands run timeout handling and fail
if command is COMMUNICATION_TEST:
  return first response data byte, or no data on timeout
map response state byte to a symbolic state
if state is OK:
  parse DATA according to echoed command id
if state is CRC_ERR:
  retry until retry budget is exhausted
otherwise:
  apply command/state-specific error handling
```

## Command id and request payload table

This table is the observed command-id inventory. Commands marked reserved are
known ids without a public high-level command in this documentation set.

| Command | ID | Request payload bytes | Public direct macro | Timeout | Confidence / notes |
|---|---:|---|---|---:|---|
| `CREATE_CONNECT` | `0x00` | none | `BOX_CREATE_CONNECT` | 2 s | High. Ignored by macro while printing. |
| `GET_RFID` | `0x01` | `SLOT_MASK` | `BOX_GET_RFID` | 2 s | High. Masked to low 4 bits. |
| `GET_REMAIN_LEN` | `0x02` | `SLOT_MASK` | `BOX_GET_REMAIN_LEN` | 2 s | High. Masked to low 4 bits. |
| `SET_BOX_MODE` | `0x03` | `SLOT`, `MODE` | `BOX_SET_BOX_MODE` | 2 s | High. Public send values are `PRINT` and `IDLE`. |
| `GET_BUFFER_STATE` | `0x04` | none | `BOX_GET_BUFFER_STATE` | 2 s | High for response meaning; cache/storage behavior should be validated. |
| `CTRL_MATERIAL_MOTOR_ACTION` | `0x05` | unknown | none | 2 s | Medium. Reserved/not documented for normal operation. |
| `CTRL_CONNECTION_MOTOR_ACTION` | `0x06` | `ACTION` | `BOX_CTRL_CONNECTION_MOTOR_ACTION` | 2 s | High. |
| `GET_FILAMENT_SENSOR_STATE` | `0x07` | `SENSOR_BANK` | `BOX_GET_FILAMENT_SENSOR_STATE` | 2 s | High. |
| `SET_MOTOR_SPEED` | `0x08` | unknown | none | 2 s | Medium. Reserved/not documented for normal operation. |
| `GET_BOX_STATE` | `0x09` | none | `BOX_GET_BOX_STATE` | 3600 s | High for request and dry/humidity parsing; full payload needs hardware validation. |
| `SET_PRE_LOADING` | `0x0a` | `SLOT_MASK`, `ACTION` | `BOX_SET_PRE_LOADING` | dynamic | High. Timeout depends on action and selected slots. |
| `MEASURING_WHEEL` | `0x0b` | `ACTION` | `BOX_MEASURING_WHEEL` | 2 s | Medium. Request is clear; float response condition is uncertain. |
| `TIGHTEN_UP_ENABLE` | `0x0c` | `ENABLE` | `BOX_TIGHTEN_UP_ENABLE` | 150 s | High. |
| `EXTRUDE_PROCESS` | `0x0d` | version-dependent; normally `SLOT`, `STAGE`, `EXTRUDE_AMOUNT` | `BOX_EXTRUDE_PROCESS` | 15 s | High for supported versions documented here. |
| `RETRUDE_PROCESS` | `0x0e` | `SLOT`, `TRIGGER` | `BOX_RETRUDE_PROCESS` | 150 s | High. |
| `EXTRUDE_PROCESS_MODEL2` | `0x0f` | `SLOT`, `TRIGGER` | `BOX_EXTRUDE_2_PROCESS` | 150 s | High. |
| `GET_VERSION_SN` | `0x10` | none | `BOX_GET_VERSION_SN` | 2 s | High. |
| `GET_HARDWARE_STATUS` | `0x11` | unknown | none | 2 s | Medium. Reserved/not documented for normal operation. |
| `COMMUNICATION_TEST` | `0x55` | `TEST_BYTE` | `BOX_COMMUNICATION_TEST` | macro-supplied, default 0.1 s | High. Bypasses normal state handling. |

## Shared byte enums

### Slot masks and slot bytes

Material slots use a low-nibble bit mask.

| Slot selector | Byte |
|---|---:|
| none / special `0` | `0x00` |
| A | `0x01` |
| B | `0x02` |
| C | `0x04` |
| D | `0x08` |
| all slots | `0x0f` |

The same low-nibble layout is used for slot selectors, selected-slot masks,
material sensor masks, connection sensor masks, and several update payloads.

### `SET_BOX_MODE` payload

```text
SLOT | MODE
```

| `MODE` | Byte | Notes |
|---|---:|---|
| `PRINT` | `0x00` | Send value only. Do not confuse with reported/stored mode values. |
| `IDLE` | `0x01` | Send value only. Do not confuse with reported/stored mode values. |

### Reported/stored box mode values

The observed state model uses these values when interpreting stored box mode.
The exact `GET_BOX_STATE` response byte that populates this value remains
hardware validation is needed.

| Stored value | Meaning | Confidence |
|---:|---|---|
| `0` | `IDLE` | Medium |
| `1` | `PRELOADING` | Medium |
| `2` | `PRINTING` | Medium |
| `3` | `WRAPPERING` | Medium |
| `4` | `ERR` / error | Medium |
| `5` | `TEST` | Medium |

### `CTRL_CONNECTION_MOTOR_ACTION` payload

```text
ACTION
```

| Action | Byte |
|---|---:|
| `STOP` | `0x00` |
| `EXTRUDE` | `0x01` |
| `RETRUDE` | `0x02` |

### `GET_FILAMENT_SENSOR_STATE` payload

```text
SENSOR_BANK
```

| Sensor bank | Byte | Response data meaning |
|---|---:|---|
| `MATERIAL` | `0x00` | One slot-mask byte: material-present sensors. |
| `CONNECTIONS` | `0x01` | One slot-mask byte: connection/five-way sensors. |

### `SET_PRE_LOADING` payload

```text
SLOT_MASK | ACTION
```

| Action | Byte | Timeout behavior |
|---|---:|---|
| `CLOSE` | `0x00` | 2 s |
| `OPEN` | `0x01` | 2 s |
| `RUN` | `0x02` | 300 s times number of selected slots, unless local filament sensor blocks send |
| `TIGHT` | `0x03` | 300 s times number of selected slots, unless local filament sensor blocks send |

Macro behavior notes:

- Public macro `ADDR=0` broadcasts at wrapper level by sending one command to
  each connected address, not by sending wire address `0x00`.
- The command is skipped while printing.
- In the literal pause state, it is skipped unless a wrapper-managed forced path
  is used.
- `POWER_ON=ENABLE|DISABLE` changes wrapper saved behavior and is not sent as a
  serial payload byte.

### `MEASURING_WHEEL` payload

```text
ACTION
```

| Action | Byte | Notes |
|---|---:|---|
| `CLEAN` | `0x00` | Public request value. |
| `GET` | `0x01` | Public request value. Response decode uncertainty remains; see response section. |

### `TIGHTEN_UP_ENABLE` payload

```text
ENABLE
```

| Value | Byte |
|---|---:|
| `ENABLE` | `0x00` |
| `DISABLE` | `0x01` |

### `RETRUDE_PROCESS` payload

```text
SLOT | TRIGGER
```

| Trigger | Byte |
|---|---:|
| `BUFFER` | `0x00` |
| `MATERIAL` | `0x01` |

`SLOT` accepts A/B/C/D and special `0`.

### `EXTRUDE_PROCESS_MODEL2` payload

```text
SLOT | TRIGGER
```

| Trigger | Byte |
|---|---:|
| `CONNECTION` | `0x00` |
| `BUFFER` | `0x01` |

`SLOT` accepts A/B/C/D.

### `EXTRUDE_PROCESS` payload

For supported configuration versions (`0` and `1`):

```text
SLOT | STAGE | EXTRUDE_AMOUNT
```

| Field | Byte meaning |
|---|---|
| `SLOT` | A/B/C/D slot byte. |
| `STAGE` | Process stage. Public macro defaults to `0x02`; normal loading uses several stages. |
| `EXTRUDE_AMOUNT` | One byte. Sent as `0x00` when no explicit amount is supplied. |

Observed stage values:

| Stage | Observed use | Confidence |
|---:|---|---|
| `0x00` | Start/connection stage before material loading. | Medium |
| `0x02` | Public debug macro default for supported versions. | High |
| `0x03` | Retry command used by extrude retry handling. | Medium |
| `0x04` | Begin material extrusion part. | Medium |
| `0x05` | Poll extrusion progress. | Medium |
| `0x06` | Advance/recover during stage-7 related loading-event handling. | Medium |
| `0x07` | Final extrusion toward buffer-full detection. | Medium |

Only configured versions `0` and `1` are documented here. Older or unknown
protocol variants are outside this reference unless validated on hardware.

## Response state codes

| Symbolic state | Byte | Wrapper behavior | Confidence |
|---|---:|---|---|
| `OK` | `0x00` | Parse command-specific response data, then command succeeds. | High |
| `PARAMS_ERR` | `0x01` | Report parameter error and fail. | High |
| `CRC_ERR` | `0x02` | Retry until retry budget exhausted, then fail. | High |
| `STATE_ERR` | `0x03` | `GET_BOX_STATE` may be treated as success when reporting is enabled; other commands report state error and fail. | High |
| `LENGTH_ERR` | `0x04` | Known state code; generic failure handling. | Medium |
| `EXTRUDE_ERR1` | `0x05` | Extrude error branch; may report slot-specific extrusion failure. | High |
| `EXTRUDE_ERR4` | `0x08` | Shares filament-error handling branch. | High |
| `EXTRUDE_ERR5` | `0x09` | Reports a buffer-not-triggered style extrusion error. | High |
| `EXTRUDE_ERR6` | `0x0a` | Extrusion/load failure class. | High |
| `EXTRUDE_ERR7` | `0x0b` | Reports joint/path error when applicable, then fails. | High |
| `EXTRUDE_ERR8` | `0x0c` | Extrusion/load failure class. | High |
| `EXTRUDE_ERR10` | `0x0d` | Extrusion/load failure class. | High |
| `EXTRUDE_ERR9` | `0x0e` | Extrusion/load failure class. | High |
| `RETRUDE_ERR1` | `0x13` | Warns about failing to retreat to buffer-empty limit, then fails. | High |
| `RETRUDE_ERR2` | `0x14` | Reports failed-to-exit-connections style retrude error, then fails. | High |
| `RETRUDE_ERR3` | `0x15` | Reports unspecified/multiple-connection retrude error, then fails. | High |
| `RETRUDE_ERR4` | `0x16` | Known state code; generic failure handling. | Medium |
| `RETRUDE_ERR5` | `0x17` | Known state code; generic failure handling. | Medium |
| `RETRUDE_ERR6` | `0x19` | Reports failed-to-exit-connections style retrude error, then fails. | High |
| `RETRUDE_ERR7` | `0x1a` | Reports failed-to-exit-connections style retrude error, then fails. | High |
| `MOTOR_LOAD_ERR` | `0x22` | Reports motor load error when reporting is enabled, then fails. | High |
| `UPDATE_STATE` | `0x30` | Treats next four data bytes as A/B/C/D update-state values and succeeds. | High |
| `FILAMENT_ERR` | `0x50` | Shares filament-error handling branch. | High |
| `SPEED_ERR` | `0x51` | May record empty-print style error when a material command is active, then fails. | High |
| `ENWIND_ERR` | `0x52` | May record empty-print style error and decrement filament-useup state, then fails. | High |

## Command-specific `OK` response data

### `CREATE_CONNECT`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `GET_RFID`

Response data is parsed as text fragments in this shape:

```text
A:<value>;B:<value>;C:<value>;D:<value>;
```

Only keys `A`, `B`, `C`, and `D` are meaningful slot keys. Values are treated
as strings. The wrapper then validates RFID values and updates
vendor/color/material fields.

Known higher-level RFID handling:

| RFID value class | Effect |
|---|---|
| valid non-empty RFID | Stored as vendor; material type and color are sliced from fixed positions in the RFID string. |
| `none` | Stored as no material identity for color/material fields. |
| invalid or unexpected string | May restore saved fields or mark fields as `unknown`; exact RFID validity rules are outside this protocol reference. |

Hardware validation needed: exact firmware text encoding for empty slots and
invalid cards.

### `GET_REMAIN_LEN`

Response handling reads four data bytes as A/B/C/D remaining-length values:

```text
A_REMAIN | B_REMAIN | C_REMAIN | D_REMAIN
```

Only slots selected by the request `SLOT_MASK` are returned to higher-level
state. Values are stored as decimal strings derived from single response bytes.

Hardware validation needed: physical units and whether values can exceed one
byte in firmware variants.

### `SET_BOX_MODE`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `GET_BUFFER_STATE`

The first response data byte is interpreted as:

| Data byte | Meaning |
|---:|---|
| `0x00` | Middle / not full. |
| `0x01` or greater | Full. |

The public command reports this value. Independent wrappers should decide
explicitly whether to cache it; status-cache behavior should be validated on the
target system.

### `CTRL_CONNECTION_MOTOR_ACTION`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `GET_FILAMENT_SENSOR_STATE`

The first response data byte is a slot mask. The request `SENSOR_BANK` selects
which state surface is updated:

| Request bank | Response data byte |
|---|---|
| `MATERIAL` | Material-present mask using A/B/C/D bits. |
| `CONNECTIONS` | Connection/five-way mask using A/B/C/D bits. |

### `GET_BOX_STATE`

Only the first two response data bytes are currently documented with confidence:

```text
DRY_VALUE | HUMIDITY_VALUE | remaining payload unknown...
```

Both first bytes are interpreted as signed 8-bit values. If `HUMIDITY_VALUE` is
zero, the wrapper reports a dry/humidity error.

Hardware validation needed:

- complete `GET_BOX_STATE` payload layout;
- which response byte updates reported/stored box mode;
- whether connection status or environment state uses additional data bytes.

### `SET_PRE_LOADING`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `MEASURING_WHEEL`

The public action bytes and observed response decoding need hardware validation:

- Public commands use `CLEAN=0x00` and `GET=0x01`.
- Some observed behavior suggests a four-byte float response may only be decoded
  under a different action/index condition.

When a float response is present, the likely shape is a 32-bit value assembled
from four data bytes and interpreted as a single-precision float.

Likely response shape for the float branch:

```text
FLOAT_BYTE_3 | FLOAT_BYTE_2 | FLOAT_BYTE_1 | FLOAT_BYTE_0
```

Confidence is low until verified against a serial trace.

### `TIGHTEN_UP_ENABLE`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `EXTRUDE_PROCESS`

No `OK` response payload is required. The response state itself is the important
outcome. For this command, wrappers should retain the symbolic response state for
staged extrusion decisions.

Important non-OK states include extrusion errors, filament errors, motor load,
speed, and enwind errors listed in the response-state table.

### `RETRUDE_PROCESS`

No `OK` response payload is required. The response state itself is the important
outcome. Retrude-specific error states are listed in the response-state table.

### `EXTRUDE_PROCESS_MODEL2`

No command-specific `OK` payload is required. A normal `OK` response is
accepted as success.

### `GET_VERSION_SN`

Response handling treats data bytes as:

```text
VERSION_CHAR_0 | VERSION_CHAR_1 | VERSION_CHAR_2 | SN_CHARS... | trailing byte not included in SN
```

| Field | Meaning |
|---|---|
| First 3 data bytes | Version string characters. |
| Data bytes after the first 3, excluding final trailing byte | Serial-number string characters. |

Hardware validation needed: exact meaning of the excluded final byte.

### `COMMUNICATION_TEST`

This command uses special response handling. The command returns the
first response data byte directly. The diagnostic loop expects:

```text
returned byte == (sent test byte + 1) modulo 256
```

Timeout returns no data for that iteration.

## `UPDATE_STATE` response payload

`UPDATE_STATE` is a response state, not a command id. When received, the wrapper
reads four data bytes:

```text
A_UPDATE | B_UPDATE | C_UPDATE | D_UPDATE
```

| Value | Meaning | Confidence |
|---:|---|---|
| `0x00` | unchanged | High |
| `0x01` | insert | High |
| `0x02` | extract | High |
| `0x03` | completed | High |

## Retry and timeout behavior summary

| Case | Behavior |
|---|---|
| Normal commands with timeout `<= 3600 s` | Up to 5 CRC retry attempts. |
| Commands with timeout greater than `3600 s` | One attempt. No documented public command uses greater than max. |
| No response for `GET_BOX_STATE` | Return no data; heartbeat/recovery logic may mark disconnect after repeated failures. |
| No response for other commands | Run timeout handling for the address and fail. |
| `COMMUNICATION_TEST` no response | Return no data to diagnostic loop; increment timeout counter. |

## Hardware-validation checklist

To retire the remaining uncertainties, capture request and response bytes for:

1. `GET_RFID` for empty, unknown, and valid RFID slots.
2. `GET_REMAIN_LEN` for known remaining lengths.
3. `GET_BOX_STATE` across idle, preloading, printing, error, and disconnected
   states.
4. `GET_BUFFER_STATE` before and after buffer-full detection.
5. `MEASURING_WHEEL CLEAN` followed by `MEASURING_WHEEL GET` with known
   movement.
6. `EXTRUDE_PROCESS` stages `0`, `2`, `4`, `5`, `6`, and `7` during a controlled
   load.
7. Any firmware support for `CTRL_MATERIAL_MOTOR_ACTION`, `SET_MOTOR_SPEED`,
   and `GET_HARDWARE_STATUS`.
