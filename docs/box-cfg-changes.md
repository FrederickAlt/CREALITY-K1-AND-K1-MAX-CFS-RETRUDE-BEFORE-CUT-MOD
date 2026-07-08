# Robust `box.cfg` changes

This note explains the generated [`../box.cfg`](../box.cfg) and why it differs
from the original profile.

The goal is to keep your existing T0-T15 mapping style while making the custom
swap flow safer around stale state, missing `TNN` parameters, pre-cut retraction,
feedrates, and failed loads.

Related references:

- [`configuration.md`](configuration.md) for `[box]` options and calibration
  meanings.
- [`klipper-macros.md`](klipper-macros.md) for command side effects and macro
  risk categories.
- [`state-model.md`](state-model.md) for `last_cmd`, per-box `filament`, and
  observed active-material state. `currentcfsslot` is custom state local to this
  generated `box.cfg`.
- [`sensors-and-hardware.md`](sensors-and-hardware.md) for the local filament
  sensor, box sensors, cutter sensor, and hard-coded `filament_sensor_2` caveat.
- [`material-change-flow.md`](material-change-flow.md) for the normal
  cut/retrude/load/flush phases.
- [`errors-and-recovery.md`](errors-and-recovery.md) for error clearing and retry
  behavior.

## Important config changes

### `Tn_retrude_velocity` lowered

Changed:

```ini
Tn_retrude_velocity: 2400
```

Your original value was `5000`. That is aggressive for retract/retrude paths and
can contribute to gear skipping. `2400` is a conservative starting point; raise it
only after swaps are reliable.

### Existing position values preserved

The generated config keeps your original physical positions, including:

- cutter positions;
- hopper/extrude position;
- `safe_pos_y`;
- `extrude_pos_z`;
- `switch_pin`;
- `version: 1`;
- `buffer_empty_len: 10`.

Comments with old alternative values were simplified so `box.cfg` is easier to
read and less fragile for runtime save tools.

## Removed fragile preloading polling

The old `CHECK_PRELOADING` loop from the supplied original profile was removed
from `SWAP_FILAMENT`.

Reason: Klipper renders a macro before executing the generated commands. A loop
that repeatedly checks `printer['box'][...]` inside one macro invocation does not
reliably observe state changes caused by commands generated earlier in that same
macro.

Instead, the new `SWAP_FILAMENT` performs a **snapshot guard** at macro start:

```text
if any box currently reports PRELOADING:
    abort and ask the user to wait
```

The plugin's high-level commands such as `BOX_EXTRUDE_MATERIAL` and
`BOX_RETRUDE_MATERIAL` still perform their own wrapper-managed waits where applicable.
The custom low-level pre-cut buffer commands do not reliably wait for preloading,
so the safest macro-level behavior is to refuse to start when preloading is
already visible.

## Safer pre-cut retraction

The supplied original `BOX_PRE_CUT_RETRUDE` had a dangerous shape: it printed a
warning when no filament was detected, but the final buffer command and retract
could still run outside the `if` block.

The new version:

- does nothing if the toolhead filament sensor is not triggered;
- refuses to drive a buffer when the current slot is unknown or invalid;
- refuses to run when `buffer_empty_len <= 0`;
- refuses to run when the current box is in `PRELOADING`;
- uses `SAVE_GCODE_STATE`, `M83`, and `G90` for predictable motion/extrusion
  mode;
- splits pre-cut retract into buffer-sized chunks;
- only runs the remainder when it is meaningful.

This directly addresses the safety distinction in
[`sensors-and-hardware.md`](sensors-and-hardware.md#local-printer-filament-sensor):
do not move filament if the toolhead sensor says no filament is present.

## Live state is preferred over `currentcfsslot`

The supplied original `SWAP_FILAMENT` trusted the saved variable `currentcfsslot`
heavily. That variable can become stale after a failed or interrupted swap.

The new flow derives the current slot from live wrapper state only:

```text
if toolhead filament sensor is not triggered:
    current_slot = None
else if exactly one plugin status entry reports a loaded slot in printer['box'][Tn]['filament']:
    current_slot = that live plugin slot
else if multiple loaded slots are reported:
    abort because wrapper state is ambiguous
else:
    abort because the toolhead has filament but the wrapper reports no active slot
```

`currentcfsslot` is no longer used as a fallback for deciding which buffer to
move. It is a recovery hint/output variable, not authoritative state.

See [`state-model.md`](state-model.md#state-machine-6-active-material-state) for
why active material state matters.

## `currentcfsslot` is saved only after verification

The new flow sets:

```gcode
SAVE_VARIABLE VARIABLE=currentcfsslot VALUE='"None"'
```

at the start of a real swap, after saving the previous value to
`previouscfsslot` for manual recovery context. It writes the requested slot only
after live post-load verification passes.

Verification checks both:

1. the local toolhead filament sensor is triggered;
2. the plugin reports the expected slot letter in `printer['box']['Tn']['filament']`.

This is done by `BOX_ASSERT_SLOT_LOADED`.

The new swap verifies twice:

```text
after BOX_EXTRUDE_MATERIAL / BOX_EXTRUDER_EXTRUDE, before purge:
    verify loaded, but do not save currentcfsslot yet

after purge/restore:
    verify again and then save currentcfsslot
```

This avoids a silent plugin load failure being followed by a purge of the old or
wrong material.

## `BOX_EXTRUDE_MATERIAL` is always called with `TNN`

Observed behavior does not safely handle `BOX_EXTRUDE_MATERIAL` without a
valid `TNN`. The generated config therefore uses:

```gcode
BOX_EXTRUDE_MATERIAL TNN={slot}
BOX_EXTRUDER_EXTRUDE TNN={slot}
```

and changes the display/load macros to delegate to `SWAP_FILAMENT`, which
requires a `SLOT` parameter.

This follows the caveat documented in
[`klipper-macros.md`](klipper-macros.md#material-and-flush-commands).

## Manual purge remains under macro control

This `box.cfg` keeps the manual purge path rather than using the wrapper's
`BOX_MATERIAL_CHANGE_FLUSH` command.

Purge length is still computed from the slicer/requested length plus the profile
constant:

```text
purge_length = LEN + variable_extrude_before_purge
```

Feedrate is computed from the supplied `F` parameter or, if `F` is absent, from
`variable_purge_speed`, then clamped:

```text
requested_purge_feedrate = F or variable_purge_speed * 60 / 2.405
purge_feedrate = min(requested_purge_feedrate, variable_max_purge_feedrate)
```

The profile uses:

```ini
variable_extrude_before_purge: 34.0
variable_purge_speed: 15.0
variable_max_purge_feedrate: 600.0
```

This keeps flushing/purging under the macro's control and avoids the wrapper's
built-in material-change flush.

## Absolute XY/Z motion is forced

The generated macros explicitly use:

```gcode
G90
M83
```

for predictable coordinates and relative extrusion. `M83` only changes extrusion
mode; it does not make XYZ absolute. Adding `G90` avoids accidental relative XY/Z
moves if the slicer or previous macro left the printer in relative mode.

## Tool-empty assertion after unload

The new `BOX_ASSERT_TOOL_EMPTY` aborts if the toolhead filament sensor still sees
filament after cut/retrude.

This prevents loading a new material on top of an old material that failed to
unload.

## Same-slot behavior

If the requested slot is already loaded, the macro avoids destructive
unload/reload. It only:

```text
sets the box to PRINT for that slot
verifies the live loaded-slot state
saves currentcfsslot if verification passes
```

## Display/load macros changed

The old load macros `BOX_LOAD_MATERIAL_WITH_MATERIAL` and
`BOX_LOAD_MATERIAL_WITHOUT_MATERIAL` called plugin load commands without `TNN`.
That is unsafe with observed wrapper behavior.

The generated versions simply delegate to:

```gcode
SWAP_FILAMENT {rawparams}
```

That means they require a slot, for example:

```gcode
BOX_LOAD_MATERIAL_WITHOUT_MATERIAL SLOT=T1A
```

If your display calls these without parameters, adapt the display macro or add a
machine-specific default deliberately. Do not silently default to a random slot
unless that is really what you want.

## What stayed intentionally the same

- Your T0-T15 wrapper style is preserved.
- Your CFS serial path is preserved.
- Your cutter switch pin is preserved.
- Your hopper/cut/safe coordinates are preserved.
- Your custom purge-after-load strategy is preserved, but made safer.

## Caveats

### This remains a custom swap flow

Your T0-T15 macros call `SWAP_FILAMENT`, not the plugin's built-in full `Tn_action`
flow. That is intentional here because the supplied profile already used a custom
purge/swap policy, but it means your macro owns more of the swap policy:

- active slot decisions from live wrapper `filament` status;
- `currentcfsslot` as a verified output/recovery hint;
- purge behavior;
- pre-cut retraction;
- final verification.

Several lower-level plugin macros may record recoverable errors instead of
raising a Klipper macro error immediately. The generated flow adds
sensor/plugin-state assertions after unload and load to catch visible failures,
but if the plugin has recorded recovery state you may still need
`BOX_TNN_RETRY_PROCESS` or `BOX_ERROR_CLEAR`; see
[`errors-and-recovery.md`](errors-and-recovery.md).

The config adds a best-effort `BOX_MODIFY_TN` mirror so manual remaps and normal
auto-refill remaps that emit `BOX_MODIFY_TN` can be resolved before loading.
However, plain macros cannot read the wrapper's persisted
`creality/userdata/box/tn_data.json`; a `BOX_POWER_LOSS_RESTORE` or manual JSON
edit can still make the macro-side mirror stale. Reissue the needed
`BOX_MODIFY_TN ...` command or run `CFS_RESET_TNN_MAP` deliberately after such
recovery flows. See [`auto-refill.md`](auto-refill.md) and
[`state-model.md`](state-model.md#state-machine-7-persisted-and-resume-state).

### `filament_sensor_2` is still hard-coded

The generated macros use:

```text
filament_switch_sensor filament_sensor_2
```

That matches your profile and also matches several hard-coded wrapper commands
noted in [`sensors-and-hardware.md`](sensors-and-hardware.md#local-printer-filament-sensor).
If you rename the sensor, update both your macros and any wrapper paths that
still hard-code `filament_sensor_2`.

### No macro can make inline polling fully reliable

The new snapshot preloading guard is safer than the old polling loop, but it is
not a live wait. If a box enters preloading after the macro is rendered, the macro
cannot see that state change through inline Jinja checks.

### Test with no print first

Recommended validation order:

Prerequisites before any motion/extrusion test:

- keep the toolhead path clear and stay near emergency stop;
- verify the target box is connected and not preloading;
- know which slot is actually loaded before passing `CUR_SLOT=...` to a low-level
  buffer command.

Recommended order:

1. `BOX_GET_BOX_STATE ADDR=1`
2. `BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL`
3. Home the machine or let `BOX_GOTO_HOPPER` home it, then test `BOX_GOTO_HOPPER`
   with the toolhead path clear.
4. Test `BOX_PRE_CUT_RETRUDE CUR_SLOT=<actual-loaded-slot>` only when that slot
   is physically known to be loaded and the box is idle. Do not hard-code `T1A`
   unless T1A is really the loaded slot.
5. `SWAP_FILAMENT SLOT=T1A LEN=20`
6. T0/T1 wrappers only after direct `SWAP_FILAMENT` works

Use the hardware checklist in
[`sensors-and-hardware.md`](sensors-and-hardware.md#suggested-hardware-validation-checklist)
and the validation checks in
[`hardware-validation-checklist.md`](hardware-validation-checklist.md).
