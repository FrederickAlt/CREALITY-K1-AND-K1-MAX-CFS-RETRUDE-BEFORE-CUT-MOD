# Persistence reference

## Scope

This document describes the wrapper persistence file used for interoperability and the
runtime lifecycle around power-loss restore, cleanup, slot identity, remaps, and
macro-side mirrors.

It is written for readers using public behavior, deployed files, and device-visible
state. It uses prose, tables, and pseudocode only.

Related docs:

- [`state-model.md`](state-model.md) for the conceptual state machines.
- [`auto-refill.md`](auto-refill.md) for runout/remap behavior.
- [`box-cfg-changes.md`](box-cfg-changes.md) and
  [`box-explicit-wrapper-flush.md`](box-explicit-wrapper-flush.md) for custom
  config profiles that maintain a macro-side remap mirror.

## File location

The wrapper persists selected state in:

```text
creality/userdata/box/tn_data.json
```

The path is built under the wrapper's base directory. On a deployed
machine, treat it as the material-box userdata JSON file, not as a Klipper macro
variable store.

## What this means for a new wrapper

This JSON file is **wrapper state**, not material-box firmware state. The box does
not require this file to load, unload, cut, or report sensors. The old wrapper
uses it to remember material identity, remaps, enable state, and the last active
slot across restart or power loss.

For a new wrapper, decide explicitly whether you need compatibility with this
file:

| New-wrapper goal | Recommended handling |
|---|---|
| Independent replacement with its own UI/state | Ignore `tn_data.json` or import it once during migration, then store state in your own format. |
| Coexist with the old wrapper or Creality UI | Treat this format as an interoperability contract. Read it conservatively and avoid writing partial `tnn_map` objects. |
| Preserve power-loss restore/remap behavior from existing installs | Implement at least `enable`, complete `tnn_map`, and `last_cmd` restore semantics. |
| Preserve cached material identity | Read/write `base_data`, but refresh RFID and sensor state before trusting it as physical truth. |
| Implement only the box hardware protocol | Do not depend on this file; query sensors/RFID live and maintain active-slot state in your own wrapper. |

Important distinction:

```text
serial commands and sensors = box hardware contract
creality/userdata/box/tn_data.json = old-wrapper persistence contract
```

Deleting or changing `tn_data.json` can confuse restore/remap behavior, but it
does not directly move motors or change firmware state inside the box. Conversely,
the box can contain different physical material than the cached `base_data` says
until the wrapper refreshes sensors/RFID.

## Top-level JSON structure

The file is a JSON object. The long-lived section is `base_data`; the other
known top-level fields are resume-oriented fields written by specific wrapper
commands or material-load paths.

```json
{
  "base_data": {
    "T1": {
      "vender": ["-1", "-1", "-1", "-1"],
      "remain_len": ["-1", "-1", "-1", "-1"],
      "color_value": ["-1", "-1", "-1", "-1"],
      "material_type": ["-1", "-1", "-1", "-1"]
    },
    "T2": {
      "vender": ["-1", "-1", "-1", "-1"],
      "remain_len": ["-1", "-1", "-1", "-1"],
      "color_value": ["-1", "-1", "-1", "-1"],
      "material_type": ["-1", "-1", "-1", "-1"]
    },
    "T3": {
      "vender": ["-1", "-1", "-1", "-1"],
      "remain_len": ["-1", "-1", "-1", "-1"],
      "color_value": ["-1", "-1", "-1", "-1"],
      "material_type": ["-1", "-1", "-1", "-1"]
    },
    "T4": {
      "vender": ["-1", "-1", "-1", "-1"],
      "remain_len": ["-1", "-1", "-1", "-1"],
      "color_value": ["-1", "-1", "-1", "-1"],
      "material_type": ["-1", "-1", "-1", "-1"]
    }
  },
  "enable": 1,
  "tnn_map": {
    "T1A": "T1A",
    "T1B": "T1B",
    "T1C": "T1C",
    "T1D": "T1D",
    "T2A": "T2A",
    "T2B": "T2B",
    "T2C": "T2C",
    "T2D": "T2D",
    "T3A": "T3A",
    "T3B": "T3B",
    "T3C": "T3C",
    "T3D": "T3D",
    "T4A": "T4A",
    "T4B": "T4B",
    "T4C": "T4C",
    "T4D": "T4D"
  },
  "last_cmd": "T1A"
}
```

Only `base_data` is guaranteed to exist after first initialization. `enable`,
`tnn_map`, and `last_cmd` appear when the wrapper has written those resume
fields. Cleanup removes those resume fields and leaves `base_data` intact.

The compatibility key is spelled `vender`; docs preserve that spelling when
referring to the actual JSON field.

## Field reference

| Field | Shape | Writer/lifecycle | Meaning |
|---|---|---|---|
| `base_data` | Object keyed by `T1`..`T4` | Created on first initialization and updated when saved material identity fields change. Kept during cleanup. | Long-lived per-slot material identity cache. |
| `base_data.Tn.vender` | Four-element list in A/B/C/D order | Updated from valid saved identity changes. | RFID/vendor value, `unknown`, `none`, or `-1` depending on slot history. |
| `base_data.Tn.remain_len` | Four-element list in A/B/C/D order | Updated by remain-length/material identity paths. | Cached remaining filament length per slot. |
| `base_data.Tn.color_value` | Four-element list in A/B/C/D order | Updated from RFID/material identity paths. | Cached color identifier per slot. |
| `base_data.Tn.material_type` | Four-element list in A/B/C/D order | Updated from RFID/material identity paths. | Cached material type identifier per slot. |
| `enable` | Integer, normally `0` or `1` | Written by `BOX_ENABLE_CFS_PRINT`. Removed by cleanup. | Saved CFS/material-box print enable state to replay after power loss. |
| `tnn_map` | Object with physical-slot keys and physical-slot values | Written by `BOX_MODIFY_TN`; auto-refill uses the same remap mechanism. Removed by cleanup. | Saved logical-physical remap table. |
| `last_cmd` | Physical slot string such as `T1A` | Written when a box material load starts during printing or pause. Removed by cleanup. | Saved active physical slot for resume. |

Slot-list index order is always:

| Index | Slot |
|---:|---|
| `0` | `A` |
| `1` | `B` |
| `2` | `C` |
| `3` | `D` |

## `base_data` lifecycle

`base_data` is the persistent material identity cache. It is not merely a resume
field.

Startup behavior:

```text
if tn_data.json exists and contains base_data:
    load base_data into the wrapper's saved slot cache
else:
    create base_data for T1..T4 with four -1 values per saved field
    write tn_data.json
```

Update behavior:

```text
when a saved per-slot identity field changes:
    update live slot state
    if the new value is not the literal string "none":
        update base_data for that slot and field
        rewrite tn_data.json with the new base_data
```

Implications:

- `base_data` survives normal end-print/power-loss cleanup.
- A live slot can be marked empty as `none` without necessarily overwriting the
  saved cache with `none` in every path.
- External tools should treat `base_data` as a cache that helps restore identity,
  not as a perfect real-time sensor view.
- Use live wrapper status and refreshed RFID/sensor commands when deciding what
  is physically inserted now.

## `enable` lifecycle

`enable` is written by:

```gcode
BOX_ENABLE_CFS_PRINT ENABLE=<0|1>
```

Observed behavior:

| Saved value | Wrapper action at command time | Restore behavior |
|---:|---|---|
| `0` | Stores disabled-box-print state and enables the local filament sensor. | Replays `BOX_ENABLE_CFS_PRINT ENABLE=0`. |
| `1` | Stores enabled-box-print state and disables the local filament sensor. | Replays `BOX_ENABLE_CFS_PRINT ENABLE=1`. |

Power-loss restore only replays the command when the saved value exists and is
`<= 1`. The intended values are `0` and `1`.

Cleanup removes `enable` from `tn_data.json`.

## `tnn_map` lifecycle

`Tnn_map` is the mutable remap table used by the wrapper's own material-change
entrypoint.

Initial in-memory mapping is identity:

| Key | Default value |
|---|---|
| `T1A` | `T1A` |
| `T1B` | `T1B` |
| `...` | `...` |
| `T4D` | `T4D` |

Manual remap command:

```gcode
BOX_MODIFY_TN T1A=T2C
```

Conceptual command behavior:

```text
for each known physical-slot key:
    if the command includes a value for that key:
        update the in-memory remap table
write the whole remap table to tn_data.json as tnn_map
```

Auto-refill can also change this mapping so the old logical material points to a
replacement physical slot. For example:

```text
before: T1A -> T1A
after:  T1A -> T2C
```

Restore behavior:

```text
if tn_data.json contains a non-empty tnn_map:
    for every known physical-slot key:
        replace the in-memory mapping with the persisted value
```

Important constraints:

- Persist the complete 16-key map, not a partial object. Restore behavior expects
  each known key to exist when `tnn_map` is non-empty.
- Cleanup removes `tnn_map` from `tn_data.json`.
- End-print cleanup also resets the live mapping to identity in observed
  lifecycle behavior.

## `last_cmd` lifecycle

`last_cmd` is the active physical slot that the wrapper believes is loaded into
the printer path.

During a box material load:

```text
if print state is printing or paused:
    write last_cmd=<target physical slot> to tn_data.json

if the load succeeds:
    set runtime last_cmd=<target physical slot>
    set the per-box active slot marker for that slot
```

During retrude/unload:

```text
remember the current active slot for recovery/refill context
clear the per-box active slot marker
set runtime active material to none
```

Power-loss restore behavior:

```text
if tn_data.json contains last_cmd:
    set runtime last_cmd to that physical slot
```

Known limitation: restore sets the runtime active-material field, but may not
restore every exposed active-slot status surface. Custom macros that read only
per-box `filament` status may therefore miss restored active material until state
is refreshed.

Cleanup removes `last_cmd` from `tn_data.json`.

## Cleanup lifecycle

The cleanup path is used by end-print/power-loss cleanup behavior.

Conceptual behavior:

```text
load tn_data.json
remove enable if present
remove tnn_map if present and non-empty
remove last_cmd if present
write tn_data.json
keep base_data unchanged
```

If the file is missing, unreadable, or empty, cleanup logs/returns without
creating a replacement file.

## `BOX_POWER_LOSS_RESTORE` versus `BOX_MODIFY_TN`

These commands affect mapping differently.

| Command | Reads `tn_data.json`? | Writes `tn_data.json`? | Changes live `Tnn_map`? | Emits/uses `BOX_MODIFY_TN`? | Also restores `enable`/`last_cmd`? |
|---|---:|---:|---:|---:|---:|
| `BOX_MODIFY_TN ...` | No | Yes, writes `tnn_map` | Yes, from command parameters | It is the public remap command | No |
| `BOX_POWER_LOSS_RESTORE` | Yes | No | Yes, from persisted `tnn_map` when present | No; it copies persisted mapping directly | Yes |

Operationally:

```text
BOX_MODIFY_TN:
    command parameters are authoritative
    update live mapping
    persist the resulting mapping

BOX_POWER_LOSS_RESTORE:
    persisted JSON is authoritative
    optionally replay BOX_ENABLE_CFS_PRINT
    copy persisted tnn_map into live mapping
    copy persisted last_cmd into runtime active-material state
```

This distinction matters for config-only macro mirrors.

## Macro-side remap mirror limitations

The wrapper's current `Tnn_map` is not exposed as a complete status table to
plain Klipper macros. Config-only macros also cannot read wrapper runtime fields or parse
`creality/userdata/box/tn_data.json` by themselves.

A custom config can maintain a mirror by wrapping or observing:

```gcode
BOX_MODIFY_TN T1A=T2C
```

That mirror is useful when custom macros do not use the wrapper's full `T*` path and
call lower-level commands with a physical `TNN` parameter. It is still only a
mirror of commands the macro layer can see.

| Situation | Mirror reliability | Why |
|---|---|---|
| Manual `BOX_MODIFY_TN ...` after config load | Usually reliable | The macro wrapper can update its mirror and then forward the command. |
| Auto-refill path that emits `BOX_MODIFY_TN ...` after config load | Usually reliable | The visible remap command updates both wrapper state and the mirror. |
| `BOX_POWER_LOSS_RESTORE` loads persisted `tnn_map` | Not reliable | Restore copies JSON state directly into wrapper memory without issuing `BOX_MODIFY_TN`. |
| Restart with persisted resume state | Not reliable until remirrored | Macro variables may reset to config defaults while the wrapper restores state from persisted data. |
| Manual edit of `tn_data.json` | Not reliable | Plain macros do not parse the JSON file. |
| Any path mutates `Tnn_map` without `BOX_MODIFY_TN` | Not reliable | The macro mirror only sees visible remap commands. |

Recommended operator policy for config-only mirrors:

```text
after restore/restart/manual JSON edits:
    either reissue all desired BOX_MODIFY_TN commands
    or deliberately reset the macro mirror and wrapper mapping to identity
before using custom lower-level swap macros
```

If exact remap state matters after restore, prefer the wrapper-managed full `T*`
material-change entrypoint because it reads the current mapping directly.

## Manual editing cautions

If you edit `tn_data.json` by hand:

- keep valid JSON syntax;
- keep all four `base_data` boxes and all four A/B/C/D entries per saved field;
- keep `tnn_map` complete if present, or remove the whole `tnn_map` field;
- use physical slot strings such as `T1A` for both `tnn_map` values and
  `last_cmd`;
- reissue/remirror `BOX_MODIFY_TN ...` commands before relying on config-only
  remap mirrors;
- refresh material identity and sensor state on hardware before trusting cached
  `base_data` values.

## Known uncertainties and risks

| Area | Notes |
|---|---|
| Atomicity | The file may be rewritten directly rather than through an atomic temporary-file swap. Avoid editing while the wrapper is running. |
| Partial `tnn_map` | Restore expects all known mapping keys if the object is non-empty. A partial hand-written map can fail restore. |
| Active marker restore | `BOX_POWER_LOSS_RESTORE` restores runtime `last_cmd` but may not restore every exposed active-slot marker. |
| Cached identity freshness | `base_data` is a cache and can differ from current physical spool state until sensors/RFID are refreshed. |
