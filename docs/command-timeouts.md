# Material box command timeouts

## Scope

This document records observed per-command timeout values used by the
material-box serial wrapper. Values are application-level timeouts passed to the
serial request/response call. They are not necessarily physical bus timeouts or
firmware watchdog values.

See also: [Protocol reference](protocol-reference.md) and
[Serial communication interface](serial-protocol.md).

## Timeout constants

| Name | Value | Meaning |
|---|---:|---|
| Short | 2 s | Default for simple queries and simple control commands. |
| Long | 150 s | Long-running motor/process command timeout. |
| Longer | 300 s | Base timeout for pre-loading `RUN`/`TIGHT` per selected slot. |
| Max | 3600 s | Used by `GET_BOX_STATE`; also affects retry-budget selection. |

## Per-command timeout table

| Command | ID | Timeout | Notes / confidence |
|---|---:|---:|---|
| `CREATE_CONNECT` | `0x00` | 2 s | High. |
| `GET_RFID` | `0x01` | 2 s | High. |
| `GET_REMAIN_LEN` | `0x02` | 2 s | High. |
| `SET_BOX_MODE` | `0x03` | 2 s | High. |
| `GET_BUFFER_STATE` | `0x04` | 2 s | High. |
| `CTRL_MATERIAL_MOTOR_ACTION` | `0x05` | 2 s | Medium: timeout known, command reserved/not documented for normal operation. |
| `CTRL_CONNECTION_MOTOR_ACTION` | `0x06` | 2 s | High. |
| `GET_FILAMENT_SENSOR_STATE` | `0x07` | 2 s | High. |
| `SET_MOTOR_SPEED` | `0x08` | 2 s | Medium: timeout known, command reserved/not documented for normal operation. |
| `GET_BOX_STATE` | `0x09` | 3600 s | High. Timeout returns no data instead of normal timeout failure. |
| `SET_PRE_LOADING CLOSE` | `0x0a` | 2 s | High. Applies when action byte is `0x00`. |
| `SET_PRE_LOADING OPEN` | `0x0a` | 2 s | High. Applies when action byte is `0x01`. |
| `SET_PRE_LOADING RUN` | `0x0a` | `300 s × selected slot count` | High. Send may be skipped if local filament sensor is detecting filament. |
| `SET_PRE_LOADING TIGHT` | `0x0a` | `300 s × selected slot count` | High. Send may be skipped if local filament sensor is detecting filament. |
| `MEASURING_WHEEL` | `0x0b` | 2 s | High for timeout; response decode remains uncertain. |
| `TIGHTEN_UP_ENABLE` | `0x0c` | 150 s | High. |
| `EXTRUDE_PROCESS` | `0x0d` | 15 s | High. Staged loading may call it repeatedly. |
| `RETRUDE_PROCESS` | `0x0e` | 150 s | High. |
| `EXTRUDE_PROCESS_MODEL2` | `0x0f` | 150 s | High. |
| `GET_VERSION_SN` | `0x10` | 2 s | High. |
| `GET_HARDWARE_STATUS` | `0x11` | 2 s | Medium: timeout known, command reserved/not documented for normal operation. |
| `COMMUNICATION_TEST` | `0x55` | macro-supplied; default 0.1 s | High. Public macro clamps supplied timeout to `0.0`..`1.0` seconds. |

## Retry behavior tied to timeouts

Observed retry selection:

```text
if timeout is greater than 3600 seconds:
  retry budget is 1
else:
  retry budget is 5
```

A `CRC_ERR` response consumes the retry budget and resends. Other response-state
errors do not use the CRC retry loop.

No documented public command uses a timeout greater than 3600 seconds, so the
documented command set effectively gets up to five CRC retries. `GET_BOX_STATE`
uses exactly 3600 seconds and therefore remains in the five-retry category.

## Timeout failure behavior

| Case | Behavior |
|---|---|
| `GET_BOX_STATE` no response | Return no data to caller. Higher-level heartbeat/recovery logic may handle repeated misses. |
| `COMMUNICATION_TEST` no response | Return no data for that iteration and increment the diagnostic timeout count. |
| Any other command no response | Run wrapper timeout handling for the address and fail the command. |

## Dynamic `SET_PRE_LOADING` timeout details

For `SET_PRE_LOADING`, the timeout is not always the table default:

| Action byte | Public action | Timeout rule |
|---:|---|---|
| `0x00` | `CLOSE` | 2 s |
| `0x01` | `OPEN` | 2 s |
| `0x02` | `RUN` | 300 s multiplied by number of selected low-nibble slot bits. |
| `0x03` | `TIGHT` | 300 s multiplied by number of selected low-nibble slot bits. |

Examples:

| Slot mask | Selected slots | `RUN`/`TIGHT` timeout |
|---:|---:|---:|
| `0x01` | 1 | 300 s |
| `0x03` | 2 | 600 s |
| `0x07` | 3 | 900 s |
| `0x0f` | 4 | 1200 s |

Hardware-validation-needed: whether firmware uses the same per-slot budget or
whether this is only a conservative host-side wait.
