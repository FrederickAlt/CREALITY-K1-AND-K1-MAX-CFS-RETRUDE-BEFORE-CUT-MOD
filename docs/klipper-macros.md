# Klipper / G-code macro interface

## Scope

This document catalogs the G-code commands exposed by the material-box wrapper.
It is written for readers using public commands, device-visible behavior, and deployed configuration.

It focuses on:

- command syntax;
- parameters and defaults;
- what each command does at a high level;
- important side effects;
- whether a command is direct/low-level or workflow-oriented.

For serial byte layouts, see [`serial-protocol.md`](serial-protocol.md). For
configuration commands and option meanings, see
[`configuration.md`](configuration.md). For state names and mappings, see
[`state-model.md`](state-model.md). For hardware/sensor details, see
[`sensors-and-hardware.md`](sensors-and-hardware.md). For compatibility caveats that affect macro error handling, see
[`compatibility-caveats.md`](compatibility-caveats.md). For a full public command
inventory, hard-coded runtime names, and side-effect matrix, see
[`runtime-reference.md`](runtime-reference.md).

## Command categories

| Category | Description |
|---|---|
| Direct serial commands | Mostly translate one G-code macro to one box serial command. |
| State/mapping commands | Inspect or mutate wrapper state such as `Tnn_map`, material identity, or auto-refill. |
| Configuration/calibration commands | Modify `box.cfg`-backed settings or move to configured calibration positions. |
| Sensor/hardware diagnostics | Query box sensors, cut sensor, measuring wheel, fans, and hardware state. |
| Motion/cleaning commands | Move to box positions, cut, clean nozzle, blow fan, save/restore fans. |
| Material/tool-change commands | Load/retrude/flush/switch material. These can move axes, heat, cut, and change state. |
| Print lifecycle commands | Start/end print setup, power-loss restore, resume helpers. |
| Error/recovery commands | Retry or clear recorded box errors. |

## Safety notes

Many commands are not simple status reads. Some can move axes, heat the hotend,
change fan state, cut material, write config files, or remap tools.

General guidance:

- Treat material/tool-change commands as **motion and extrusion commands**.
- Treat `MODIFY_BOX_CFG` and `SAVE_BOX_CFG` as **persistent configuration-
  changes**.
- Treat `BOX_MODIFY_TN*` commands as **state overrides**.
- Avoid running material-change commands during resume unless the command is
  explicitly intended for resume.
- Use direct serial diagnostics first when validating hardware.

## Generated tool commands

The wrapper registers two sets of `T*` commands.

### Physical slot commands: `T1A`..`T4D`

Syntax:

```gcode
T1A
T1B
...
T4D
```

These request a material change to a specific physical box slot.

Example:

```gcode
T2C
```

means:

```text
Use physical box 2, slot C, after applying any Tnn remapping.
```

### Virtual tool commands: `T0`..`T15`

Syntax:

```gcode
T0
T1
...
T15
```

These map through the virtual tool table described in
[`state-model.md`](state-model.md#state-machine-5-virtual-tool-mapping).

Default mapping:

```text
T0 -> T1A
T1 -> T1B
...
T15 -> T4D
```

Both physical and virtual tool commands run the full material-change action.
That may include:

- saving the current tool;
- cutting current filament;
- retruding old material;
- loading new material from the box;
- extruding through the printer extruder;
- flushing/nozzle cleaning;
- Z moves and safe-position moves;
- error recording if a phase fails.

The detailed flow belongs in `material-change-flow.md`.

## Direct serial / low-level commands

These commands are the most direct bridge to the box serial protocol. They still
use normal response/error handling, but they do not intentionally run the full
material-change workflow.

For payload details and examples, see
[`serial-protocol.md`](serial-protocol.md#low-level-klipperg-code-macros).

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_CREATE_CONNECT` | `BOX_CREATE_CONNECT ADDR=<1..4>` | Send `CREATE_CONNECT`. Ignored while printing. |
| `BOX_GET_BOX_STATE` | `BOX_GET_BOX_STATE ADDR=<1..4>` | Query box state. |
| `BOX_GET_VERSION_SN` | `BOX_GET_VERSION_SN ADDR=<1..4>` | Query box version and serial number. |
| `BOX_GET_RFID` | `BOX_GET_RFID ADDR=<1..4> NUM=<0..15>` | Query RFID for selected slot mask. |
| `BOX_GET_REMAIN_LEN` | `BOX_GET_REMAIN_LEN ADDR=<1..4> NUM=<0..15>` | Query remain length for selected slot mask. |
| `BOX_GET_BUFFER_STATE` | `BOX_GET_BUFFER_STATE ADDR=<1..4>` | Query box buffer state. |
| `BOX_GET_FILAMENT_SENSOR_STATE` | `BOX_GET_FILAMENT_SENSOR_STATE ADDR=<1..4> POSITION=MATERIAL\|CONNECTIONS` | Query material or connection sensor mask. |
| `BOX_SET_BOX_MODE` | `BOX_SET_BOX_MODE ADDR=<1..4> MODE=PRINT\|IDLE [NUM=A\|B\|C\|D\|0]` | Set box mode for a slot or no slot. `NUM` defaults to `0`. |
| `BOX_CTRL_CONNECTION_MOTOR_ACTION` | `BOX_CTRL_CONNECTION_MOTOR_ACTION ADDR=<1..4> ACTION=STOP\|EXTRUDE\|RETRUDE` | Control connection motor action. |
| `BOX_MEASURING_WHEEL` | `BOX_MEASURING_WHEEL NUM=<1..4> ACTION=GET\|CLEAN` | Read or clear measuring wheel for box address `NUM`. |
| `BOX_TIGHTEN_UP_ENABLE` | `BOX_TIGHTEN_UP_ENABLE ADDR=<1..4> ENABLE=ENABLE\|DISABLE` | Enable/disable box tighten-up behavior. |
| `BOX_EXTRUDE_PROCESS` | `BOX_EXTRUDE_PROCESS ADDR=<1..9> NUM=A\|B\|C\|D [VELOCITY=<1..30>]` | Send legacy/debug `EXTRUDE_PROCESS` command. See caveat below. |
| `BOX_EXTRUDE_2_PROCESS` | `BOX_EXTRUDE_2_PROCESS ADDR=<1..4> NUM=A\|B\|C\|D TRIGGER=CONNECTION\|BUFFER` | Send `EXTRUDE_PROCESS_MODEL2`. |
| `BOX_RETRUDE_PROCESS` | `BOX_RETRUDE_PROCESS ADDR=<1..4> NUM=A\|B\|C\|D\|0 TRIGGER=BUFFER\|MATERIAL` | Send `RETRUDE_PROCESS`. |
| `BOX_COMMUNICATION_TEST` | `BOX_COMMUNICATION_TEST ADDR=<1..4> NUM=<0..255> [COUNT=<n>] [TIMEOUT=<s>] [INTERVAL=<s>]` | Repeated communication diagnostic. |
| `BOX_SEND_DATA` | `BOX_SEND_DATA ADDR=<1..4> NUM=<0..32> STATE=<0..255> TIMEOUT=<1..120> [DATA=<digits>]` | Raw-ish normal-frame sender. |

### `BOX_EXTRUDE_PROCESS` caveat

For the supported protocol versions documented here, `BOX_EXTRUDE_PROCESS` accepts a
`VELOCITY` parameter but the built serial payload uses the default stage value
rather than placing velocity directly into the frame. Treat this macro as a
legacy/debug command, not as a normal user extrusion command.

### `BOX_SEND_DATA` caveat

`BOX_SEND_DATA` is not a complete raw byte-frame sender. It still builds a normal
application-level request frame and computes the length byte automatically. Its
`DATA` parameter is parsed digit-by-digit, so `DATA=15` means bytes `01 05`, not
byte `0x0f`. See
[`serial-protocol.md`](serial-protocol.md#raw-ish-sender-box_send_data).

## State and mapping commands

| Command | Syntax | Purpose / caution |
|---|---|---|
| `BOX_MODIFY_TN` | `BOX_MODIFY_TN T1A=T2C` | Remap logical slot `T1A` to physical slot `T2C` and persist `Tnn_map`. Direct state override. |
| `BOX_MODIFY_TN_DATA` | `BOX_MODIFY_TN_DATA ADDR=<1..4> PART=<field> [NUM=A\|B\|C\|D] DATA=<value>` | Edit live material-box state, for example `PART=vender NUM=A DATA=unknown`. Can make state disagree with hardware. |
| `BOX_MODIFY_TN_INNER_DATA` | `BOX_MODIFY_TN_INNER_DATA ADDR=<1..4> PART=<field> [NUM=<value>] DATA=<value>` | Low-level state edit. Public macro shape is not useful for editing material/connection sensor subparts; prefer real sensor queries. |
| `BOX_SHOW_TNN_INNER_DATA` | `BOX_SHOW_TNN_INNER_DATA` | Print current material-box state. |
| `BOX_UPDATE_SAME_MATERIAL_LIST` | `BOX_UPDATE_SAME_MATERIAL_LIST` | Recompute same-material groups from current material identity. |
| `BOX_ENABLE_AUTO_REFILL` | `BOX_ENABLE_AUTO_REFILL ENABLE=<0\|1>` | Set runtime auto-refill flag. Intended values are `0`/`1`. Separate from `BOX_ENABLE_CFS_PRINT`. |
| `BOX_ADD_TNN` | `BOX_ADD_TNN TNN=<Tnn>` | Store a resume target material, for example `TNN=T1A`. |

Config-only custom toolchange macros sometimes wrap `BOX_MODIFY_TN` to maintain a
macro-side remap mirror. That mirror can track visible manual and auto-refill
remaps, but it cannot read `tn_data.json` or see mappings restored directly by
`BOX_POWER_LOSS_RESTORE`. See
[`state-model.md#macro-visibility-of-tnn_map`](state-model.md#macro-visibility-of-tnn_map).

## Configuration and calibration commands

These commands are documented in detail in
[`configuration.md`](configuration.md#runtime-configuration-commands).

| Command | Syntax | Purpose |
|---|---|---|
| `MODIFY_BOX_CFG` | `MODIFY_BOX_CFG KEY=value ...` | Change supported box config values in memory. |
| `SAVE_BOX_CFG` | `SAVE_BOX_CFG` | Persist pending config changes into `box.cfg`. |
| `BOX_SAVE_EXTRUDE_POS` | `BOX_SAVE_EXTRUDE_POS [X=<mm> Y=<mm>]` | Save current or supplied extrude XY position. |
| `MOVE_BOX_PRE_CUT_POS` | `MOVE_BOX_PRE_CUT_POS` | Move to configured pre-cut position. |
| `MOVE_BOX_CUT_POS` | `MOVE_BOX_CUT_POS` | Move to configured cut position. |
| `TEST_BOX_EXTRUDE` | `TEST_BOX_EXTRUDE` | Move to configured extrude XYZ test position. |
| `BOX_FIND_CUT_POS` | `BOX_FIND_CUT_POS` | Search for cut Y position with the cut sensor and save it. |

`BOX_FIND_CUT_POS` is more than a config write: it homes/moves the toolhead,
searches using the cut sensor, then calls `MODIFY_BOX_CFG CUT_POS_Y=...` and
`SAVE_BOX_CFG` when it finds a position.

## Sensor and hardware diagnostic commands

For hardware behavior details, see
[`sensors-and-hardware.md`](sensors-and-hardware.md).

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_CUT_STATE` | `BOX_CUT_STATE` | Report cut sensor state. |
| `BOX_GET_FIVE_WAY_STATE` | `BOX_GET_FIVE_WAY_STATE` | Query connection/five-way sensor state across connected boxes. |
| `BOX_CUT_HALL_ZERO` | `BOX_CUT_HALL_ZERO` | Cutter hall/zero routine. Registered only when `switch_pin` exists. |
| `BOX_CUT_HALL_TEST` | `BOX_CUT_HALL_TEST` | Cutter hall test. Registered only when `switch_pin` exists. |
| `BOX_SAVE_FAN` | `BOX_SAVE_FAN` | Save fan output values and turn configured fans off. |
| `BOX_RESTORE_FAN` | `BOX_RESTORE_FAN` | Restore saved fan output values. |
| `BOX_BLOW` | `BOX_BLOW` | Turn `fan0` on briefly, then off. |
| `BOX_GET_FLUSH_LEN` | `BOX_GET_FLUSH_LEN SCV=<RRGGBB> TCV=<RRGGBB>` | Compute/report flush length for source/target colors. |
| `BOX_GET_FLUSH_VELOCITY_TEST` | `BOX_GET_FLUSH_VELOCITY_TEST LAST=<Tnn> NEXT=<Tnn>` | Compute/report flush velocity profile for a material change. |
| `BOX_GENERATE_FLUSH_ARRAY` | `BOX_GENERATE_FLUSH_ARRAY FLUSH_ARRAY=<matrix>` | Parse/log a flush matrix string for diagnostics. |

### `BOX_GET_FLUSH_LEN`

Example:

```gcode
BOX_GET_FLUSH_LEN SCV=FF0000 TCV=00FF00
```

`SCV` and `TCV` must be six hex characters each. This computes the color-change
flush length using the configured nozzle volume and flush multiplier.

### `BOX_GENERATE_FLUSH_ARRAY`

The macro accepts an underscore-separated matrix string, for example:

```gcode
BOX_GENERATE_FLUSH_ARRAY FLUSH_ARRAY=1,2,3,4_5,6,7,8_9,10,11,12_13,14,15,16
```

It parses and logs the matrix; it is primarily diagnostic. Use a complete 4x4
integer matrix; malformed input can fail.

## Motion and cleaning commands

These commands move the printer or operate fans/cleaning hardware.

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_GO_TO_EXTRUDE_POS` | `BOX_GO_TO_EXTRUDE_POS` | Move to configured box extrude position. Ignored during resume. |
| `BOX_GO_TO_BOX_EXTRUDE_POS` | `BOX_GO_TO_BOX_EXTRUDE_POS` | Move to retrude/box-extrude position. |
| `BOX_MOVE_TO_SAFE_POS` | `BOX_MOVE_TO_SAFE_POS` | Move to configured safe position if XY is homed. |
| `RESTORE_POSITION` | `RESTORE_POSITION` | Run post-extrude restore behavior unless in resume. |
| `BOX_MOVE_TO_CUT` | `BOX_MOVE_TO_CUT` | Run cutter movement flow. |
| `BOX_CUT_MATERIAL` | `BOX_CUT_MATERIAL` | Close model fan and run cutter flow; queues macro error on failure. |
| `BOX_NOZZLE_CLEAN` | `BOX_NOZZLE_CLEAN` | Run nozzle-clean wipe/fan sequence. |
| `TEST_BOX_CLEAN` | `TEST_BOX_CLEAN` | Alias to nozzle clean. |
| `BOX_TN_EXTRUDE` | `BOX_TN_EXTRUDE` | Run configured default `Tn_Extrude`. |

These commands can home axes, move X/Y/Z, change acceleration limits, run fans,
and emit `M400` waits.

## Material and flush commands

These commands are higher-level and can involve heating, extrusion, box mode
changes, sensor checks, and state changes.

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_EXTRUDE_MATERIAL` | `BOX_EXTRUDE_MATERIAL TNN=<Tnn>` | Load/extrude material from a selected box slot. Always supply `TNN`. |
| `BOX_RETRUDE_MATERIAL` | `BOX_RETRUDE_MATERIAL` | Retrude/unload current material from the box path. |
| `BOX_RETRUDE_MATERIAL_WITH_TNN` | `BOX_RETRUDE_MATERIAL_WITH_TNN [TNN=<Tnn>]` | Retrude a specific material slot, or retrude unspecified material when no TNN is given. Not resume-guarded. |
| `BOX_EXTRUDER_EXTRUDE` | `BOX_EXTRUDER_EXTRUDE [TNN=<Tnn>]` | Run extruder-side extrusion bookkeeping/action for a slot. |
| `BOX_MATERIAL_FLUSH` | `BOX_MATERIAL_FLUSH [LEN=<mm>] [VELOCITY=<csv>] [TEMP=<c>] [PERCENT=<csv>]` | Direct material flush/extrusion and nozzle clean. |
| `BOX_MATERIAL_CHANGE_FLUSH` | `BOX_MATERIAL_CHANGE_FLUSH [LAST_TNN=<Tnn>] [TNN=<Tnn>]` | Flush for a material change from previous to current material. |
| `BOX_EXTRUSION_ALL_MATERIALS` | `BOX_EXTRUSION_ALL_MATERIALS` | Extrude remaining/ending material until sensor conditions are satisfied or limit is reached. |
| `BOX_CHECK_MATERIAL_REFILL` | `BOX_CHECK_MATERIAL_REFILL` | Check/refire refill handling after filament-error state. |
| `WAIT_EXTRUSION_ALL_MATERIALS` | `WAIT_EXTRUSION_ALL_MATERIALS` | Wait while auto-refill or extrusion-all-materials flows are running. |

### Resume behavior

Several material commands check whether the printer is in a resume flow. When in
resume, they log a warning and do nothing:

- `BOX_EXTRUDE_MATERIAL`
- `BOX_RETRUDE_MATERIAL`
- `BOX_EXTRUDER_EXTRUDE`
- `BOX_MATERIAL_FLUSH`
- `BOX_GO_TO_EXTRUDE_POS`
- `RESTORE_POSITION`
- `BOX_CUT_MATERIAL`

Use `BOX_RESUME_EXTRUDE` for the dedicated resume path.

Caveat: observed behavior for `BOX_RETRUDE_MATERIAL_WITH_TNN` is not
resume-guarded; it can move, extrude/retract, and send box retrude commands even
during resume handling.

### Composing lower-level material phases

A custom toolchange macro can avoid the wrapper's full hidden `T*` sequence by
calling lower-level phases explicitly, for example:

```text
pre-cut retract command
BOX_CUT_MATERIAL
BOX_RETRUDE_MATERIAL
BOX_EXTRUDE_MATERIAL TNN=<target>
BOX_EXTRUDER_EXTRUDE TNN=<target>
BOX_MATERIAL_CHANGE_FLUSH LAST_TNN=<old> TNN=<target>
```

This gives the macro control over where flush happens, but it also means the
macro owns sequencing and visible verification. These lower-level commands may
record/queue wrapper errors rather than raising a Klipper macro error at the
exact point of failure. Custom macros should add external checks such as local
filament-sensor assertions and loaded-slot verification before/after flush.

Direct macro error behavior to account for:

| Command | Failure/active-error behavior |
|---|---|
| `BOX_CUT_MATERIAL` | Records a macro cut error and queues the command line for macro-error retry. |
| `BOX_RETRUDE_MATERIAL` | Records a macro retrude error on its own failure and queues the command line. |
| `BOX_EXTRUDE_MATERIAL` | Queues when another error is active. Verify target loaded state manually after use. |
| `BOX_EXTRUDER_EXTRUDE` | Queues when another error is active; records a macro extruder error on its own failure. |
| `BOX_MATERIAL_CHANGE_FLUSH` | Queues when another error is active; records a macro flush error on its own failure. |

Queued command lines are replayed by `BOX_TNN_RETRY_PROCESS` only from a
`macro_*` saved-error branch. `BOX_ERROR_CLEAR` discards the queue.

### `BOX_MATERIAL_FLUSH` parameters

| Parameter | Meaning |
|---|---|
| `LEN` | Flush/extrude length. Defaults to configured `Tn_retrude`; max `200`. |
| `VELOCITY` | Comma-separated feedrate list, e.g. `2400,1200`. Defaults to configured `Tn_extrude_velocity` on parse failure. |
| `TEMP` | Hotend temperature. Values outside `180..300` fall back to configured `Tn_extrude_temp`. |
| `PERCENT` | Comma-separated percent list. Defaults to configured `Tn_extrude_percent` on parse failure. |

After extrusion, it runs nozzle cleaning and a small retract.

## Print lifecycle, preloading, and recovery commands

| Command | Syntax | Key behavior / side effects |
|---|---|---|
| `BOX_START_PRINT` | `BOX_START_PRINT` | Start-print setup for connected boxes; may close preloading and disables tighten-up. |
| `BOX_END_PRINT` | `BOX_END_PRINT` | End-print cleanup: may open preloading, enable tighten-up, reset mapping, clean resume fields, set box enable state to `0`, and disable `filament_sensor_2`. |
| `BOX_END` | `BOX_END` | Higher-level end action; may cut, retrude, or move safe depending on state. |
| `BOX_ENABLE_CFS_PRINT` | `BOX_ENABLE_CFS_PRINT ENABLE=<0\|1>` | Persist CFS/material-box print enable. `0` enables `filament_sensor_2`; `1` disables it. |
| `BOX_POWER_LOSS_RESTORE` | `BOX_POWER_LOSS_RESTORE` | Restore persisted enable state, virtual mapping, and last active material. |
| `BOX_RESUME_EXTRUDE` | `BOX_RESUME_EXTRUDE` | Dedicated resume extrusion path. Runs only when the resume target matches current material; can move Z/XY, heat, set box `PRINT`, toggle `fan0`, extrude, clean, retract, and restore speed/Z. |
| `DO_AFTER_PAUSE` | `DO_AFTER_PAUSE` | Wait for pause state and run the pause-after hook. |
| `BOX_SET_PRE_LOADING` | `BOX_SET_PRE_LOADING [ADDR=<0..4>] ACTION=CLOSE\|OPEN\|RUN\|TIGHT [NUM=<0..15>] [POWER_ON=ENABLE\|DISABLE]` | Open/close/run/tighten preloading. `ADDR=0` or omitted applies to connected boxes. `POWER_ON` stores startup behavior. May be skipped while printing/paused unless forced by a wrapper-managed path. |
| `BOX_TIGHTEN_UP_ENABLE` | `BOX_TIGHTEN_UP_ENABLE ADDR=<1..4> ENABLE=ENABLE\|DISABLE` or `BOX_TIGHTEN_UP_ENABLE NUM=<1..4> ENABLE=ENABLE\|DISABLE` | Enable/disable tighten-up behavior for one box. |
| `BOX_TNN_RETRY_PROCESS` | `BOX_TNN_RETRY_PROCESS` | Retry the currently recorded material-box error path; may move, heat, flush, and resume a paused print after success. |
| `BOX_ERROR_CLEAR` | `BOX_ERROR_CLEAR` | Clear the recorded error; may set an affected box/slot idle. Does not replay queued macro work. |
| `BOX_CUSTOM_COMMAND` | `BOX_CUSTOM_COMMAND CMD=<subcommand>` | Helper command. Supported `CMD`: `XYZ_ZERO`, `COORDINATES_ADJUST_PREPARE`, `COORDINATES_ADJUST_SAVE_POS`, `Y_SAFE`. |

Detailed recovery behavior belongs in [`errors-and-recovery.md`](errors-and-recovery.md).
Power-loss restore state is summarized in
[`state-model.md#state-machine-7-persisted-and-resume-state`](state-model.md#state-machine-7-persisted-and-resume-state).

## Known macro-interface caveats

| Area | Notes |
|---|---|
| Direct vs workflow commands | Some commands that sound direct, such as `BOX_CUT_MATERIAL`, run larger workflows and can move hardware. |
| Error queue semantics | Several direct material macros queue command lines instead of aborting immediately; see [`errors-and-recovery.md`](errors-and-recovery.md#commands-that-may-defer-failure-handling). |
| `BOX_ERROR_CLEAR` side effects | May set a box idle, clears the last alarm string, and discards queued macro replay. |
| `BOX_TNN_RETRY_PROCESS` side effects | May move axes, heat, set box modes, flush, restore acceleration, and resume a paused print after success. |
| `BOX_GET_BUFFER_STATE` | Treat command output as an immediate observation unless your wrapper owns a reliable cache; see [`compatibility-caveats.md`](compatibility-caveats.md#buffer-state-caution). |
| `BOX_EXTRUDE_PROCESS` | The accepted `VELOCITY` parameter is not clearly transmitted as velocity in supported protocol versions. |
| `BOX_SEND_DATA` | Raw-ish only; cannot send arbitrary bytes directly. |
| `BOX_MODIFY_TN_INNER_DATA` | The public macro shape does not pass the required sensor subpart, so it cannot update `filament_sensor.material` or `.connections` through the visible macro path. |
| Runtime config writes | `MODIFY_BOX_CFG` uses expression parsing and may skip zero/falsey values in the public save path; see [`configuration.md`](configuration.md#known-configuration-caveats). |
| Resume guard | Several material/motion commands silently no-op with warnings during resume. |
| `BOX_RETRUDE_MATERIAL_WITH_TNN` | Observed behavior is not resume-guarded. |
| `BOX_EXTRUDE_MATERIAL` failure path | Verify on target firmware before relying on queued recovery behavior. |
| More caveats | See [`compatibility-caveats.md`](compatibility-caveats.md). |
