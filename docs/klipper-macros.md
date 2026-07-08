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
- Treat `MODIFY_BOX_CFG` and `SAVE_BOX_CFG` as **persistent configuration
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

### `BOX_MODIFY_TN`

Changes the mutable virtual/physical slot mapping.

Syntax:

```gcode
BOX_MODIFY_TN T1A=T2C
```

Meaning:

```text
logical slot T1A now points to actual physical slot T2C
```

This updates and persists the wrapper's `Tnn_map`. See
[`state-model.md`](state-model.md#state-machine-5-virtual-tool-mapping).

Caution: this is a direct state override. Use it only when you understand the
current material mapping.

Config-only custom toolchange macros sometimes wrap `BOX_MODIFY_TN` to maintain a
macro-side mirror of remaps. That mirror can track normal manual remaps and
normal auto-refill remaps that pass through `BOX_MODIFY_TN`, but it cannot read
`tn_data.json` or see mappings restored directly by `BOX_POWER_LOSS_RESTORE`.
See [`state-model.md`](state-model.md#macro-visibility-of-tnn_map).

### `BOX_MODIFY_TN_DATA`

Directly edits live material-box state.

Syntax:

```gcode
BOX_MODIFY_TN_DATA ADDR=<1..4> PART=<field> [NUM=A|B|C|D] DATA=<value>
```

Examples:

```gcode
BOX_MODIFY_TN_DATA ADDR=1 PART=vender NUM=A DATA=unknown
BOX_MODIFY_TN_DATA ADDR=1 PART=state DATA=connect
```

Use cases:

- manual correction during testing;
- forcing a state for diagnostics;
- recovering from inconsistent status.

Caution: this can make wrapper state disagree with physical box state.

### `BOX_MODIFY_TN_INNER_DATA`

Attempts to directly edit inner state.

Syntax:

```gcode
BOX_MODIFY_TN_INNER_DATA ADDR=<1..4> PART=<field> [NUM=<value>] DATA=<value>
```

Known caveat: the public macro shape does not clearly pass a sensor subpart such as
`material` or `connections`, so it may not be useful for editing filament sensor
inner state. Prefer real sensor queries:

```gcode
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=CONNECTIONS
```

### `BOX_SHOW_TNN_INNER_DATA`

Prints the current material-box state object. Color values may be converted to
known display color names when recognized.

Syntax:

```gcode
BOX_SHOW_TNN_INNER_DATA
```

### `BOX_UPDATE_SAME_MATERIAL_LIST`

Recomputes compatible-material groups from current material identity state.

Syntax:

```gcode
BOX_UPDATE_SAME_MATERIAL_LIST
```

See [`state-model.md`](state-model.md#derived-same-material-groups).

### `BOX_ENABLE_AUTO_REFILL`

Sets the runtime auto-refill flag.

Syntax:

```gcode
BOX_ENABLE_AUTO_REFILL ENABLE=<0|1>
```

Intended values are `0` and `1`; other values should be avoided. This affects runtime state. It is separate from the CFS-print
enable flag controlled by `BOX_ENABLE_CFS_PRINT`.

### `BOX_ADD_TNN`

Stores a resume target material on the toolhead state.

Syntax:

```gcode
BOX_ADD_TNN TNN=<Tnn>
```

Example:

```gcode
BOX_ADD_TNN TNN=T1A
```

Used by resume-related paths.

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

## Print lifecycle commands

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_START_PRINT` | `BOX_START_PRINT` | Start-print setup for connected boxes. |
| `BOX_END_PRINT` | `BOX_END_PRINT` | End-print cleanup for connected boxes. |
| `BOX_END` | `BOX_END` | Higher-level end action; may cut/retrude/move safe depending on state. |
| `BOX_ENABLE_CFS_PRINT` | `BOX_ENABLE_CFS_PRINT ENABLE=<0|1>` | Enable/disable CFS/material-box printing state and persist it. |
| `BOX_POWER_LOSS_RESTORE` | `BOX_POWER_LOSS_RESTORE` | Restore selected persisted state after power loss. |
| `BOX_RESUME_EXTRUDE` | `BOX_RESUME_EXTRUDE` | Dedicated resume extrusion path. |
| `DO_AFTER_PAUSE` | `DO_AFTER_PAUSE` | Wait for pause state and then run `WAIT_PAUSE`. |

### `BOX_ENABLE_CFS_PRINT`

Syntax:

```gcode
BOX_ENABLE_CFS_PRINT ENABLE=<0|1>
```

Behavior:

| `ENABLE` | Meaning | Local filament sensor command |
|---:|---|---|
| `0` | Disable box/CFS printing. | Enable `filament_sensor_2`. |
| `1` | Enable box/CFS printing. | Disable `filament_sensor_2`. |

This value is persisted for power-loss restore.

### `BOX_START_PRINT`

For each connected box:

- optionally closes preloading if preloading-on-power is enabled;
- disables tighten-up behavior.

### `BOX_END_PRINT`

For each connected box:

- opens preloading if preloading is enabled;
- enables tighten-up behavior;
- resets virtual slot mapping to identity;
- cleans power-loss resume fields;
- sets box enable state to `0`;
- disables `filament_sensor_2`.

### `BOX_POWER_LOSS_RESTORE`

Restores persisted:

- enable state;
- virtual slot mapping;
- last active material.

See [`state-model.md`](state-model.md#state-machine-7-persisted-and-resume-state).

### `BOX_RESUME_EXTRUDE`

`BOX_RESUME_EXTRUDE` only proceeds when it can find a resume target and that
resolved target matches the current `last_cmd` physical material content. When it
runs, it has substantial side effects: it raises Z, moves to the extrude
position, sets the box to `PRINT`, heats, temporarily turns `fan0` off, extrudes,
cleans the nozzle, moves safe, retracts slightly, restores Z, and restores the
saved G-code speed.

## Preloading and tighten-up commands

### `BOX_SET_PRE_LOADING`

Syntax:

```gcode
BOX_SET_PRE_LOADING [ADDR=<0..4>] ACTION=CLOSE|OPEN|RUN|TIGHT [NUM=<0..15>] [POWER_ON=ENABLE|DISABLE]
```

Behavior:

| Parameter | Meaning |
|---|---|
| `ADDR` | Box address. `0` or omitted means apply to all connected boxes. |
| `ACTION` | `CLOSE`, `OPEN`, `RUN`, or `TIGHT`. |
| `NUM` | Slot mask, default `15` / all slots. |
| `POWER_ON` | Optional persistent runtime flag: `ENABLE` or `DISABLE`. |

Side effects:

- `ACTION=CLOSE` marks preloading disabled.
- `ACTION=OPEN` marks preloading enabled.
- `POWER_ON=ENABLE|DISABLE` sets whether preloading should run on power-on/start.

The underlying serial command may be skipped while printing or paused unless the
the wrapper uses a forced path.

### `BOX_TIGHTEN_UP_ENABLE`

Syntax:

```gcode
BOX_TIGHTEN_UP_ENABLE ADDR=<1..4> ENABLE=ENABLE|DISABLE
```

Alternative address parameter:

```gcode
BOX_TIGHTEN_UP_ENABLE NUM=<1..4> ENABLE=ENABLE|DISABLE
```

Enables or disables tighten-up behavior for one box.

## Error and recovery commands

Detailed error behavior belongs in `errors-and-recovery.md`.

| Command | Syntax | Purpose |
|---|---|---|
| `BOX_TNN_RETRY_PROCESS` | `BOX_TNN_RETRY_PROCESS` | Retry the currently recorded material-box error path. |
| `BOX_ERROR_CLEAR` | `BOX_ERROR_CLEAR` | Clear or partially resolve the currently recorded error. |

High-level behavior:

```text
BOX_TNN_RETRY_PROCESS:
    inspect current recorded error
    choose retry path based on error type
    run recovery material/cut/flush path
    resume print if recovery succeeded and print is paused

BOX_ERROR_CLEAR:
    inspect current recorded error
    set affected box/slot idle when needed
    clear recorded error state
```

Some ordinary workflow macros queue themselves when an error is already recorded.
That queued macro list is replayed by macro-error recovery.

## Custom command helper

### `BOX_CUSTOM_COMMAND`

Syntax:

```gcode
BOX_CUSTOM_COMMAND CMD=<subcommand>
```

Supported subcommands:

| Subcommand | Behavior |
|---|---|
| `XYZ_ZERO` | Runs `G28` and records result `XYZ_ZERO=0`. |
| `COORDINATES_ADJUST_PREPARE` | Moves Z/Y/X to a coordinate-adjustment preparation position. |
| `COORDINATES_ADJUST_SAVE_POS` | Saves current toolhead X/Y as extrude position using config commands. |
| `Y_SAFE` | Moves to configured safe Y. |

The last result string is exposed in wrapper status as `custom_command_result`.

## Command quick reference by risk level

For a table of every public named command and its primary
side-effect class, see
[`runtime-reference.md#public-g-code-command-inventory`](runtime-reference.md#public-g-code-command-inventory).

### Mostly read-only diagnostics

```text
BOX_GET_BOX_STATE
BOX_GET_VERSION_SN
BOX_GET_RFID
BOX_GET_REMAIN_LEN
BOX_GET_BUFFER_STATE
BOX_GET_FILAMENT_SENSOR_STATE
BOX_GET_FIVE_WAY_STATE
BOX_MEASURING_WHEEL ACTION=GET
BOX_CUT_STATE
BOX_GET_FLUSH_LEN
BOX_GET_FLUSH_VELOCITY_TEST
BOX_SHOW_TNN_INNER_DATA
```

### Direct state/config/box-mode mutation

```text
BOX_MODIFY_TN
BOX_MODIFY_TN_DATA
BOX_MODIFY_TN_INNER_DATA
BOX_ENABLE_AUTO_REFILL
BOX_ENABLE_CFS_PRINT
BOX_SET_BOX_MODE
BOX_SET_PRE_LOADING
BOX_TIGHTEN_UP_ENABLE
MODIFY_BOX_CFG
SAVE_BOX_CFG
BOX_SAVE_EXTRUDE_POS
BOX_POWER_LOSS_RESTORE
```

### Direct serial or motor actions

```text
BOX_SEND_DATA
BOX_CREATE_CONNECT
BOX_CTRL_CONNECTION_MOTOR_ACTION
BOX_EXTRUDE_PROCESS
BOX_EXTRUDE_2_PROCESS
BOX_RETRUDE_PROCESS
BOX_COMMUNICATION_TEST
```

### Motion/fan/hardware actions

```text
MOVE_BOX_PRE_CUT_POS
MOVE_BOX_CUT_POS
TEST_BOX_EXTRUDE
BOX_FIND_CUT_POS
BOX_GO_TO_EXTRUDE_POS
BOX_GO_TO_BOX_EXTRUDE_POS
BOX_MOVE_TO_SAFE_POS
BOX_MOVE_TO_CUT
BOX_CUT_MATERIAL
BOX_NOZZLE_CLEAN
TEST_BOX_CLEAN
BOX_BLOW
BOX_SAVE_FAN
BOX_RESTORE_FAN
BOX_CUSTOM_COMMAND
```

### Material workflow actions

```text
T0..T15
T1A..T4D
BOX_EXTRUDE_MATERIAL
BOX_RETRUDE_MATERIAL
BOX_RETRUDE_MATERIAL_WITH_TNN
BOX_EXTRUDER_EXTRUDE
BOX_MATERIAL_FLUSH
BOX_MATERIAL_CHANGE_FLUSH
BOX_EXTRUSION_ALL_MATERIALS
BOX_CHECK_MATERIAL_REFILL
BOX_RESUME_EXTRUDE
```

### Error/recovery/lifecycle

```text
BOX_TNN_RETRY_PROCESS
BOX_ERROR_CLEAR
BOX_START_PRINT
BOX_END_PRINT
BOX_END
DO_AFTER_PAUSE
RESTORE_POSITION
WAIT_EXTRUSION_ALL_MATERIALS
```

## Known macro-interface caveats

| Area | Notes |
|---|---|
| Direct vs workflow commands | Some commands that sound direct, such as `BOX_CUT_MATERIAL`, run larger workflows and can move hardware. |
| Error queue semantics | Several direct material macros queue command lines instead of aborting immediately; see [`errors-and-recovery.md`](errors-and-recovery.md#commands-that-record-or-queue-without-aborting). |
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
