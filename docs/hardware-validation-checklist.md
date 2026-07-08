# Hardware validation checklist

## Scope

This appendix lists behavior that should be validated on the target machine when
bringing up the material box or writing an independent wrapper. It focuses on
observed device behavior, public commands, sensor meanings, and safe operation.

Most users should start with the main references:

- [`serial-protocol.md`](serial-protocol.md)
- [`configuration.md`](configuration.md)
- [`state-model.md`](state-model.md)
- [`sensors-and-hardware.md`](sensors-and-hardware.md)
- [`klipper-macros.md`](klipper-macros.md)
- [`errors-and-recovery.md`](errors-and-recovery.md)
- [`material-change-flow.md`](material-change-flow.md)
- [`auto-refill.md`](auto-refill.md)
- [`compatibility-caveats.md`](compatibility-caveats.md)

## Confidence legend

| Confidence | Meaning |
|---|---|
| High | Reflected by public commands, visible state, or simple protocol tables. |
| Medium | Strongly implied by observed behavior, but exact hardware detail should still be checked. |
| Low | Incomplete or hardware-specific; validate before relying on it. |

## Protocol validation items

| Area | Confidence | What to validate |
|---|---|---|
| Application request body | Medium | The application-level body is `ADDR/LENGTH/STATE/CMD/DATA`. If you do not use the existing serial transport, confirm any lower-layer head/checksum/trailer bytes. |
| Response trailer | Medium-low | Some responses appear to include a trailing byte. Confirm whether it is checksum, terminator, or transport metadata. |
| `GET_BOX_STATE` payload | Low | Capture responses across idle, preloading, printing, and error modes. Identify which data fields represent mode and environment values. |
| `GET_BUFFER_STATE` | Low | Compare the immediate command response with the physical buffer condition and any status field your wrapper exposes. |
| Measuring wheel | Low | Confirm `CLEAN`, then `GET`, then controlled movement. Verify reported values change plausibly. |
| `EXTRUDE_PROCESS` stages | Medium | Correlate stages `0`, `4`, `5`, `6`, and `7` with actual loading progress on the target hardware. |
| Reserved command ids | Medium | Treat undocumented ids as reserved unless hardware testing proves they are useful. |

## State and configuration validation items

| Area | Confidence | What to validate |
|---|---|---|
| Literal `"None"` vs missing/empty state | High | Do not collapse visible string values and absent values without testing status output. |
| Box mode refresh | Low | Query state while forcing known modes; confirm exposed mode values. |
| Active filament marker | Medium | Load and unload each slot and confirm the visible active-slot marker. |
| `has_extrude_pos = 0` | Medium | Test only if you intentionally run without full extrude/safe positions. Normal material-box operation should use `1`. |
| Runtime config saving of zero values | High | Prefer direct config-file edits for zero values. |
| `buffer_empty_len` limits | High | Config and UI metadata may expose different maxima; stay within the conservative range unless validated. |
| Optional config fields | Low | Fields such as machine flags or line-length metadata may be UI/profile metadata rather than required box behavior. |

## Macro and command validation items

| Area | Confidence | What to validate |
|---|---|---|
| `BOX_SEND_DATA` shape | High | It builds a normal application request; it is not arbitrary byte injection. |
| `BOX_EXTRUDE_PROCESS VELOCITY` | Medium | Public velocity input may not map to a wire velocity field. Use normal loading commands for production flows. |
| `BOX_MODIFY_TN_INNER_DATA` usefulness | High | Prefer real sensor queries instead of direct inner-state edits. |
| Resume guards | High | Confirm which commands no-op during resume and which remain active. |
| Direct versus workflow commands | High | Commands such as `BOX_CUT_MATERIAL` run larger motion/hardware flows. Review side effects before use. |
| Deferred recovery behavior | Medium | Force a controlled macro failure and compare retry versus clear behavior before relying on automation. |

## Material-flow validation items

| Area | Confidence | What to validate |
|---|---|---|
| Ending-material handling | Low | Simulate runout with known material. Confirm how remaining filament is purged and when replacement loading starts. |
| Slicer flush metadata | Low | Compare material changes with and without supported slicer/prime-tower metadata. |
| Measuring-wheel blockage threshold | Medium-low | Run controlled free-flow and restricted-flow tests before using blockage detection as a hard stop. |
| Extruder-side phase | Medium-low | Confirm whether your workflow needs a distinct printer-extruder-side verification step. |
| Cut failure handling | Low | Trigger a safe controlled cut failure and confirm acceleration/motion state is restored. |

## Auto-refill validation items

| Area | Confidence | What to validate |
|---|---|---|
| Auto-refill enable behavior | High | Toggle auto-refill, restart if needed, and confirm whether your chosen state persists. |
| Old-slot marking | Medium | After refill, inspect whether the old slot is marked unavailable and whether same-material groups refresh. |
| Concurrent ending-material flow | Medium | Confirm whether refill waits for ending-material purge before loading the replacement. |
| Candidate availability | High | Matching material identity is not enough; the replacement slot must also report live material presence. |

## Manual protocol checks

Run these with the printer in a safe state and with motion disabled or clear of
hardware where possible.

### 1. Basic communication

Command:

```gcode
BOX_GET_RFID ADDR=1 NUM=15
```

Expected application-level request body when using the documented command layer:

```text
01 04 ff 01 0f
```

Validate the response and, if not using the existing serial transport, any
transport-level framing around this body.

### 2. Sensor-bank checks

Commands:

```gcode
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=MATERIAL
BOX_GET_FILAMENT_SENSOR_STATE ADDR=1 POSITION=CONNECTIONS
```

Expected application-level request bodies:

```text
01 04 ff 07 00
01 04 ff 07 01
```

Validate that returned slot bits match inserted material and connection-path
state.

### 3. Box mode checks

Commands:

```gcode
BOX_GET_BOX_STATE ADDR=1
BOX_SET_BOX_MODE ADDR=1 MODE=IDLE NUM=0
BOX_GET_BOX_STATE ADDR=1
BOX_SET_BOX_MODE ADDR=1 MODE=PRINT NUM=A
BOX_GET_BOX_STATE ADDR=1
```

Validate which visible mode/status fields change.

### 4. Buffer checks

Command:

```gcode
BOX_GET_BUFFER_STATE ADDR=1
```

Run before and after a known loading/buffer-full condition. Validate returned
value, visible status, and physical sensor condition.

### 5. Measuring-wheel checks

Commands:

```gcode
BOX_MEASURING_WHEEL NUM=1 ACTION=CLEAN
# move or extrude material in a controlled way
BOX_MEASURING_WHEEL NUM=1 ACTION=GET
```

Validate that the returned value changes consistently with motion.

## Manual behavior checks

| Test | Goal |
|---|---|
| Connect/disconnect heartbeat | Confirm boxes transition between connected and disconnected status. |
| Slot insert/extract | Confirm slot events update visible material identity as expected. |
| RFID invalid/unknown value | Confirm how empty, unknown, and valid material identity are represented. |
| Remap | Confirm `BOX_MODIFY_TN T1A=T2C` redirects the intended logical material. |
| Power-loss restore | Confirm enable state, remap, and active material restore policy. |
| Cut sensor trigger | Confirm cutter switch behavior with and without `switch_pin`. |
| Filament runout | Confirm pause, ending-material purge, and refill behavior. |
| Flush blockage | Confirm nozzle-blockage handling and recovery workflow. |

## Documentation update rule

When behavior is verified on hardware:

1. update the specific reference document where the behavior belongs;
2. reduce or remove the corresponding uncertainty here;
3. include the validation method and hardware/firmware version if relevant.
