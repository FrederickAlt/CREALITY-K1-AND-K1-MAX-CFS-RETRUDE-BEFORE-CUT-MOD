# Runtime reference appendix

## Scope

This appendix records observed runtime constants and command side effects useful
when operating the material-box wrapper or writing an independent wrapper. It intentionally avoids deep material-change workflow detail; see
[`material-change-flow.md`](material-change-flow.md) for sequencing and
[`errors-and-recovery.md`](errors-and-recovery.md) for retry behavior.

## Hard-coded runtime names and files

| Item | Runtime value | Used for | Notes |
|---|---|---|---|
| Primary saved `box.cfg` path | `/usr/data/printer_data/config/box.cfg` | `SAVE_BOX_CFG` target lookup | First existing path wins. |
| Fallback saved `box.cfg` path | `/mnt/UDISK/printer_data/config/box.cfg` | `SAVE_BOX_CFG` target lookup | Used only when the primary path is absent. |
| Config backup basename | `box.cfg.1` | `SAVE_BOX_CFG` backup | Created next to the selected config file before rewrite. |
| Default/generated local filament sensor | `filament_sensor_2` | Example `filament_sensor` value and several hard-coded control commands | Sensor lookup can use the configured `filament_sensor`, but enable/disable paths still target this fixed name. |
| Local filament sensor object prefix | `filament_switch_sensor <name>` | Klipper object lookup | With the default name this is `filament_switch_sensor filament_sensor_2`. |
| Model/cleaning fan output | `fan0` | Blow, close-model-fan, save/restore, resume extrusion | Missing output pin is tolerated except paths that issue direct `SET_PIN PIN=fan0` without checking. |
| Secondary saved fan output | `fan2` | Save/restore fan state | Optional. |
| Fixed USB serial adapter path | `/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0` | New-box discovery guard | Separate from the configured `bus` object. Non-CH340 or renamed adapters may need profile/firmware adjustment. |
| Persisted box state file | `creality/userdata/box/tn_data.json` under Klipper/base directory | Last active slot, enable state, virtual mapping, material identity | Used by power-loss restore and state persistence. |
| Material database file | `creality/userdata/box/material_database.json` under Klipper/base directory | Material display name, target temperature, maximum extrusion speed lookup | If missing or malformed, material-name/speed/temp lookup falls back to defaults. |
| Flushing sign file | `creality/userdata/config/flushing_sign` under Klipper/base directory | One-shot flag checked after extrusion | If present when checked, the wrapper deletes it and treats the flag as set. |
| Config API endpoint | `box/get_box_cfg` | Web/UI config metadata | Returns selected config values and UI min/max metadata. |
| Creality slicer version gate | `5.1.7.10471` | Prime-tower purge metadata use | Prime-tower purge metadata is considered only for CREALITY slicer metadata at or above this version. |

## Public G-code command inventory

### Generated tool commands

| Command range | Kind | Primary effect |
|---|---|---|
| `T1A`..`T4D` | physical-slot tool commands | Run the full wrapper material-change action for a physical slot after mapping. |
| `T0`..`T15` | virtual tool commands | Resolve through `Tnn_map`, then run the full wrapper material-change action. |

### Registered named commands

| Command | Kind | Main side-effect class |
|---|---|---|
| `BOX_ADD_TNN` | resume/state helper | Writes resume target state on the toolhead side. |
| `BOX_BLOW` | hardware action | Toggles `fan0`, waits, then turns it off. |
| `BOX_CHECK_MATERIAL_REFILL` | refill helper | May continue automatic refill or ending-material handling. |
| `BOX_COMMUNICATION_TEST` | serial diagnostic | Sends repeated box protocol requests. |
| `BOX_CREATE_CONNECT` | direct serial | Requests box connection; ignored while printing. |
| `BOX_CTRL_CONNECTION_MOTOR_ACTION` | direct serial | Starts/stops connection motor action. |
| `BOX_CUT_HALL_TEST` | cutter diagnostic | Registered only when `switch_pin` exists; moves/tests cutter hall behavior. |
| `BOX_CUT_HALL_ZERO` | cutter diagnostic | Registered only when `switch_pin` exists; cutter hall/zero routine. |
| `BOX_CUT_MATERIAL` | workflow/hardware action | Turns model fan off and runs cutter flow; records macro cut error on failure. |
| `BOX_CUT_STATE` | sensor diagnostic | Reports local cutter switch state. |
| `BOX_CUSTOM_COMMAND` | UI/helper command | Runs one of the fixed helper subcommands listed below. |
| `BOX_ENABLE_AUTO_REFILL` | runtime state | Stores the runtime auto-refill flag. |
| `BOX_ENABLE_CFS_PRINT` | persisted mode/state | Writes CFS-print enable state and controls `filament_sensor_2`. |
| `BOX_END` | lifecycle/workflow | May cut, retrude, clear loaded material state, and move safe. |
| `BOX_END_PRINT` | lifecycle | Sets connected boxes to end-print preloading/tighten-up state, resets mapping, clears power-loss data, disables `filament_sensor_2`. |
| `BOX_ERROR_CLEAR` | recovery/state | Clears recorded wrapper error; may set affected box idle. |
| `BOX_EXTRUDE_2_PROCESS` | direct serial | Sends model-2 box extrude command. |
| `BOX_EXTRUDE_MATERIAL` | material phase | Loads material from selected box slot; requires a valid `TNN` in practice. |
| `BOX_EXTRUDE_PROCESS` | direct serial/test | Sends legacy/debug extrude-process command. |
| `BOX_EXTRUDER_EXTRUDE` | material phase/state | Updates extruder-side material bookkeeping; resume-guarded. |
| `BOX_EXTRUSION_ALL_MATERIALS` | material/recovery phase | Extrudes ending/remnant material; disables `filament_sensor_2` in this path. |
| `BOX_FIND_CUT_POS` | calibration/motion/config | Homes as needed, searches Y with cut sensor, writes `cut_pos_y` on success, restores acceleration. |
| `BOX_GENERATE_FLUSH_ARRAY` | flush diagnostic | Parses/logs a 4x4 flush matrix string. |
| `BOX_GET_BOX_STATE` | direct serial diagnostic | Queries state/environment payload. |
| `BOX_GET_BUFFER_STATE` | direct serial diagnostic | Queries buffer state. |
| `BOX_GET_FILAMENT_SENSOR_STATE` | direct serial diagnostic | Queries box material or connection sensor mask. |
| `BOX_GET_FIVE_WAY_STATE` | sensor diagnostic/state | Queries connected boxes' connection sensors and updates per-box marker. |
| `BOX_GET_FLUSH_LEN` | flush diagnostic | Computes color-change flush length from source/target RGB values. |
| `BOX_GET_FLUSH_VELOCITY_TEST` | flush diagnostic/test | Computes/logs flush velocity profile for two slots. |
| `BOX_GET_REMAIN_LEN` | direct serial diagnostic | Queries remaining length for a slot mask. |
| `BOX_GET_RFID` | direct serial diagnostic | Queries RFID for a slot mask. |
| `BOX_GET_VERSION_SN` | direct serial diagnostic | Queries version and serial number. |
| `BOX_GO_TO_BOX_EXTRUDE_POS` | motion | Moves to retrude/box-extrude position. |
| `BOX_GO_TO_EXTRUDE_POS` | motion | Moves to configured extrude/flush position; no-ops during resume. |
| `BOX_MATERIAL_CHANGE_FLUSH` | material phase | Runs target-aware material-change flush. |
| `BOX_MATERIAL_FLUSH` | material phase/test | Heats/extrudes with supplied/default profile, cleans nozzle, retracts. |
| `BOX_MEASURING_WHEEL` | direct serial diagnostic | Reads or clears measuring wheel. |
| `BOX_MODIFY_TN` | state mutation | Changes and persists virtual-to-physical slot mapping. |
| `BOX_MODIFY_TN_DATA` | state mutation | Directly edits live material-box state fields. |
| `BOX_MODIFY_TN_INNER_DATA` | state mutation/test | Attempts inner state edit; public shape is incomplete for sensor subparts. |
| `BOX_MOVE_TO_CUT` | motion/hardware | Runs cutter movement flow and temporary acceleration changes. |
| `BOX_MOVE_TO_SAFE_POS` | motion | Moves to configured safe Y when XY is homed. |
| `BOX_NOZZLE_CLEAN` | motion/fan | Runs wipe/cleaning movement, fan operations, waits. |
| `BOX_POWER_LOSS_RESTORE` | persisted state | Restores selected state from persisted box data. |
| `BOX_RESTORE_FAN` | hardware state | Restores saved `fan0`/`fan2` output values. |
| `BOX_RETRUDE_MATERIAL` | material phase | Unloads/retrudes current material; resume-guarded. |
| `BOX_RETRUDE_MATERIAL_WITH_TNN` | material phase/test | Retrudes a specific or detected material; observed behavior is not resume-guarded. |
| `BOX_RETRUDE_PROCESS` | direct serial | Sends box retrude command. |
| `BOX_RESUME_EXTRUDE` | resume workflow | Re-primes same material during resume; moves Z/XY, heats, changes fan/mode, extrudes, cleans, restores. |
| `BOX_SAVE_EXTRUDE_POS` | calibration/config | Saves current or supplied XY to `extrude_pos_x/y`, then persists config. |
| `BOX_SAVE_FAN` | hardware state | Saves current `fan0`/`fan2` values and turns them off. |
| `BOX_SEND_DATA` | direct serial/test | Sends a normal wrapper request with caller-selected command byte and digit-parsed payload. |
| `BOX_SET_BOX_MODE` | direct serial/state | Sets a box to `PRINT` or `IDLE` for a slot or no slot. |
| `BOX_SET_PRE_LOADING` | direct serial/state | Opens/closes/runs/tightens preloading and may update runtime power-on flag. |
| `BOX_SHOW_TNN_INNER_DATA` | state diagnostic | Prints current material-box state object. |
| `BOX_START_PRINT` | lifecycle | Closes preloading when configured and disables tighten-up for connected boxes. |
| `BOX_TIGHTEN_UP_ENABLE` | direct serial/state | Enables/disables box tighten-up behavior. |
| `BOX_TN_EXTRUDE` | material test | Runs configured default extrusion/flush helper. |
| `BOX_TNN_RETRY_PROCESS` | recovery workflow | Retries the currently recorded wrapper error path. |
| `BOX_UPDATE_SAME_MATERIAL_LIST` | state derivation | Recomputes compatible-material groups. |
| `DO_AFTER_PAUSE` | lifecycle helper | Waits for pause state and runs pause-after hook. |
| `MODIFY_BOX_CFG` | runtime config | Mutates supported in-memory config fields and queues pending save. |
| `MOVE_BOX_CUT_POS` | calibration/motion | Moves to computed cut position. |
| `MOVE_BOX_PRE_CUT_POS` | calibration/motion | Moves to pre-cut position. |
| `RESTORE_POSITION` | motion/state restore | Runs post-extrude restore unless in resume. |
| `SAVE_BOX_CFG` | persistent config | Rewrites selected `box.cfg` path from pending runtime config changes. |
| `TEST_BOX_CLEAN` | test alias | Runs the nozzle-clean action. |
| `TEST_BOX_EXTRUDE` | calibration/motion | Moves to configured extrude XYZ test position. |
| `WAIT_EXTRUSION_ALL_MATERIALS` | synchronization | Waits for ending-material/refill extrusion flows. |

## Fixed `BOX_CUSTOM_COMMAND` subcommands

| Subcommand | Side effects | Result string |
|---|---|---|
| `XYZ_ZERO` | Homes all axes. | `XYZ_ZERO=0` |
| `COORDINATES_ADJUST_PREPARE` | Moves Z to 25 mm, moves Y to `safe_pos_y`, moves X to `extrude_pos_x`, waits for motion. | `COORDINATES_ADJUST_PREPARE=0` |
| `COORDINATES_ADJUST_SAVE_POS` | Reads current toolhead XY, writes `extrude_pos_x/y` through `MODIFY_BOX_CFG`, then runs `SAVE_BOX_CFG`. | `COORDINATES_ADJUST_SAVE_POS=0` |
| `Y_SAFE` | Moves Y to `safe_pos_y`. | `Y_SAFE=0` |

Unknown `CMD` values are ignored.

## Side-effect matrix for high-risk command families

| Command/family | Fan writes | Acceleration/speed writes | Z writes | Box mode/state writes | Persistent file writes | Notes |
|---|---|---|---|---|---|---|
| `T0`..`T15`, `T1A`..`T4D` | Yes, through clean/flush/cut phases | Yes, through cut/clean/restore phases | Yes, through safe/extrude/restore phases | Yes, current/target box mode and `last_cmd` state | May write persisted state/errors | Full workflow; use `material-change-flow.md` for sequence. |
| `BOX_MOVE_TO_CUT` | No direct fan write | Temporarily sets cut acceleration and restores it | May home/move axes before cutting | No box protocol mode change by itself | No | Skips cutting if local filament sensor reports no filament. |
| `BOX_CUT_MATERIAL` | Turns model fan off through `fan0` close path | Same as cutter flow | Same as cutter flow | Records macro error state on failure | Error persistence only | More than a simple cutter toggle. |
| `BOX_NOZZLE_CLEAN` / `TEST_BOX_CLEAN` | Uses cleaning/model fan behavior | Temporarily changes motion settings | May move Z as part of cleaning path | No direct serial mode write | No | Emits waits around motion/fan operations. |
| `BOX_BLOW` | Sets `fan0` high, waits 3 seconds, then sets it to zero | No | No | No | No | Uses `fan0` fixed name. |
| `BOX_SAVE_FAN` | Saves then sets `fan0`/`fan2` to zero when present | No | No | No | No | Missing pins are tolerated. |
| `BOX_RESTORE_FAN` | Restores saved `fan0`/`fan2` values when present | No | No | No | No | Clears saved values after restore. |
| `BOX_MATERIAL_FLUSH` | Indirectly through nozzle clean | Indirectly through extrusion/cleaning | Indirectly through cleaning | No direct mode write | May record errors | Resume-guarded; final direct retract is 5 mm. |
| `BOX_MATERIAL_CHANGE_FLUSH` | Yes through flush/clean paths | Yes through flush/clean paths | Yes through flush/clean paths | May change box state in error/retry contexts | May record errors | Target-aware flush phase only, not the whole toolchange. |
| `BOX_EXTRUDE_MATERIAL` | No direct fan write | May move/extrude and wait | May move through wrapper-managed paths | Sets box/extrude state and active material state | May record errors/state | `TNN` should be supplied. |
| `BOX_RETRUDE_MATERIAL` | No direct fan write | Extrudes/retracts and waits | May move through wrapper-managed paths | Sets selected box `IDLE`, sends retrude protocol actions, clears active material on success | May record errors/state | Resume-guarded. |
| `BOX_RETRUDE_MATERIAL_WITH_TNN` | No direct fan write | Extrudes/retracts and waits | May move through wrapper-managed paths | Sends retrude protocol actions | May record errors/state | Observed behavior is not resume-guarded. |
| `BOX_FIND_CUT_POS` | No | Sets acceleration to 30000/30000 during search, then restores previous values | Homes all axes if Z is not homed; otherwise homes X/Y | No direct mode write | Writes `cut_pos_y` through config commands on success | Requires no local filament present. |
| `BOX_RESUME_EXTRUDE` | Saves observed `fan0`, sets `fan0` to zero, restores only if previous value was positive | Restores saved G-code speed at end | Raises/restores Z and moves to extrude/safe positions | Sets target box to `PRINT`; may record error state | May record errors | Runs only when resume target matches current material content. |
| `BOX_ENABLE_CFS_PRINT` | No | No | No | Writes box enable state | Persists enable state | `ENABLE=0` enables `filament_sensor_2`; `ENABLE=1` disables it. |
| `BOX_END_PRINT` | No direct fan write | No | No | Resets mapping, clears power-loss fields, sets enable state to `0` | Persists/cleans state | Disables `filament_sensor_2`. |
| `BOX_ERROR_CLEAR` | No | No | No | May set affected box `IDLE`; clears error state | Persists cleared error fields | Does not undo physical movement already performed. |
| `MODIFY_BOX_CFG` / `SAVE_BOX_CFG` | No | No | No | Mutates live config object | `SAVE_BOX_CFG` rewrites `box.cfg` and `box.cfg.1` | Runtime mutation may skip normal config loader bounds. |

## Resume guard summary

| Command | Observed resume behavior |
|---|---|
| `BOX_EXTRUDE_MATERIAL` | Logs warning and does nothing during resume. |
| `BOX_RETRUDE_MATERIAL` | Logs warning and does nothing during resume. |
| `BOX_EXTRUDER_EXTRUDE` | Logs warning and does nothing during resume. |
| `BOX_MATERIAL_FLUSH` | Logs warning and does nothing during resume. |
| `BOX_GO_TO_EXTRUDE_POS` | Logs warning and does nothing during resume. |
| `RESTORE_POSITION` | Logs warning and does not restore during resume. |
| `BOX_CUT_MATERIAL` | Logs warning and does nothing during resume. |
| `BOX_RETRUDE_MATERIAL_WITH_TNN` | No observed resume guard; can still run. |
| `BOX_RESUME_EXTRUDE` | Dedicated resume path; performs substantial movement/extrusion if preconditions pass. |

## Flush-related runtime references

| Runtime item | Detail |
|---|---|
| Color flush input command | `BOX_GET_FLUSH_LEN SCV=<RRGGBB> TCV=<RRGGBB>` uses six hexadecimal characters for each color. |
| Color flush calculation inputs | Source color, target color, configured `nozzle_volume`, and `flush_multiplier`. |
| Slicer metadata path | Uses virtual SD card g-code metadata, not a standalone config option. |
| Prime-tower metadata gate | Only considered while printing, after the first Tnn phase is finished, with CREALITY slicer metadata version at least `5.1.7.10471`. |
| Flushing sign behavior | The wrapper checks `creality/userdata/config/flushing_sign`; if the file exists, it is deleted and the check returns true once. |
