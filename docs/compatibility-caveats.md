# Compatibility caveats

## Scope

This appendix records compatibility and safety caveats that matter when operating
the material box or writing an independent wrapper. It describes observable
command behavior and user-facing consequences only. It does not describe vendor
source code or implementation internals.

For normal operation, see:

- [`klipper-macros.md`](klipper-macros.md)
- [`errors-and-recovery.md`](errors-and-recovery.md)
- [`hardware-validation-checklist.md`](hardware-validation-checklist.md)

## Commands may defer failure handling

Some public material commands may leave the wrapper in a recoverable error state
instead of stopping exactly at the failing command. A following recovery command
may then decide what to retry.

Practical rule for custom macros:

```text
after every critical phase:
    verify the externally visible result before continuing
```

Useful checks include:

- local toolhead filament sensor is clear after unload/retrude;
- expected target slot is reported loaded after box loading;
- target slot is still reported loaded after flushing;
- target box is not in preloading/error mode before another material command.

## Direct material commands need visible verification

`BOX_EXTRUDE_MATERIAL TNN=<slot>` should be called with an explicit target slot.
After it returns, do not assume success from command completion alone. Verify the
observable state:

```text
run BOX_EXTRUDE_MATERIAL TNN=<target>
verify target slot is loaded or local filament is present
if verification fails:
    stop the custom sequence and use recovery/manual inspection
```

This is especially important for custom lower-level toolchange macros that do
not use the wrapper-managed `T0`..`T15` path.

## Retry versus clear

The two main recovery commands have different meanings.

| Command | Compatibility behavior |
|---|---|
| `BOX_TNN_RETRY_PROCESS` | Attempts the recorded recovery path. It may move axes, heat, set box modes, flush material, and resume a paused print after success. |
| `BOX_ERROR_CLEAR` | Acknowledges/clears the recorded error. It may set an affected box to `IDLE`, but it does not generally repeat the failed operation. |

Use retry when you want the wrapper to attempt recovery. Use clear only after the
hardware is safe and you do not want pending recovery work to run.

## Buffer-state caution

`BOX_GET_BUFFER_STATE` reports a buffer value for the addressed box. Some status
surfaces may not refresh in the same way as the immediate command response.

For an independent wrapper, treat buffer state as an immediate command result:

```text
query buffer state
use the returned response for the current decision
query again before making a later decision that depends on the buffer
```

Do not rely on a cached buffer field unless your wrapper owns and updates that
cache.

## Runtime config write caveat

Runtime config commands are convenient for calibration, but zero-like values may
not be a reliable way to update persistent config through the public save path.
When intentionally setting an option to `0`, prefer direct config-file editing
followed by the normal reload/restart process.

## Motion and hardware side effects

Several commands that sound like simple actions can move hardware or change
printer state.

| Command family | Important side effects |
|---|---|
| `BOX_CUT_MATERIAL`, cutter movement commands | Can home/move axes and temporarily change acceleration. |
| `BOX_MATERIAL_CHANGE_FLUSH`, `BOX_MATERIAL_FLUSH` | Can heat, extrude, clean, move, retract, and use fans. |
| `BOX_TNN_RETRY_PROCESS` | Can run a larger recovery sequence and resume a print. |
| `BOX_ENABLE_CFS_PRINT` | Controls the local fallback filament sensor behavior. |
| `BOX_END_PRINT` | Resets mapping and cleans persisted resume fields. |

Run motion/extrusion commands only with the toolhead path clear and the hotend,
cutter, and box state understood.

## Resume behavior caution

Several material and motion commands intentionally do nothing during resume
handling. One specific retrude command can still move material during resume. For
custom wrappers or macros, make resume a first-class state:

```text
if printer is resuming:
    avoid ordinary material-change commands
    use the dedicated resume/re-prime path or wait until resume handling ends
```

## Independent-wrapper checklist

A new wrapper does not need to reproduce every compatibility behavior above. It
should, however, make explicit choices for these observable contracts:

1. how active loaded material is tracked;
2. how material commands report failure;
3. whether recovery is automatic, manual, or operator-confirmed;
4. whether clearing an error also cancels pending retry work;
5. how buffer state is cached or re-queried;
6. how remaps and auto-refill assignments persist during a print;
7. which commands are allowed during pause/resume;
8. what safety checks run before cutter, unload, load, and flush operations.
