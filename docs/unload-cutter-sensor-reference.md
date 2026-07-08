# Unload, cutter, and sensor reference

## Scope

This reference describes observed unload/retrude, cutter, buffer, and
measuring-wheel behavior for the material-box wrapper. It is written as
interoperability and safe-operation documentation using prose, tables, and
pseudocode only.

For the overall tool-change sequence, see
[`material-change-flow.md`](material-change-flow.md). For basic hardware and
sensor definitions, see [`sensors-and-hardware.md`](sensors-and-hardware.md).
For option names and defaults, see [`configuration.md`](configuration.md).

## Key terms

| Term | Meaning |
|---|---|
| retrude / unload | Pull filament back from the printer path toward the material box. |
| active material | The physical slot remembered as currently loaded. This is stored as `last_cmd` in the wrapper state. |
| tool sensor / local sensor | Printer-side filament switch sensor near the toolhead/path. If no sensor is configured, the wrapper treats filament as present. |
| material sensor | Box-side slot sensor showing whether material exists at the slot/material position. |
| connection / five-way sensor | Box-side sensor showing whether material is in the connection path for a slot. |
| buffer | Box-side buffer state used during loading and some unload recovery. |
| measuring wheel | Box-side movement measurement used mainly during segmented extrusion/flush blockage checks. |

## Unload / retrude decision tree

The unload command first chooses one of three branches:

```text
if an active material slot is known:
    unload that exact slot
else if the local tool sensor does not detect filament:
    clean up any box slots detected by both material and connection sensors
else:
    infer the occupied path from box connection sensors and unload it
```

Failure normally propagates as a retrude failure in the full material-change
workflow. Some lower-level branches also store the failing box address so retry
logic can target the same box.

### Branch 1: active `last_cmd` is known

This is the normal full tool-change unload path.

```text
last = active physical slot
look up its box address and slot letter
wait for that box to leave PRELOADING
set that box to IDLE
choose unload velocity and temperature from the current material profile

if the local tool sensor reports filament present:
    ask the box to retrude from the printer/buffer side toward BUFFER
    if that succeeds:
        run a small printer-extruder reverse move to help unload
    else if the local tool sensor still reports filament present:
        report retrude sensor/path error and fail
    else:
        set the box back to IDLE and continue
    run an additional printer-extruder retract

ask the box to retrude the slot back to the MATERIAL sensor
if that succeeds:
    remember previous active slot
    clear the active filament position and last_cmd
    require the local tool sensor to be clear
    return success
else:
    enter the material-retrude recovery path
```

#### Active-slot material-retrude recovery

This recovery runs only after the final retrude-to-material command fails.

Operationally, the recovery has these externally relevant stages:

| Stage | Observable behavior / purpose |
|---|---|
| Stop on incompatible saved state | If the saved error state indicates a state/mode problem, recovery stops instead of continuing to move material. |
| Require a clear local path before material-side recovery | If the local tool sensor still sees filament at the start of this recovery, the flow reports a retrude sensor/path error. |
| Reset the affected box | The affected box is set to `IDLE` before recovery movements. This is important because a box left in print/feed/preload mode can reject or fight retrude movements. |
| Check buffer state | If the buffer appears active/full, the flow may run a bounded buffer-side recovery movement and a small printer-extruder nudge before retrying. |
| Loosen and retry from a known state | The material path may be loosened mechanically, then the box is set back to `IDLE`. |
| Bounded buffer-side retry | If the local tool sensor reports filament after the reset/loosen phase, the flow may retry retrude-toward-`BUFFER` up to three times, with printer-side unload assist between attempts. |
| Final material-side retry | The flow retries retrude-to-`MATERIAL`. It succeeds only if active-material state can be cleared as in the normal unload path. |

Important sensor interaction: in the active-slot branch, a successful final
`MATERIAL` retrude is **not** enough by itself. The local tool sensor must also
be clear at the end, otherwise the branch reports a retrude failure.

### Branch 2: no active material and local tool sensor is clear

This branch assumes there is no filament at the printer-side sensor, but there
may be material stranded inside connected boxes or the five-way path.

```text
for each connected box address 1..4:
    refresh connection and material sensor masks
    for each slot A..D:
        if that slot is present in both masks:
            wait for the box to leave PRELOADING
            set the box to IDLE
            retrude that slot to the MATERIAL sensor
            if any step fails:
                record the box address and fail
return success
```

The wrapper only unloads slots where **both** the connection/five-way sensor and
the material sensor are set. A material-only slot is left alone in this branch.

### Branch 3: no active material but local tool sensor sees filament

This branch tries to infer which connected box owns filament already in the
printer path.

Operational behavior:

- if the hotend cannot currently extrude, the flow starts heating toward the
  configured unload temperature before printer-side assist moves are needed;
- connected boxes are checked with live connection and material sensor masks;
- boxes with no connection/five-way sensor evidence are skipped;
- a candidate box is waited out of `PRELOADING` and then set to `IDLE` before
  shared-path retrude movements;
- the flow first tries to pull filament from the shared path toward `BUFFER`
  using the slot-0/all-slots style retrude request;
- after the shared-path step, the flow looks for a concrete slot that is present
  in both the connection and material sensor masks;
- the selected confirmed slot is assisted by printer-side unload motion and then
  retruded back to the `MATERIAL` sensor;
- if no connected box produces a slot confirmed by both sensors, the branch
  reports a sensor error and fails.

The important interoperability rule is that a slot must be supported by live
connection and material sensor evidence before it is treated as the owner of the
occupied path.

#### Initial BUFFER retrude failure in the inferred branch

When the shared `BUFFER` retrude fails, the wrapper uses live box masks to decide
whether it can clean up before stopping.

Externally relevant behavior:

- if the local tool sensor still sees filament after the failed shared-path
  retrude, the branch reports a retrude sensor/path error and fails;
- otherwise the affected box is set to `IDLE` before cleanup movements;
- slots that are present in both material and connection sensor masks may be
  retruded back to `MATERIAL` as cleanup;
- a material-only slot in this failure path is treated as a material/sensor
  inconsistency;
- the branch is still treated as failed because the shared `BUFFER` retrude
  already failed.

Even successful cleanup does not make the original unload operation successful.

### Unload failure outcomes

| Condition | Typical outcome |
|---|---|
| Box remains in PRELOADING or wait fails | Unload fails; address may be recorded in non-active branches. |
| Setting the box to IDLE fails | Unload fails. |
| BUFFER retrude fails while local tool sensor still sees filament | Retrude sensor/path error; unload fails. |
| Final MATERIAL retrude fails | Active branch enters recovery; inferred/no-sensor branches fail. |
| Final active-slot unload succeeds but local tool sensor still sees filament | Retrude error; unload fails. |
| Material sensor is set but connection sensor is not set in the initial-buffer failure path | Material/sensor inconsistency error for that slot; unload fails. |
| No connected box has usable connection/material sensor evidence while the local tool sensor sees filament | Sensor error; unload fails. |
| Saved state error appears during active-slot recovery | Recovery stops and unload fails. |

## Cutter path behavior

The cutter path is motion-led. A configured cut switch adds confirmation; without
it, the wrapper performs blind strokes and assumes success if the motion finishes.

### Homing and positioning assumptions

Before cutter motion, the wrapper checks whether X is homed. If X is not homed,
it homes X and Y, clears bed mesh in the main cutter path, and waits for motion.
Several cutter commands use the same practical assumption: X/Y must be
homed before cut, pre-cut, clean, or safe-position moves.

If `has_extrude_pos` is enabled, the cut flow first performs the wrapper's Z move
handling, switches to absolute positioning, moves to pre-cut X, then moves to
pre-cut X/Y. If `has_extrude_pos` is disabled, this pre-positioning behavior may not run.

If the hotend can extrude, the wrapper retracts a short length before cutting.
After blind-cut return, it performs a smaller additional retract.

### Position model

| Position / value | Role |
|---|---|
| `pre_cut_pos_x` | X used before cutter strokes. |
| `pre_cut_pos_y` | Y used as the return / pre-cut Y position. |
| `cut_pos_y` + `cut_pos_offset` | Requested cut Y. |
| `cut_pos_y_max` | Upper cap for real cut Y. |
| `cut_velocity` | Feedrate for cutter and pre-cut moves. |
| `cut_accel`, `cut_accel_to_decel` | Temporary acceleration limits during cutting. |
| `retrude_position_y` | Return position used by cut-position search. |

The real cut Y is computed conceptually as:

```text
real_cut_y = cut_pos_y + cut_pos_offset
if real_cut_y > cut_pos_y_max:
    real_cut_y = cut_pos_y_max
```

### Shared cutter flow

```text
save current acceleration limits
home X/Y if X is not homed
if local tool sensor reports no filament:
    log no-filament and return success without cutting
move to pre-cut path if configured
if hotend can extrude:
    retract before cutting
apply cutter acceleration limits

if no cut switch is configured:
    run blind-cut branch
else:
    run sensor-confirmed branch
```

### Blind-cut branch: no cut sensor

```text
repeat cut_run_count * 3 times:
    home X/Y if needed
    move to pre-cut X
    move to pre-cut Y
    move to real cut Y

home X/Y if needed
move back to pre-cut Y
wait for motion
if hotend can extrude:
    run small retract
wait for motion
report cutter return OK
restore saved acceleration limits
return success
```

Validation caution: this branch cannot prove that a blade moved, returned, or
cut filament. It only proves the wrapper completed the configured motion path.
Use conservative cut positions and validate mechanically before relying on blind
mode.

### Sensor-confirmed branch: cut switch configured

```text
home X/Y if needed
clear the sticky cut-trigger flag
for up to five attempts:
    repeat cut_run_count strokes:
        move to pre-cut Y
        move to pre-cut X
        move to real cut Y
        wait after each stroke move
    wait about one second
    if current cut switch state indicates return/success:
        report cutter return OK
        stop retrying
    move back to pre-cut Y
    report return failed
    move to the box extrude position

if the sticky cut-trigger flag was set at any time:
    clear it
    report cut sensor triggered
    restore saved acceleration limits
    return success
else:
    report cut sensor not triggered
    record cut error
    return failure
```

Important distinction: observed behavior checks both the current switch state
and a sticky "happened" flag. The current state can stop retries, but final
success depends on the sticky flag having been triggered.

### Acceleration and velocity side effects

| Operation | Side effect |
|---|---|
| Main cut flow | Saves current `max_accel` and `max_accel_to_decel`. |
| During cutting | Applies `cut_accel` and `cut_accel_to_decel`. |
| Successful blind cut | Restores saved acceleration limits. |
| Successful sensor-confirmed cut | Restores saved acceleration limits. |
| Failed sensor-confirmed cut | Acceleration restoration on failure should be validated on target firmware. |
| Cut-position search | Applies a very high temporary acceleration limit, then restores saved limits at the end. |
| Cutter moves | Use `cut_velocity` for pre-cut and cut moves. |

### Cut-position search behavior

The cut-position search macro is intended for calibration, not routine cutting.
It refuses to search when the local filament sensor reports filament present.
It homes the machine, moves to the configured pre-cut position, clears the sticky
cut flag, then scans Y in small increments from near the Y maximum minus
`cut_find_start_offset_y`. When the cut switch triggers, it saves the detected Y
as the configured cut position and asks the user to persist the box config.

Validation cautions:

- Run this only with the cutter sensor installed and behaving correctly.
- Confirm the machine can safely move near the configured Y maximum.
- Confirm `cut_pos_offset` and `cut_pos_y_max` after calibration; routine cuts
  use the capped real cut Y, not necessarily the raw found Y.

## Buffer interactions

The buffer state is most visible during loading, but unload recovery also uses
it.

| Context | Buffer use |
|---|---|
| Active-slot unload recovery | After final MATERIAL retrude fails, the wrapper queries buffer state. If the stored/visible buffer value is active/nonzero, it asks the box to advance through early extrusion stages, nudges the printer extruder, then queries buffer again before retrying unload. |
| Inferred unload branch | A shared slot-0 BUFFER retrude is used to pull filament back from the common path before selecting a concrete slot from sensor masks. |
| Loading/final verification | The wrapper repeatedly checks whether the buffer has become full enough after advancing material. |

Observed buffer interpretation remains limited:

| Value | Practical reading |
|---:|---|
| `0` | middle / not full |
| `>= 1` | full / active |
| unknown | response was not read or cache/status was not updated |

## Measuring-wheel and blockage behavior

The measuring wheel can be read or cleared through the diagnostic command with
`ACTION=GET` or `ACTION=CLEAN`. The wrapper uses wheel readings when extrusion is
associated with a target material slot and the extrusion is long enough to be
segmented or checked.

### When wheel checks are used

| Extrusion shape | Wheel behavior |
|---|---|
| No target slot is supplied | The wrapper extrudes directly; no wheel-based blockage check is tied to a box slot. |
| One velocity/percent profile, target slot supplied, and total length is at least twice `buffer_empty_len` | The wrapper splits the extrusion into flush segments and accumulates measuring-wheel deltas across all segments. |
| Multiple velocity/percent segments and target slot supplied | The wrapper accumulates commanded segment length and reads the wheel whenever pending commanded length reaches twice `buffer_empty_len`. |
| Short target-slot extrusion below twice `buffer_empty_len` | The wrapper uses direct extrusion without the segmented wheel check. |

### Segmented flush/check pseudocode

```text
read measuring wheel before extrusion
last_wheel = stored wheel value
cumulative_diff = 0
split requested length into configured flush segments

for each segment:
    move to box extrude position
    extrude the segment
    read measuring wheel again
    diff = current_wheel - last_wheel
    add diff to cumulative_diff
    last_wheel = current_wheel
    retract slightly
    if a box error appeared:
        if ending-material mode is active:
            clean nozzle and nudge extrusion, then continue
        else:
            fail

if cumulative_diff > diff_length:
    optionally report nozzle-blocked error
    fail
else:
    return success
```

For multi-percent extrusion, the same threshold is applied to each checked
wheel delta rather than to one cumulative value across the whole flush.

### Observed thresholds and defaults

| Setting | Observed default / range | Use |
|---|---:|---|
| `buffer_empty_len` | default `30`, range `0..60` | Wheel checks start after about twice this much commanded extrusion. |
| `diff_length` | default `-10`, range `-30..30` | If measured wheel delta is greater than this threshold, the wrapper treats it as nozzle/blockage failure. |
| `box_first_clean_length` | default `90`, range `0..600` | First or minimum flush segment length. |
| `box_need_clean_length` | default `80`, range `0..box_first_clean_length` | Normal repeated segment length. |
| `box_need_clean_length_max` | default `100`, range `0..600` | Cap/redistribution value used while building segment lengths. |

Threshold caution: observed compatibility behavior compares `measured_delta > diff_length`.
Because the default `diff_length` is negative, the sign and scale of the wheel
value must be validated on real hardware before changing this threshold.

### Measuring-wheel uncertainties

The public diagnostic shape is recoverable: `NUM` selects box address 1..4 and
`ACTION` is `GET` or `CLEAN`. Treat wheel telemetry as hardware-validated only
after confirming that `GET`, `CLEAN`, and stored wheel values move in the
expected direction on the target machine.

## Practical validation checklist

1. Confirm local tool sensor truth values with filament inserted and removed.
2. Confirm each box's material and connection masks for every used slot.
3. Test unload with a known active slot and verify the local tool sensor clears.
4. Test the no-active/no-tool-sensor cleanup branch only after confirming a slot
   is present in both material and connection masks.
5. Test cutter motion first without filament, then with filament, and confirm
   acceleration limits are restored after both success and failure.
6. If using blind cutter mode, manually inspect cut quality and cutter return.
7. Clear and read the measuring wheel, move filament, then verify the wheel value
   changes consistently before trusting blockage detection.
8. Validate `diff_length` with controlled good extrusion and controlled blocked
   or restricted extrusion on the target hardware.

## Known gaps and uncertainties

| Area | Gap |
|---|---|
| Exact firmware meaning of BUFFER vs MATERIAL retrude triggers | The wrapper-side behavior is recoverable, but low-level box firmware semantics are inferred from names and outcomes. |
| Buffer persistence | The buffer query is visible, but cache/status update behavior should be validated. |
| Measuring-wheel response decode | Public commands are clear, but exact response decoding should be validated. |
| Cut failure restoration | The sensor-confirmed failure path does not visibly restore acceleration limits before returning failure. Verify on hardware. |
| Local filament sensor polarity | The wrapper treats missing local sensor as present and treats sensor values `<= 1` as present. Confirm deployed Klipper sensor behavior. |
| Error key names | The broad failure phases are recoverable; exact user-facing error strings can vary through the wrapper error translation layer. |
