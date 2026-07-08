# Box configuration reference

## Scope

This document explains how to read and maintain the material-box `box.cfg`
configuration. It is written for readers who have access to the deployed
`box.cfg` artifact, but not necessarily to the source code.

It covers:

- the `[box]` section options;
- practical setup/calibration order;
- option defaults, bounds, and units;
- runtime config commands that update `box.cfg`;
- common caveats when editing or saving configuration.

It does not describe the serial frame protocol; see
[`serial-protocol.md`](serial-protocol.md). It also does not describe the full
material-change or recovery workflows. For hard-coded runtime names/files and a
command side-effect matrix, see [`runtime-reference.md`](runtime-reference.md).

## Where the config lives

The runtime save command searches for `box.cfg` in this order:

1. `/usr/data/printer_data/config/box.cfg`
2. `/mnt/UDISK/printer_data/config/box.cfg`

`SAVE_BOX_CFG` creates a backup next to the selected file:

```text
box.cfg.1
```

Other runtime files are also fixed relative to the Klipper/base
directory, including persisted box state, the material database, and the
one-shot flushing sign. See
[`runtime-reference.md#hard-coded-runtime-names-and-files`](runtime-reference.md#hard-coded-runtime-names-and-files).

## Minimal example `[box]` section

This is an illustrative starting point, not a verified hardware profile. Use the
coordinates for your printer.

```ini
[box]
bus: serial485
has_extrude_pos: 1
version: 0
filament_sensor: filament_sensor_2
switch_pin: ^your_cut_sensor_pin

pre_cut_pos_x: 0
pre_cut_pos_y: 0
cut_pos_x: -6.5
cut_pos_y: 0
cut_pos_offset: 0
cut_velocity: 6000
cut_run_count: 1

safe_pos_y: 350
extrude_pos_x: 127
extrude_pos_y: 368
retrude_position_x: 42
retrude_position_y: 100
clean_velocity: 6000

Tn_extrude_temp: 220
Tn_extrude: 140
Tn_extrude_percent: 100
Tn_extrude_velocity: 2400
Tn_retrude: -60
Tn_retrude_velocity: 2400

buffer_empty_len: 30
diff_length: -10
```

## Safe setup and calibration order

Recommended order for a new or unknown machine:

1. **Configure the serial bus**
   - Set `bus` to match your `[serial_485 ...]` object name.
   - Confirm low-level communication with direct box commands documented in
     [`serial-protocol.md`](serial-protocol.md#low-level-klipperg-code-macros).

2. **Enable the full position model**
   - Set `has_extrude_pos: 1` unless you know your profile does not need the
     extrude/clean/safe/retrude positions.

3. **Configure sensors**
   - Set `filament_sensor` if the printer has a local filament switch sensor
     managed by Klipper.
   - Set `switch_pin` if a cutter/cut-return sensor is installed.

4. **Calibrate safe and extrude positions**
   - Move the toolhead to the desired extrude position.
   - Use `BOX_SAVE_EXTRUDE_POS` to write `extrude_pos_x` and `extrude_pos_y`.
   - Verify `safe_pos_y` keeps the toolhead clear of box/cutter hardware.

5. **Calibrate cutter positions**
   - Use `MOVE_BOX_PRE_CUT_POS` to test the pre-cut position.
   - Use `MOVE_BOX_CUT_POS` to test the computed cut position.
   - Adjust `cut_pos_y` and `cut_pos_offset` as needed.

6. **Tune material flow**
   - Start with conservative defaults for temperature, extrusion, retrude, and
     flush settings.
   - Tune `Tn_extrude`, `Tn_extrude_velocity`, and clean lengths only after
     basic loading/retruding works.

7. **Persist changes carefully**
   - `MODIFY_BOX_CFG` changes runtime values.
   - `SAVE_BOX_CFG` writes pending runtime values into `box.cfg`.
   - Restart/reload the printer stack if the surrounding firmware requires it.

## Units and conventions

| Setting type | Unit / convention |
|---|---|
| X/Y/Z positions | millimeters |
| Extrusion/retrusion lengths | millimeters of filament unless noted otherwise |
| G-code feedrates / velocities | mm/min |
| Acceleration settings | printer acceleration units, usually mm/s² |
| Temperatures | °C |
| Slot selectors | A/B/C/D or 4-bit masks depending on command/context |

## Required connection options

| Option | Default | Bounds | Meaning |
|---|---:|---|---|
| `bus` | required | n/a | Name of the `serial_485` bus object. Example: `[serial_485 serial485]` pairs with `bus: serial485`. |
| `version` | `0` | `0..1` | Protocol/config version. Normal profiles use `0` or `1`. |

If `bus` is missing, the wrapper cannot initialize the material box interface.

## Feature gates and sensors

| Option | Default | Bounds | Meaning |
|---|---:|---|---|
| `has_extrude_pos` | `0` | `min=0` | Set to `1` to enable the full extrude/clean/safe/retrude position set. Any value other than `1` behaves as disabled. |
| `filament_sensor` | unset | n/a | Optional local Klipper filament switch sensor name, for example `filament_sensor_2`. |
| `switch_pin` | unset | n/a | Optional cutter/cut-return switch input. If missing, cutter logic uses blind cut strokes instead of sensor-confirmed cutting. |

> Practical recommendation: use `has_extrude_pos: 1` for normal material-box
> deployments. With `has_extrude_pos: 0`, Z/safe/retrude fields are not
> initialized; API output and commands such as `TEST_BOX_EXTRUDE` may fail or
> return incomplete results.

## Cutter and pre-cut position options

These control the cutter approach and cut location.

| Option | Default | Bounds | Unit | Meaning |
|---|---:|---|---|---|
| `pre_cut_pos_x` | `0` | X min..max | mm | X position before the cut stroke. |
| `pre_cut_pos_y` | `0` | Y min..max | mm | Y position before the cut stroke. |
| `cut_pos_x` | `-6.5` | X min..max | mm | X position for cutting. |
| `cut_pos_y` | `0` | Y min..max | mm | Base Y cut position. |
| `cut_pos_offset` | `0` | `-1.0..5.0` | mm | Offset added to `cut_pos_y`. |
| `cut_find_start_offset_y` | `6` | `0..20.0` | mm | Offset from Y maximum used by automatic cut-position search. |
| `cut_velocity` | `6000` | `1200..printer.max_velocity*60` | mm/min | Feedrate for cutter moves. |
| `cut_accel` | `5000` | `1000..10000` | mm/s² | Temporary acceleration limit while cutting. |
| `cut_accel_to_decel` | `30000` | `10000..50000` | mm/s² | Temporary accel-to-decel limit while cutting. |
| `cut_run_count` | `0` | none visible | count | Number of cut stroke repetitions. If no `switch_pin` is configured, the blind-cut motion uses this value; the default `0` means zero blind cut strokes while still returning success. |

The effective cut Y is:

```text
real_cut_pos_y = cut_pos_y + cut_pos_offset
```

but it is capped to:

```text
stepper_y.position_max - 0.1
```

## Extrude, clean, safe, and retrude positions

These options are used when `has_extrude_pos: 1`.

| Option | Default | Bounds | Unit | Meaning |
|---|---:|---|---|---|
| `safe_pos_y` | `350` | Y min..max | mm | Safe Y position used before/after many box operations. |
| `extrude_pos_x` | `127` | X min..max | mm | X position used for flushing/extruding near the box. |
| `extrude_pos_y` | `368` | Y min..max | mm | Y position used for flushing/extruding near the box. |
| `retrude_position_x` | `42` | X min..max | mm | X position used by retrude-related moves. |
| `retrude_position_y` | `100` | Y min..max | mm | Y position used by retrude-related moves. |
| `clean_left_pos_x` | `150` | X min..max | mm | Left cleaning X position. |
| `clean_left_pos_y` | `368` | Y min..max | mm | Left cleaning Y position. |
| `clean_right_pos_x` | `170` | X min..max | mm | Right cleaning / safe X position. |
| `clean_right_pos_y` | `368` | Y min..max | mm | Right cleaning Y position. |
| `clean_velocity` | `6000` | `1200..printer.max_velocity*60` | mm/min | Feedrate for cleaning and extrude-position moves. |
| `clean_pos_min_x` | `152` | X min..max | mm | Minimum X of the configured cleaning region. |
| `clean_pos_min_y` | `362` | Y min..max | mm | Minimum Y of the configured cleaning region. |
| `clean_pos_max_x` | `156` | X min..max | mm | Maximum X of the configured cleaning region. |
| `clean_pos_max_y` | `368` | Y min..max | mm | Maximum Y of the configured cleaning region. |

### Z position options

When `has_extrude_pos: 1`, Z defaults are based on:

```text
default_z = stepper_z.position_min + 25
```

| Option | Default | Bounds | Unit | Meaning |
|---|---:|---|---|---|
| `extrude_pos_z` | `default_z` | Z min..max | mm | Z position used by test/extrude-position commands. |
| `clean_left_pos_z` | `default_z` | Z min..max | mm | Left cleaning Z position. |
| `clean_right_pos_z` | `default_z` | Z min..max | mm | Right cleaning Z position. |

## Material flow and flush options

These options control material extrusion, retrusion, flushing, and temperature.

| Option | Default | Bounds | Unit / shape | Meaning |
|---|---:|---|---|---|
| `Tn_retrude` | `-60` | `-120..-1` | mm filament | Default retrude length. Negative means retract. |
| `Tn_retrude_velocity` | `[2400]` | list | mm/min | Feedrate list for retrude-style moves. |
| `Tn_extrude_temp` | `220` | `180..300` | °C | Default hotend temperature for box extrusion/flush. |
| `Tn_extrude` | `140` | `0..200` | mm filament | Default extrusion/flush length. |
| `Tn_extrude_percent` | `[100]` | list | percent | Segment percentages for extrusion. Must match `Tn_extrude_velocity` length. |
| `Tn_extrude_velocity` | `[2400]` | list | mm/min | Segment feedrates for extrusion. Must match `Tn_extrude_percent` length. |
| `nozzle_volume` | `200` | `0..300` | volume unit | Added to color-change flush volume before converting to length. |
| `flush_multiplier` | `1` | `0..10` | multiplier | Multiplies computed flush length. |
| `buffer_empty_len` | `30` | `0..60` | mm filament | Threshold used in buffer-empty / measurement decisions. |
| `diff_length` | `-10` | `-30..30` | measuring-wheel delta | Threshold used when comparing extrusion to measuring-wheel motion. |

`Tn_extrude_percent` and `Tn_extrude_velocity` must have the same number of
entries. A mismatch prevents configuration from loading.

## Cleaning segment options

These tune how a long flush is split into segments.

| Option | Default | Bounds | Unit | Meaning |
|---|---:|---|---|---|
| `box_first_clean_length` | `90` | `0..600` | mm filament | Initial/minimum flush segment length. |
| `box_need_clean_length` | `80` | `0..box_first_clean_length` | mm filament | Normal repeated segment length. |
| `box_need_clean_length_max` | `100` | `0..600` | mm filament | Maximum/cap used when calculating segmented flushes. |
| `clean_run_count` | `0` | none visible | count | Stored/exposed cleaning repetition count. |

## Material loading process options

These tune the box-to-extruder loading sequence.

| Option | Default | Bounds | Unit | Meaning |
|---|---:|---|---|---|
| `box_extrude_retry_num` | `5` | `0..5` | count | Configured retry count. Some workflows use fixed retry policies, so practical effect may be limited. |
| `extrude_material_len_for_box` | `3` | `0..60` | protocol amount / mm-like | Amount sent to the box during final loading stage. |
| `extrude_material_len_for_extruder` | `18` | `0..60` | mm filament | Extruder-side extrusion amount during final loading stage. |
| `extrude_material_velocity` | `180` | none visible | mm/min | Feedrate for `extrude_material_len_for_extruder`. |
| `extrude_material_times` | `6` | none visible | count | Number of final loading attempts. |

## Miscellaneous options

| Option | Default | Bounds | Meaning |
|---|---:|---|---|
| `Tn_line_len` | `0` | none visible | Stored/exposed line-length value. Runtime effect is unclear from behavior. |
| `cr_machine` | `0` | none visible | Stored machine flag/id. Runtime effect is unclear from behavior. |

## Runtime configuration commands

These commands are useful when calibrating without hand-editing `box.cfg`.

### `MODIFY_BOX_CFG`

Changes selected configuration values in memory and records them for later save.

Example:

```gcode
MODIFY_BOX_CFG EXTRUDE_POS_X=127.50 EXTRUDE_POS_Y=368.00
```

Supported keys:

| Macro key | `box.cfg` option |
|---|---|
| `PRE_CUT_POS_X` | `pre_cut_pos_x` |
| `PRE_CUT_POS_Y` | `pre_cut_pos_y` |
| `CUT_POS_X` | `cut_pos_x` |
| `CUT_POS_Y` | `cut_pos_y` |
| `CUT_VELOCITY` | `cut_velocity` |
| `TN_RETRUDE` | `Tn_retrude` |
| `TN_RETRUDE_VELOCITY` | `Tn_retrude_velocity` |
| `BUFFER_EMPTY_LEN` | `buffer_empty_len` |
| `CLEAN_LEFT_POS_X` | `clean_left_pos_x` |
| `CLEAN_LEFT_POS_Y` | `clean_left_pos_y` |
| `CLEAN_RIGHT_POS_X` | `clean_right_pos_x` |
| `CLEAN_RIGHT_POS_Y` | `clean_right_pos_y` |
| `CLEAN_LEFT_POS_Z` | `clean_left_pos_z` |
| `CLEAN_RIGHT_POS_Z` | `clean_right_pos_z` |
| `CLEAN_VELOCITY` | `clean_velocity` |
| `EXTRUDE_POS_X` | `extrude_pos_x` |
| `EXTRUDE_POS_Y` | `extrude_pos_y` |
| `EXTRUDE_POS_Z` | `extrude_pos_z` |
| `TN_LINE_LEN` | `Tn_line_len` |
| `CLEAN_RUN_COUNT` | `clean_run_count` |
| `CUT_RUN_COUNT` | `cut_run_count` |

Caveats:

- `MODIFY_BOX_CFG` changes runtime state first; it does not write the file until
  `SAVE_BOX_CFG` is run.
- Values are parsed as expressions by the wrapper, not as a strict numeric-only
  input parser.
- Falsey values such as `0` may not be recorded/saved by this path. Edit
  `box.cfg` manually for zero-valued settings if needed.
- Runtime modification does not apply the original config bounds checks.

### `SAVE_BOX_CFG`

Writes pending `MODIFY_BOX_CFG` values into `box.cfg`.

Behavior:

1. picks the first existing config path from the known paths;
2. backs it up to `box.cfg.1`;
3. rewrites existing unindented matching keys;
4. appends missing pending keys only when `[box]` is followed by another section;
5. runs `sync`.

If `[box]` is the last section and a pending key is absent, the public save
path may not append that missing key. Existing keys are matched by simple line
prefix, so indented keys or unusual formatting may not be rewritten.

Example:

```gcode
MODIFY_BOX_CFG CUT_POS_Y=360.20
SAVE_BOX_CFG
```

### `BOX_SAVE_EXTRUDE_POS`

Convenience command for saving the current or supplied extrude XY position.

With explicit coordinates:

```gcode
BOX_SAVE_EXTRUDE_POS X=127.50 Y=368.00
```

Without coordinates, it reads the current toolhead X/Y position.

It emits these public config commands:

```gcode
MODIFY_BOX_CFG EXTRUDE_POS_X=<x>
MODIFY_BOX_CFG EXTRUDE_POS_Y=<y>
SAVE_BOX_CFG
```

### `MOVE_BOX_PRE_CUT_POS`

Moves to:

```text
X = pre_cut_pos_x
Y = pre_cut_pos_y
F = cut_velocity
```

Use it to test the pre-cut position after editing `box.cfg`.

### `MOVE_BOX_CUT_POS`

Moves to:

```text
X = cut_pos_x
Y = min(cut_pos_y + cut_pos_offset, stepper_y.position_max - 0.1)
F = cut_velocity
```

Use it to test the actual cutter position after editing `box.cfg`.

### `TEST_BOX_EXTRUDE`

Moves to:

```text
X = extrude_pos_x
Y = extrude_pos_y
Z = extrude_pos_z
```

Use it to test the configured extrude position.

## Web/API config endpoint

The wrapper exposes a web endpoint:

```text
box/get_box_cfg
```

It returns selected config values with min/max metadata for UI use. The endpoint
includes positions, cut velocity, retrude settings, clean/extrude positions,
`buffer_empty_len`, `Tn_line_len`, and run counts. The web response exposes
`cut_run_count` under the surprising key `cut_succeed_num`.

Known mismatch: this endpoint reports `buffer_empty_len` max as `30`, while the
config loader accepts up to `60`.

## Common tuning symptoms

| Symptom | Likely settings to inspect |
|---|---|
| Box cannot communicate | `bus`, serial_485 object name, physical serial wiring. |
| Toolhead moves to unsafe Y during box operation | `safe_pos_y`, `extrude_pos_y`, `retrude_position_y`. |
| Cutter misses or overtravels | `pre_cut_pos_x`, `pre_cut_pos_y`, `cut_pos_x`, `cut_pos_y`, `cut_pos_offset`, `cut_run_count`. |
| Cut search starts in the wrong area | `cut_find_start_offset_y`, Y axis max. |
| Flush length too short/long | `Tn_extrude`, `Tn_extrude_percent`, `box_first_clean_length`, `box_need_clean_length`, `flush_multiplier`, `nozzle_volume`. |
| Material load does not reach buffer/extruder reliably | `extrude_material_len_for_box`, `extrude_material_len_for_extruder`, `extrude_material_velocity`, `extrude_material_times`. |
| Retraction pulls too far/not enough | `Tn_retrude`, `Tn_retrude_velocity`. |
| Temperature too low/high during box operations | `Tn_extrude_temp`, material database temperature if present. |

## Alphabetical option index

| Option | Section |
|---|---|
| `box_extrude_retry_num` | Material loading process options |
| `box_first_clean_length` | Cleaning segment options |
| `box_need_clean_length` | Cleaning segment options |
| `box_need_clean_length_max` | Cleaning segment options |
| `buffer_empty_len` | Material flow and flush options |
| `bus` | Required connection options |
| `clean_left_pos_x` | Extrude, clean, safe, and retrude positions |
| `clean_left_pos_y` | Extrude, clean, safe, and retrude positions |
| `clean_left_pos_z` | Z position options |
| `clean_pos_max_x` | Extrude, clean, safe, and retrude positions |
| `clean_pos_max_y` | Extrude, clean, safe, and retrude positions |
| `clean_pos_min_x` | Extrude, clean, safe, and retrude positions |
| `clean_pos_min_y` | Extrude, clean, safe, and retrude positions |
| `clean_right_pos_x` | Extrude, clean, safe, and retrude positions |
| `clean_right_pos_y` | Extrude, clean, safe, and retrude positions |
| `clean_right_pos_z` | Z position options |
| `clean_run_count` | Cleaning segment options |
| `clean_velocity` | Extrude, clean, safe, and retrude positions |
| `cr_machine` | Miscellaneous options |
| `cut_accel` | Cutter and pre-cut position options |
| `cut_accel_to_decel` | Cutter and pre-cut position options |
| `cut_find_start_offset_y` | Cutter and pre-cut position options |
| `cut_pos_offset` | Cutter and pre-cut position options |
| `cut_pos_x` | Cutter and pre-cut position options |
| `cut_pos_y` | Cutter and pre-cut position options |
| `cut_run_count` | Cutter and pre-cut position options |
| `cut_velocity` | Cutter and pre-cut position options |
| `diff_length` | Material flow and flush options |
| `extrude_material_len_for_box` | Material loading process options |
| `extrude_material_len_for_extruder` | Material loading process options |
| `extrude_material_times` | Material loading process options |
| `extrude_material_velocity` | Material loading process options |
| `extrude_pos_x` | Extrude, clean, safe, and retrude positions |
| `extrude_pos_y` | Extrude, clean, safe, and retrude positions |
| `extrude_pos_z` | Z position options |
| `filament_sensor` | Feature gates and sensors |
| `flush_multiplier` | Material flow and flush options |
| `has_extrude_pos` | Feature gates and sensors |
| `nozzle_volume` | Material flow and flush options |
| `pre_cut_pos_x` | Cutter and pre-cut position options |
| `pre_cut_pos_y` | Cutter and pre-cut position options |
| `retrude_position_x` | Extrude, clean, safe, and retrude positions |
| `retrude_position_y` | Extrude, clean, safe, and retrude positions |
| `safe_pos_y` | Extrude, clean, safe, and retrude positions |
| `switch_pin` | Feature gates and sensors |
| `Tn_extrude` | Material flow and flush options |
| `Tn_extrude_percent` | Material flow and flush options |
| `Tn_extrude_temp` | Material flow and flush options |
| `Tn_extrude_velocity` | Material flow and flush options |
| `Tn_line_len` | Miscellaneous options |
| `Tn_retrude` | Material flow and flush options |
| `Tn_retrude_velocity` | Material flow and flush options |
| `version` | Required connection options |

## Known configuration caveats

| Area | Notes |
|---|---|
| `has_extrude_pos = 0` | Many normal material-box flows expect fields that are only loaded when this is `1`; commands/API output can fail or be incomplete with `0`. |
| Runtime save of zero values | The public `MODIFY_BOX_CFG`/`SAVE_BOX_CFG` path may skip falsey values such as `0`; hand-edit the file for those cases. |
| Expression parsing | `MODIFY_BOX_CFG` evaluates values as expressions. Treat it as powerful and potentially unsafe. |
| Runtime bounds | Hand edits are checked when config loads. Runtime `MODIFY_BOX_CFG` changes may skip those checks. |
| `box_extrude_retry_num` | Present in config, but practical retry behavior may be governed by fixed retry paths. |
| `Tn_line_len` and `cr_machine` | Exposed/stored, but their practical runtime effect is unclear from observed behavior. |
| `buffer_empty_len` max mismatch | Config accepts up to `60`; the web config endpoint reports max `30`. |
