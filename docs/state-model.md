# Box state model

## Scope

This document explains the observable state model used by the material-box
wrapper interface. It is written for readers who do **not** have access to source
code and for authors of independent wrappers.

The wrapper state is best understood as several related state surfaces:

1. box connection state;
2. box operating mode;
3. per-slot material identity;
4. sensor and buffer state;
5. virtual-to-physical tool mapping;
6. active loaded material;
7. persisted/remap/resume state.

The lower-level serial frame and bit-mask details are documented in
[`serial-protocol.md`](serial-protocol.md). Configuration defaults and bounds are
documented in [`configuration.md`](configuration.md). Exact persisted JSON shape,
restore/cleanup lifecycle, and remap-mirror limits are documented in
[`persistence-reference.md`](persistence-reference.md). Error-specific recovery is
documented in [`errors-and-recovery.md`](errors-and-recovery.md).

## Conceptual entities

### Physical boxes

A system can have up to four physical material boxes:

| Box | Address |
|---|---:|
| `T1` | `1` |
| `T2` | `2` |
| `T3` | `3` |
| `T4` | `4` |

Each box has four material slots:

| Slot | Bit mask |
|---|---:|
| `A` | `0x01` |
| `B` | `0x02` |
| `C` | `0x04` |
| `D` | `0x08` |

A physical material slot is written as:

```text
T<box><slot>
```

Examples:

```text
T1A = box 1, slot A
T2D = box 2, slot D
T4C = box 4, slot C
```

### Virtual tools

The wrapper also exposes virtual tools:

```text
T0, T1, T2, ..., T15
```

These are user/G-code-facing aliases. They do not have to stay permanently bound
to the original physical slot because the wrapper can remap them, for example
during auto-refill.

Conceptually:

```text
virtual tool -> default physical slot -> actual physical slot
```

Example default mapping:

```text
T0  -> T1A
T1  -> T1B
T15 -> T4D
```

If refill/remap changes `T1A` to `T2C`, then a future `T0` command may resolve to
`T2C` even though `T0` originally pointed at `T1A`.

### Active material

The active material is the physical slot currently believed to be loaded into the
printer path. This is separate from slot availability and separate from virtual
mapping.

## State surface 1: box connection

Each physical box has a connection state.

```text
unknown/not seen -> connected -> disconnected
```

Common visible values:

| State value | Meaning |
|---|---|
| `"None"` or unset/unknown | Initial or unknown connection state. |
| `connect` | The wrapper believes the box is connected. |
| `disconnect` | The wrapper believes the box is disconnected. |

Observed transition rules:

```text
if online discovery and serial queries succeed:
    mark the box connected

if repeated heartbeat/query failures occur:
    mark the box disconnected

if at least one box is connected:
    aggregate box state = connected
else:
    aggregate box state = disconnected
```

A direct `BOX_GET_BOX_STATE` query can prove communication with a box, but status
connection state may also depend on heartbeat/discovery behavior.

## State surface 2: box operating mode

Each physical box can report or be assigned a mode.

| Mode code | Name | Meaning |
|---:|---|---|
| `0` | `IDLE` | Box is idle. |
| `1` | `PRELOADING` | Box is preloading material. |
| `2` | `PRINTING` | Box is in print/feed mode. |
| `3` | `WRAPPERING` | Observed mode name; exact firmware meaning should be validated if used. |
| `4` | `ERR` / `ERROR` | Box reports an error mode. |
| `5` | `TEST` | Box is in test mode. |

Common mode transitions:

```text
on startup/connect:
    query box state
    if mode is PRELOADING:
        set mode to IDLE before normal work

before feeding material for print:
    set mode PRINT for the selected slot

before retrude/recovery/manual actions:
    set mode IDLE

while waiting for preloading to finish:
    poll until mode is not PRELOADING or a timeout policy stops the wait
```

Commanded `PRINT`/`IDLE` values and reported mode names are separate concepts;
see [`serial-protocol.md`](serial-protocol.md#set_box_mode-data).

## State surface 3: per-slot material identity

Each physical slot can have material identity data from RFID, saved state,
insert/extract events, and sensor information.

Common identity values:

| Value | Meaning |
|---|---|
| `"-1"` | Uninitialized saved value. |
| `"none"` | Slot is explicitly empty / no material identity. |
| `"unknown"` | Material may be present, but identity is unknown. |
| RFID/material string | Slot has identified material data. |

When a valid material identity is known, the wrapper can derive or store:

| Field | Purpose |
|---|---|
| vendor/RFID value | Raw or display material identity. |
| material type | Material category used for matching/replacement. |
| color value | Color identifier used for matching and flush estimates. |
| remaining length | Remaining material estimate. |

### Slot events

The box can report four per-slot update events in A/B/C/D order:

| Event code | Meaning |
|---:|---|
| `0` | unchanged |
| `1` | insert |
| `2` | extract |
| `3` | completed |

High-level behavior:

```text
on extract:
    mark slot empty/unavailable

on insert/completed:
    refresh or restore material identity according to print/preload state
    update same-material groups when identity changes
```

The practical rule for independent wrappers: do not rely on cached identity alone
when selecting material. Refresh RFID/material identity and sensor state when the
physical spool may have changed.

## State surface 4: sensors and buffer

Box-side slot sensors are represented as four-bit masks.

| Sensor mask | Meaning |
|---|---|
| material mask | Which slots have material present at the material sensor. |
| connection mask | Which slots are detected at the connection/five-way sensor. |

Bit layout:

```text
bit:   7 6 5 4 3 2 1 0
       - - - - D C B A
mask:          8 4 2 1
```

Examples:

| Mask | Meaning |
|---:|---|
| `0x00` | no slots detected |
| `0x01` | slot A detected |
| `0x03` | slots A and B detected |
| `0x08` | slot D detected |
| `0x0f` | all slots detected |

The buffer state is simpler:

| Buffer value | Meaning |
|---:|---|
| `0` | middle / not full |
| `>= 1` | full / ready / active, depending on hardware wording |
| unknown | not read or not trusted |

Use live sensor queries for material availability, retrude decisions, refill
candidate selection, and load verification.

## State surface 5: virtual tool mapping

Virtual mapping lets the wrapper redirect a logical material to another physical
slot.

Initial identity mapping:

```text
T0  -> T1A -> T1A
T1  -> T1B -> T1B
...
T15 -> T4D -> T4D
```

Conceptually:

```text
logical_slot = default slot for virtual T command
actual_slot = current remap for logical_slot
use actual_slot
```

Mapping can change through:

| Mechanism | Effect |
|---|---|
| `BOX_MODIFY_TN` | Directly changes the remap table. |
| auto-refill | Redirects the old logical material to a compatible replacement slot. |
| power-loss restore | Restores saved remap state from `tn_data.json`. |
| end-print cleanup | May reset mappings to identity. |

This mapping state is independent of slot identity. A slot may remain mapped even
if its material later becomes empty or unknown.

### Macro visibility of remaps

The wrapper-managed `T*` material-change path reads the current remap table. A
plain Klipper macro that does not use that path and calls lower-level commands must
resolve remaps itself.

A config-only macro can:

- assume identity mapping;
- mirror visible `BOX_MODIFY_TN ...` commands;
- require the operator to reissue or reset mappings after restore/recovery.

It cannot reliably see mappings restored directly from `tn_data.json` unless it
has its own JSON-reading mechanism.

## State surface 6: active material

The active material tracks which physical slot is currently loaded into the
printer path.

```text
no active material
    -> successful load of target slot
active material = target slot
    -> successful unload/retrude
no active material
```

Common transitions:

```text
on successful load:
    mark target slot active
    mark global filament available

on successful unload:
    remember previous active slot if needed for recovery/refill
    clear active slot

on runout/refill:
    preserve enough active-slot context to choose a replacement
```

Active material matters because:

- if the requested slot is already active, a wrapper can use a fast path;
- if a different material is active, the wrapper must cut/retrude/load/flush;
- if no material is active, the wrapper can skip some unload work.

## State surface 7: persisted and resume state

The wrapper persists selected state to survive restart or power loss. This
section summarizes the state effect; see
[`persistence-reference.md`](persistence-reference.md) for the exact
`creality/userdata/box/tn_data.json` interoperability shape and cautions.

Persisted sections:

| Section | Survives restart | Meaning |
|---|---|---|
| `base_data` | yes | Saved slot identity: vendor/RFID, remaining length, color, material type. |
| `enable` | transient resume field | Saved material-box/CFS enable flag. |
| `tnn_map` | transient resume field | Saved logical-to-physical remap table. |
| `last_cmd` | transient resume field | Saved active physical slot name. |

Restore behavior:

```text
on power-loss restore:
    restore enable state if present
    restore remap table if present
    restore active-material hint if present
```

Cleanup behavior:

```text
on end-print/power-loss cleanup:
    remove transient enable/remap/active fields
    keep base_data identity cache
```

`base_data` is a long-lived material identity cache. `enable`, `tnn_map`, and
`last_cmd` are resume-oriented fields. They are old-wrapper interoperability
state, not box firmware state.

### Persisted mapping versus macro mirrors

Power-loss restore can load `tnn_map` directly from `tn_data.json` into wrapper
state without issuing `BOX_MODIFY_TN`.

A macro-side remap mirror can therefore be stale after:

- power-loss restore;
- restart with persisted resume state;
- manual edits to `tn_data.json`;
- any path that changes mapping without a visible `BOX_MODIFY_TN` command.

If a custom macro depends on a mirror, reissue the needed `BOX_MODIFY_TN ...`
commands after restore, or reset both wrapper and macro mapping to identity.

## Observable state summary

A consumer observing wrapper status should expect these broad groups.

| Group | What it answers |
|---|---|
| aggregate box state | Is any material box connected? |
| per-box connection state | Is this physical box connected? |
| per-box mode | What mode does this physical box report? |
| per-slot identity | What material does this slot contain? |
| per-slot remaining length | How much material is believed to remain? |
| material sensor mask | Which slots physically report material present? |
| connection sensor mask | Which slots are detected at the connection/five-way sensor? |
| virtual mapping | Which physical slot will a virtual tool command use? |
| active material | Which physical slot is currently loaded? |
| persisted state | Which identity/mapping/active values survive restart? |

## Default virtual mapping

| Virtual tool | Default physical slot |
|---|---|
| `T0` | `T1A` |
| `T1` | `T1B` |
| `T2` | `T1C` |
| `T3` | `T1D` |
| `T4` | `T2A` |
| `T5` | `T2B` |
| `T6` | `T2C` |
| `T7` | `T2D` |
| `T8` | `T3A` |
| `T9` | `T3B` |
| `T10` | `T3C` |
| `T11` | `T3D` |
| `T12` | `T4A` |
| `T13` | `T4B` |
| `T14` | `T4C` |
| `T15` | `T4D` |

## Derived same-material groups

The wrapper builds same-material groups from slot identity.

Conceptual shape:

```text
material_type + color_value -> [physical slots]
```

A slot participates only if:

```text
vendor/RFID is not empty/uninitialized
material_type is known
color_value is known
```

These groups are used by auto-refill to find replacement material with matching
material type and color.

## State updates from serial responses

| Response | State effect |
|---|---|
| `GET_RFID` | Updates material identity and may trigger same-material refresh. |
| `GET_REMAIN_LEN` | Updates remaining-length estimate for requested slots. |
| `MEASURING_WHEEL` | Updates or reports measuring-wheel value when supported. |
| `GET_FILAMENT_SENSOR_STATE` | Updates material or connection sensor mask. |
| `GET_VERSION_SN` | Updates visible version and serial-number fields. |
| `GET_BUFFER_STATE` | Reports buffer value; independent wrappers should decide explicitly whether and how to cache it. |
| `GET_BOX_STATE` | Feeds heartbeat/mode/environment state; exact payload layout should be hardware-validated. |
| `UPDATE_STATE` | Applies A/B/C/D slot events: unchanged, insert, extract, completed. |

## Known state-model uncertainties

| Area | Notes |
|---|---|
| `GET_BOX_STATE` mode update | The exact response byte for box mode should be validated on hardware. |
| Buffer state caching | Treat buffer command responses as immediate observations unless your wrapper owns a reliable cache. |
| Literal `"None"` vs absent values | Both can appear in status surfaces. External tools should not collapse them without testing. |
| Active filament marker | Confirm exposed active-slot status on the target machine after load and unload. |
| `WRAPPERING` mode spelling | Name is observed in status; exact firmware meaning is unclear. |
