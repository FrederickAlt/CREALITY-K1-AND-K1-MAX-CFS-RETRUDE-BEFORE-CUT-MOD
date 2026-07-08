# Sensors and hardware integration

## Scope

This document explains the hardware-facing pieces of the material-box wrapper:
serial bus discovery, box sensors, the local filament sensor, cutter sensor,
buffer state, measuring wheel, fans, and useful diagnostic macros.

It is written for readers who do **not** have access to source code. Details are
described as behavior, interfaces, and safety constraints.

For byte-level protocol details, see [`serial-protocol.md`](serial-protocol.md).
For configuration options and calibration settings, see
[`configuration.md`](configuration.md). For how these readings are stored in
status/state, see [`state-model.md`](state-model.md). For detailed unload,
cutter, buffer, and measuring-wheel decision paths, see
[`unload-cutter-sensor-reference.md`](unload-cutter-sensor-reference.md). For
hard-coded names such as `filament_sensor_2`, `fan0`, persisted state files, and
the flushing sign file, see [`runtime-reference.md`](runtime-reference.md).

## Hardware overview

```text
Klipper / wrapper
      |
      | serial_485 <bus>
      v
+----------------------+       +----------------------+
| material box T1      |  ...  | material box T4      |
| slots A/B/C/D        |       | slots A/B/C/D        |
| material sensors     |       | material sensors     |
| connection sensors   |       | connection sensors   |
| buffer sensor/state  |       | buffer sensor/state  |
| RFID / remain length |       | RFID / remain length |
+----------------------+       +----------------------+

Printer-side hardware:
  - local filament switch sensor, optional
  - cutter/cut-return switch, optional
  - fan0 / fan2 output pins, optional
  - toolhead, heater, and motion axes
```

The wrapper combines printer-side sensors with box-side sensors. A box may report
material in a slot even when the printer-side filament sensor does not currently
see material at the toolhead path, and vice versa.

## Serial bus and physical boxes

The `[box]` setting `bus` selects the Klipper serial object:

```ini
[box]
bus: serial485
```

This corresponds to a runtime object named:

```text
serial_485 serial485
```

Known bring-up caveat: observed discovery behavior also checks for the fixed
serial device path `/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0` before
marking a newly online box connected. A differently named adapter may require
firmware/profile adjustment even if `bus` points at the right Klipper object.

The wrapper supports four physical box addresses:

| Box | Serial address |
|---|---:|
| `T1` | `1` |
| `T2` | `2` |
| `T3` | `3` |
| `T4` | `4` |

Each box has four slots: `A`, `B`, `C`, and `D`.

## Box connection heartbeat

A background heartbeat repeatedly checks connected boxes. Conceptually:

```text
for each heartbeat cycle:
    discover newly online boxes
    query one connected box's state
    track repeated failures
    mark box disconnected after repeated timeout
    update aggregate connect/disconnect state
```

Connection status is state, not a physical sensor by itself. The wrapper infers
it from serial communication success/failure and external online indications.

Useful direct macro:

```gcode
BOX_GET_BOX_STATE ADDR=1
```

See [`state-model.md`](state-model.md#state-machine-1-box-connection-state) for
the connection-state model.

## Slot bit masks

Box-side slot sensors are represented as four-bit masks.

```text
bit:   7 6 5 4 3 2 1 0
       - - - - D C B A
mask:          8 4 2 1
```

| Slot | Bit | Mask |
|---|---:|---:|
| `A` | 0 | `0x01` |
| `B` | 1 | `0x02` |
| `C` | 2 | `0x04` |
| `D` | 3 | `0x08` |

Examples:

| Mask | Meaning |
|---:|---|
| `0x00` | no slots detected |
| `0x01` | slot A detected |
| `0x02` | slot B detected |
| `0x03` | slots A and B detected |
| `0x08` | slot D detected |
| `0x0f` | all slots detected |

## Box material sensors

The material sensor bank answers:

```text
Which slots currently have material present at the box material sensor?
```

Direct diagnostic macro:

```gcode
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL
```

The returned value is stored as the box's material sensor mask. A slot is treated
as materially available when its bit is set in this mask.

Example:

```text
material mask = 0x05
```

means:

```text
slot A present
slot C present
slot B/D not present
```

This sensor bank is used by material availability checks, retrude decisions,
auto-refill candidate selection, and state synchronization.

## Box connection / five-way sensors

The connection sensor bank answers:

```text
Which slots are detected at the connection/five-way sensor?
```

Direct diagnostic macro:

```gcode
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=CONNECTIONS
```

There is also a convenience macro:

```gcode
BOX_GET_FIVE_WAY_STATE
```

It checks connected boxes and reports their connection/five-way state. It also
updates a per-box filament marker to `A`, `B`, `C`, `D`, or the literal string
`None` based on the first matching bit.

The wrapper uses connection sensor data for:

- deciding whether material is already in the connection path;
- retrude and recovery branches;
- detecting conflict between CFS/box mode and rack/five-way mode;
- waiting for filament-error tighten-up handling.

## Local printer filament sensor

The local filament sensor is configured with:

```ini
[box]
filament_sensor: filament_sensor_2
```

At runtime, this is looked up as:

```text
filament_switch_sensor filament_sensor_2
```

If no local filament sensor is configured, the wrapper treats local filament
detection as **true/present**. This prevents optional hardware from blocking box
flows.

Behavioral interpretation:

```text
if no local sensor is configured:
    local filament present = true
else:
    local filament present = value interpreted from the configured Klipper sensor
```

Compatibility caveat: observed behavior treats sensor values `<= 1` as
present/true and values `> 1` as absent/false. Do not assume this is a normal
boolean without checking the deployed sensor behavior.

Additional caveat: the configured `filament_sensor` name is used for lookup, but
several enable/disable commands are hard-coded to `filament_sensor_2`. Profiles
using a different sensor name may read the sensor correctly but fail to control
it through wrapper macros.

The local filament sensor is used before cutting, during retrude, during loading,
and to decide whether ending material needs to be flushed.

## Sensor conflict checks

The wrapper has a small conflict check between printer-side and five-way sensor
state.

Conceptually:

```text
filament_sensor = local printer filament sensor
five_way_sensor = any box connection/five-way sensor bit set

if print_type == "cfs":
    conflict = (five_way_sensor is false) and (filament_sensor is true)

if print_type == "rack":
    conflict = (five_way_sensor is true)
```

This is intended to detect inconsistent material-path state before starting or
choosing a print mode.

## Buffer state

The box exposes a buffer state through:

```gcode
BOX_GET_BUFFER_STATE ADDR=1
```

Observed interpretation:

| Value | Meaning |
|---:|---|
| `0` | middle / not full |
| `>= 1` | full |
| `None` | unknown or not read |

The wrapper intends to use buffer state during material loading and some retrude
recovery paths. For example, after pushing material forward, it checks whether
the buffer has become full enough to consider loading successful.

Known caveat: the command reports `middle`/`full`, but status-cache behavior may
differ from the immediate command response. Independent wrappers should decide
explicitly whether and how to cache buffer state.

## Measuring wheel

Detailed measuring-wheel blockage behavior and observed thresholds are covered
in [`unload-cutter-sensor-reference.md`](unload-cutter-sensor-reference.md#measuring-wheel-and-blockage-behavior).

The measuring wheel is controlled with:

```gcode
BOX_MEASURING_WHEEL NUM=1 ACTION=GET
BOX_MEASURING_WHEEL NUM=1 ACTION=CLEAN
```

Here `NUM` is the box address.

Actions:

| Action | Meaning |
|---|---|
| `GET` | Read current measuring-wheel value. |
| `CLEAN` | Clear/reset measuring-wheel value. |

The wrapper uses measuring-wheel deltas during extrusion/flush checks. The basic
idea is:

```text
read wheel before extrusion
extrude a segment
read wheel after extrusion
diff = current_wheel - previous_wheel
if diff crosses configured threshold:
    report possible nozzle/blockage problem
```

The threshold is configured as `diff_length`; see
[`configuration.md`](configuration.md#material-flow-and-flush-options).

Known uncertainty: public command actions are clear, but exact measuring-wheel
response decoding should be verified on hardware.

## Cutter / cut sensor

Detailed cutter motion, homing, blind-cut, sensor-confirmed, and validation
behavior is covered in
[`unload-cutter-sensor-reference.md`](unload-cutter-sensor-reference.md#cutter-path-behavior).

The cutter sensor is configured with:

```ini
[box]
switch_pin: ^your_cut_sensor_pin
```

If `switch_pin` is configured, the wrapper registers it as a button input and
tracks two pieces of cut state:

| State | Meaning |
|---|---|
| `cut_present` | Current switch state. |
| `cut_happened` | Sticky flag set when the switch has triggered; cleared manually by the wrapper. |

If `switch_pin` is **not** configured, the wrapper still performs cutter motion,
but it uses blind cut strokes and cannot confirm the cut sensor event.

### Cut flow with no cut sensor

```text
if local filament sensor says no filament:
    skip cut and return success

move to pre-cut/cut positions
repeat blind cut strokes
return to pre-cut/safe path
return success
```

### Cut flow with cut sensor

```text
if local filament sensor says no filament:
    skip cut and return success

clear sticky cut_happened flag
for up to 5 attempts:
    perform configured cut strokes
    wait briefly
    if cut_present indicates return/success:
        break
    otherwise move back toward box extrude position and retry

if cut_happened was set:
    clear flag and return success
else:
    report cut error and return failure
```

Useful macros:

| Macro | Purpose |
|---|---|
| `BOX_CUT_STATE` | Report current cut sensor state. |
| `BOX_MOVE_TO_CUT` | Run cut-position/cut movement flow. |
| `BOX_CUT_MATERIAL` | Close model fan and run the cut flow. |
| `BOX_CUT_HALL_ZERO` | Find cutter hall/zero position; registered only when `switch_pin` exists. |
| `BOX_CUT_HALL_TEST` | Test cutter hall movement; registered only when `switch_pin` exists. |
| `BOX_FIND_CUT_POS` | Search for a cut Y position using the cut sensor and save it. |

Position and velocity settings are documented in
[`configuration.md`](configuration.md#cutter-and-pre-cut-position-options).

## Fan outputs

The wrapper optionally interacts with these output pins:

| Output pin | Use |
|---|---|
| `fan0` | Model fan / cleaning fan. Used by blow, nozzle clean, and close-model-fan behavior. |
| `fan2` | Saved/restored when present. |

If the output pins are not defined, the wrapper logs that they are unavailable
and continues.

Useful macros:

| Macro | Purpose |
|---|---|
| `BOX_SAVE_FAN` | Save current fan output values and turn configured fans off. |
| `BOX_RESTORE_FAN` | Restore saved fan output values. |
| `BOX_BLOW` | Turn `fan0` on, wait, then turn it off. |

The wrapper often turns fans off before material-change or extrusion operations
and restores them afterward. A side-effect matrix for fan-writing commands is in
[`runtime-reference.md#side-effect-matrix-for-high-risk-command-families`](runtime-reference.md#side-effect-matrix-for-high-risk-command-families).

## Toolhead, heater, and homing dependencies

Although not sensors in the box, the wrapper depends heavily on printer motion
state.

Important behavior:

| Dependency | Use |
|---|---|
| homed X/Y | Required before most cut, clean, safe, and extrude-position moves. The wrapper homes X/Y in several flows if needed. |
| homed Z | Required for `z_down`; if Z is not homed, Z moves are skipped with a warning. |
| hotend `can_extrude` | Determines whether the wrapper can retract/extrude immediately or should heat first. |
| current toolhead position | Used for safe moves, Z restore tracking, and saving extrude position. |
| acceleration limits | Temporarily changed during cutting and cleaning, then restored. |

The wrapper emits `M400` frequently to wait for motion completion before reading
or changing hardware state.

## Dry/humidity response

The `GET_BOX_STATE` response includes dry/humidity-related bytes. The observed
behavior reports a humidity sensor error when the humidity value is `0`.

Known uncertainty: the visible behavior does not fully document how dry/humidity
values are stored or exposed beyond that error check.

## Diagnostic macro quick reference

Low-level hardware/sensor diagnostics:

| Macro | Example | Reads / affects |
|---|---|---|
| `BOX_GET_BOX_STATE` | `BOX_GET_BOX_STATE ADDR=1` | Box connection/mode/environment state. |
| `BOX_GET_FILAMENT_SENSOR_STATE` | `BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL` | Box material or connection sensor mask. |
| `BOX_GET_FIVE_WAY_STATE` | `BOX_GET_FIVE_WAY_STATE` | Connection/five-way sensor state across connected boxes. |
| `BOX_GET_BUFFER_STATE` | `BOX_GET_BUFFER_STATE ADDR=1` | Box buffer state. |
| `BOX_MEASURING_WHEEL` | `BOX_MEASURING_WHEEL NUM=1 ACTION=GET` | Measuring wheel. |
| `BOX_CUT_STATE` | `BOX_CUT_STATE` | Cutter switch state. |
| `BOX_CUT_HALL_ZERO` | `BOX_CUT_HALL_ZERO` | Cutter hall/zero calibration, if cut sensor exists. |
| `BOX_CUT_HALL_TEST` | `BOX_CUT_HALL_TEST` | Cutter hall test, if cut sensor exists. |
| `BOX_FIND_CUT_POS` | `BOX_FIND_CUT_POS` | Cut-position search and save. |
| `BOX_SAVE_FAN` | `BOX_SAVE_FAN` | Save/disable fan outputs. |
| `BOX_RESTORE_FAN` | `BOX_RESTORE_FAN` | Restore fan outputs. |
| `BOX_BLOW` | `BOX_BLOW` | Toggle `fan0` for a timed blow operation. |

## Suggested hardware validation checklist

For a machine bring-up, validate in this order:

1. **Serial bus**
   - Run `BOX_GET_BOX_STATE ADDR=1` for each expected box address as a direct
     communication check.
   - Let the heartbeat/discovery logic run and confirm wrapper connection state
     changes from unknown/disconnect to connected. `BOX_GET_BOX_STATE` alone
     does not necessarily mark a box connected.

2. **Slot material sensors**
   - Insert material in slot A.
   - Run `BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL`.
   - Confirm bit `0x01` appears.

3. **Connection/five-way sensors**
   - Move material into the connection path.
   - Run `BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=CONNECTIONS`.
   - Confirm the matching slot bit appears.

4. **Local filament sensor**
   - Confirm the configured `filament_sensor` reports expected presence/absence
     through the printer UI or Klipper status.
   - Be aware of the observed `<= 1` present/true behavior.

5. **Buffer state**
   - Run `BOX_GET_BUFFER_STATE ADDR=1` before and after loading material.
   - Confirm the value changes as expected for your hardware.

6. **Cut sensor**
   - Run `BOX_CUT_STATE` before and after manually triggering the cut switch.
   - If configured, use `BOX_CUT_HALL_TEST` cautiously.

7. **Measuring wheel**
   - Run `BOX_MEASURING_WHEEL NUM=1 ACTION=CLEAN`.
   - Move/extrude material.
   - Run `BOX_MEASURING_WHEEL NUM=1 ACTION=GET` and confirm a plausible value.

8. **Fans**
   - Run `BOX_SAVE_FAN`, verify fans turn off.
   - Run `BOX_RESTORE_FAN`, verify prior fan values return.

## Known hardware/sensor uncertainties

| Area | Notes |
|---|---|
| Local filament sensor truth value | Observed behavior treats sensor values `<= 1` as present/true and `> 1` as absent/false. Verify on the target machine. |
| Buffer storage | `GET_BUFFER_STATE` reports the value, but status-cache behavior should be validated. |
| Measuring-wheel decode | Public action bytes are clear, but response decoding should be verified on hardware. |
| Dry/humidity payload | A humidity value of `0` triggers an error, but complete payload layout/storage should be hardware-validated. |
| Blind cutter mode | If `switch_pin` is absent, cuts are motion-only and cannot be confirmed by the wrapper. |
| Fan availability | `fan0` and `fan2` are optional output pins; absence is tolerated. |
| Serial discovery path | New-box discovery also checks a hard-coded `/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0` path in observed behavior. |
