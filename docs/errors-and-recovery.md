# Errors and recovery

## Scope

This document explains observable error and recovery behavior for the material-box
wrapper interface. It is written for readers who do **not** have access to source
code and for authors of independent wrappers.

It covers:

- box protocol response states versus workflow errors;
- when printing may pause;
- how public retry and clear commands should be understood;
- recovery paths for cut, retrude, loading, printer-side extrusion, flush,
  filament runout, and empty-print conditions.

For serial response codes, see [`serial-protocol.md`](serial-protocol.md). For
state names such as active material and remaps, see
[`state-model.md`](state-model.md). For command syntax, see
[`klipper-macros.md`](klipper-macros.md). For public caveats that affect custom
macros and independent wrappers, see
[`compatibility-caveats.md`](compatibility-caveats.md).

## Error layers

The wrapper exposes several kinds of failure information. They are related, but
not the same thing.

| Layer | Example | Meaning |
|---|---|---|
| Box protocol response state | `CRC_ERR`, `FILAMENT_ERR`, `EXTRUDE_ERR7` | Result code returned by the material box. |
| User-visible alarm | nozzle blocked, filament error, state error | Message reported to Klipper/UI; during printing it may pause the print. |
| Workflow recovery condition | cut failure, retrude failure, load failure, flush failure | High-level phase used by retry/clear behavior. |
| Deferred command/retry work | material command waiting behind an error | Some direct commands may be retried later by the public recovery command. |
| Background flow state | ending-material/refill in progress | Used by runout and refill handling. |

A serial error does **not** always become a workflow recovery condition. Some
serial errors only fail the current command; the surrounding material-change
phase decides whether to retry, report, or save a recovery condition.

## Immediate alarm behavior

When a user-visible alarm is reported:

1. the message is exposed to Klipper/UI;
2. if the print is currently printing, the print may be paused;
3. recovery waits until the print is in a safe paused state before continuing.

A user-visible alarm is not the same as a retry plan. Use the recovery command
only after checking which phase failed and whether the hardware is safe.

## One active recovery condition

Compatibility behavior is easiest to understand as one active recovery condition
at a time.

```text
if no recovery condition is active:
    record the current failed phase
else:
    keep the first active condition unless the wrapper explicitly replaces it
```

Practical consequences:

- the first failure is often more important than later symptoms;
- clearing an error can discard pending retry/deferred command state;
- exact UI message text is less reliable than the failed phase and sensor state;
- custom macros should add visible assertions around critical phase boundaries.

Useful visible assertions:

- local filament sensor is clear after retrude/unload;
- expected slot is loaded after `BOX_EXTRUDE_MATERIAL`;
- expected slot remains loaded after flush;
- target box is not `PRELOADING` before starting another material command.

## High-level recovery categories

| Category | Typical meaning |
|---|---|
| cut failure | Cutter motion or cut sensor did not confirm a cut. |
| retrude failure | Old material did not unload/retract as expected. |
| box-load failure | Target material did not load from the box side. |
| printer-extruder-side failure | Material reached the wrapper's load phase but the printer-side extrusion phase failed. |
| state/mode failure | Box rejected a command because its current mode was not valid. |
| filament/runout failure | Expected material was not present or ran out. |
| empty-print / movement mismatch | Material movement appeared inconsistent with printer extrusion demand. |
| flush failure | Material-change purge/flush failed. |
| print-end failure | End-print cleanup failed. |
| direct macro failure | A user-invoked material macro failed and may need retry or manual clear. |

## Protocol response handling overview

Serial response states are documented in detail in
[`serial-protocol.md`](serial-protocol.md#response-state-codes). This table
summarizes the practical recovery meaning.

| Response state | Practical behavior |
|---|---|
| `OK` | Parse response data and continue. |
| `CRC_ERR` | Retry the serial request within the command's retry policy. |
| `PARAMS_ERR` | Report parameter error and fail the command. |
| `STATE_ERR` | Usually report a state/mode error; `GET_BOX_STATE` has special query behavior. |
| `EXTRUDE_ERR*` | Loading/progress failure class; surrounding load logic decides recovery. |
| `RETRUDE_ERR*` | Unload/retrude failure class; surrounding unload logic decides recovery. |
| `FILAMENT_ERR` | Filament/runout handling; may start ending-material or auto-refill behavior. |
| `SPEED_ERR` / `ENWIND_ERR` | Movement mismatch or entanglement-style failure. |
| `MOTOR_LOAD_ERR` | Motor load failure; stop and inspect. |
| `UPDATE_STATE` | Slot insert/extract/completed state update. |
| timeout | `GET_BOX_STATE` may return no data; other commands fail and may mark communication unhealthy. |

Important distinction: a serial failure can be handled within a material-change
phase before a public workflow error appears.

## Commands that may defer failure handling

Several material commands can leave recovery work pending instead of aborting at
exactly the failing command.

| Command | Practical consequence |
|---|---|
| `BOX_CUT_MATERIAL` | On failure, use recovery or manual inspection before continuing. |
| `BOX_RETRUDE_MATERIAL` | Verify the local sensor is clear before loading a new material. |
| `BOX_EXTRUDE_MATERIAL` | Always pass `TNN`; verify the target is loaded afterward. |
| `BOX_EXTRUDER_EXTRUDE` | Treat failure as separate from box-side loading. |
| `BOX_MATERIAL_CHANGE_FLUSH` | Verify flush completed before resuming print. |

See [`compatibility-caveats.md`](compatibility-caveats.md) for custom macro rules.

## Built-in self-retry before public failure

The box-load phase may attempt bounded recovery before reporting a public load
failure. Typical strategies include:

| Strategy | Purpose |
|---|---|
| reset to `IDLE` and retry | Recover from a rejected load start. |
| retrude toward material sensor | Return material to a known position before restarting. |
| loosen/retry material path | Recover from sensor/progress mismatch. |
| final buffer/local verification retry | Confirm material reached the expected path. |

A visible box-load failure usually means these bounded attempts did not produce a
verified loaded state. Independent wrappers can choose their own retry policy; do
not loop indefinitely.

## `BOX_TNN_RETRY_PROCESS`

`BOX_TNN_RETRY_PROCESS` is the main public retry command.

Syntax:

```gcode
BOX_TNN_RETRY_PROCESS
```

Conceptual behavior:

```text
if a recovery condition is active:
    choose the recovery path for the failed phase
    move/heat/set box modes as needed
    retry the failed or following phases
    resume the print only if recovery succeeds and the printer is paused
else:
    do nothing or report that no retry is needed
```

### Retry dispatch table

| Failed phase | Retry behavior |
|---|---|
| cut | Clear the failure and rerun the original material action if safe. |
| retrude | Set affected box idle, unload/retrude as needed, then continue through load/extrude/flush. |
| box-load | Set target box idle, reload target material, run printer-side extrusion, then flush. |
| printer-extruder-side | Set target box idle, retry printer-side extrusion, then flush. |
| flush | Reposition, set target box mode, rerun material-change flush. |
| filament/runout | Wait for ending-material/refill handling if needed, then load/extrude/flush target material. |
| empty-print/movement mismatch | Re-establish print mode and heat as needed; this is not a full material switch. |
| direct macro failure | Clear the condition and rerun deferred public macro work when supported. |
| print-end failure | Set last material box idle if known, clear condition, then run end-print cleanup. |

Errors not covered by a known retry path generally need `BOX_ERROR_CLEAR` or
manual intervention.

## `BOX_ERROR_CLEAR`

`BOX_ERROR_CLEAR` acknowledges and clears the active recovery condition. It may
also set an affected box to `IDLE`.

Syntax:

```gcode
BOX_ERROR_CLEAR
```

Use it when:

- hardware has been inspected and made safe;
- you do **not** want automatic retry;
- you need to clear stale recovery state before manual operation.

Do not use it when you expect a deferred material command to replay; clearing can
cancel pending retry work.

## Recovery path summaries

### Cut failure

Recovery normally reruns the original material action after clearing the failure.
If cutter position or cut-sensor behavior is suspect, recalibrate before retrying.
See [`sensors-and-hardware.md`](sensors-and-hardware.md#cutter--cut-sensor) and
[`configuration.md`](configuration.md#cutter-and-pre-cut-position-options).

### Retrude failure

Typical recovery:

```text
set affected box to IDLE
if local filament is still present, move/cut/retrude as needed
retry old-material retrude
load target material
run printer-side extrusion
flush
```

If later stages fail, the active recovery condition changes to the later failed
phase.

### Box-load failure

Typical recovery:

```text
set target box to IDLE
move to load/extrude position
retry box-side loading
retry printer-side extrusion
flush
```

### Printer-extruder-side failure

Typical recovery:

```text
set target box to IDLE
retry printer-side extrusion
flush
```

### Flush failure

Typical recovery:

```text
move to extrude/flush position
set target box IDLE or PRINT as needed
rerun material-change flush
restore motion/fan/Z state
```

Repeated blockage-style failures should be treated as a real hardware/material
problem, not as a reason to keep purging indefinitely.

### Filament runout / auto-refill

Filament/runout handling may involve ending-material extrusion and automatic
replacement selection before normal load/extrude/flush recovery. See
[`auto-refill.md`](auto-refill.md) for details.

### Empty-print / movement mismatch

This condition indicates material movement did not match printer extrusion demand
or the material may be tangled. Recovery usually re-establishes box `PRINT` mode
and heats to the target material temperature; it is not a full cut/reload flow by
itself.

### Direct macro failure

A direct macro failure may be recoverable through the retry command, but custom
macros should not rely on hidden state alone. Verify visible sensor and loaded
slot state before continuing.

### Print-end failure

If the last active material is known, recovery sets that box idle, clears the
condition, and runs end-print cleanup.

## Filament runout and ending-material behavior

`FILAMENT_ERR` and related extrusion-empty states enter special handling.

Conceptually:

```text
if no active material is known:
    report or ignore according to the current command context
else if auto-refill is enabled and print is printing/paused:
    mark ending-material/refill handling active
    pause or defer normal recovery until remaining material is handled
else:
    report filament unavailable/runout
```

The wrapper may also run a background stabilization/tighten-up behavior after
runout. Treat that as part of the runout recovery policy, not as a standalone
user command.

## Nozzle-blocked / measuring-wheel errors

Flush/extrusion paths may use the measuring wheel to detect abnormal movement.

Conceptual behavior:

```text
measure wheel before extrusion
extrude a segment
measure wheel after extrusion
if movement delta crosses configured threshold:
    report possible blockage
    fail the flush/extrusion step
```

The configured threshold is `diff_length`; see
[`configuration.md`](configuration.md#material-flow-and-flush-options). Validate
its sign and scale on the target machine before treating it as a hard safety
criterion.

## Timeout behavior

| Command type | Timeout behavior |
|---|---|
| `GET_BOX_STATE` | May return no data; heartbeat/recovery logic can later mark a box disconnected. |
| Other serial commands | Fail the command and may run communication timeout handling for the address. |

## Which recovery command should I use?

| Situation | Suggested command |
|---|---|
| A material-change operation failed and the print is paused | `BOX_TNN_RETRY_PROCESS` |
| You want to acknowledge/clear an error without retrying the failed operation | `BOX_ERROR_CLEAR` |
| A direct material macro failed and should be retried | `BOX_TNN_RETRY_PROCESS`, after checking hardware is safe |
| State is wrong but hardware is safe/idle | `BOX_ERROR_CLEAR`, then inspect state/mapping commands |
| Filament ran out and auto-refill is expected | Wait for/refire refill handling, then use `BOX_TNN_RETRY_PROCESS` if needed |
| Cutter position/sensor failed repeatedly | Calibrate cutter settings before retrying |

## Known caveats and uncertainties

| Area | Notes |
|---|---|
| One active recovery condition | Later failures may be hidden behind the first active condition. |
| Deferred command behavior | Some direct material commands may defer retry work instead of aborting immediately. |
| Clear versus retry | `BOX_ERROR_CLEAR` can discard pending retry work. |
| State/mode failure retry | State/mode failures usually need clear/manual correction rather than a large retry flow. |
| Flush retry target | Flush recovery needs a clear target material; stale or missing target state can require manual intervention. |
| Buffer state | Treat buffer queries as immediate observations unless your wrapper owns a reliable cache. |
| Filament sensor semantics | Runout behavior depends on local sensor truth values; validate on the target machine. |
| Auto-refill interplay | Runout may defer recovery while ending-material/refill handling is active. |
| Message text | UI messages are less stable than phase, response category, and sensor state. |
