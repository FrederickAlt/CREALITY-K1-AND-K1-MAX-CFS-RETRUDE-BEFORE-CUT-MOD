# Box wrapper documentation

This project documents observed material-box and wrapper behavior for
interoperability and safe operation with independently written software. It does
not contain vendor source code, decompiler output, circumvention tools,
access-control bypasses, or authentication secrets. Vendor/device names are used
only to identify compatibility.

These docs are for people who want to understand how to use the wrapper and how 
the CFS works including how to use the serial interface of it and its commands. 
Some behavior is hardware-specific or not fully validated; see
[`hardware-validation-checklist.md`](hardware-validation-checklist.md) for checks
that should be run on real machines.

Before running direct serial, cutter, extrusion, or diagnostic commands, review
the safety notes in [`klipper-macros.md`](klipper-macros.md),
[`sensors-and-hardware.md`](sensors-and-hardware.md), and
[`compatibility-caveats.md`](compatibility-caveats.md).

## Recommended reading order

1. [`serial-protocol.md`](serial-protocol.md)
2. [`configuration.md`](configuration.md)
3. [`state-model.md`](state-model.md)
4. [`sensors-and-hardware.md`](sensors-and-hardware.md)
5. [`klipper-macros.md`](klipper-macros.md)
6. [`errors-and-recovery.md`](errors-and-recovery.md)
7. [`material-change-flow.md`](material-change-flow.md)
8. [`auto-refill.md`](auto-refill.md)
9. [`hardware-validation-checklist.md`](hardware-validation-checklist.md)

## Documents

| Document | Outline |
|---|---|
| [`serial-protocol.md`](serial-protocol.md) | Application-level serial frame format, command ids, bit masks, response states, direct serial-style macros, and frame examples. |
| [`protocol-reference.md`](protocol-reference.md) | Device command inventory, payload layouts, response shapes, enums, and protocol caveats. |
| [`command-timeouts.md`](command-timeouts.md) | Observed command timeout values and timeout behavior. |
| [`configuration.md`](configuration.md) | `box.cfg` options, setup/calibration order, runtime config commands, units, defaults, and caveats. |
| [`runtime-reference.md`](runtime-reference.md) | Runtime names/paths, public/test G-code inventory, and side-effect matrix. |
| [`state-model.md`](state-model.md) | Box, slot, virtual mapping, active material, persistence, and observable state model. |
| [`persistence-reference.md`](persistence-reference.md) | `tn_data.json` interoperability shape, restore/cleanup behavior, and remap mirror limitations. |
| [`sensors-and-hardware.md`](sensors-and-hardware.md) | Serial bus, material sensors, five-way sensors, local filament sensor, buffer, cutter sensor, measuring wheel, and fans. |
| [`unload-cutter-sensor-reference.md`](unload-cutter-sensor-reference.md) | Unload/retrude decision tree, cutter behavior, measuring-wheel/blockage notes, and sensor interactions. |
| [`klipper-macros.md`](klipper-macros.md) | Public G-code command catalog with syntax, side effects, and risk notes. |
| [`errors-and-recovery.md`](errors-and-recovery.md) | Protocol errors, workflow errors, pause behavior, retry/clear behavior, and recovery paths. |
| [`compatibility-caveats.md`](compatibility-caveats.md) | Public compatibility caveats for independent wrappers and custom macros. |
| [`material-change-flow.md`](material-change-flow.md) | Full material-change flow: resolving tools, cutting, retruding, loading, flushing, restoring, prime-tower and resume branches. |
| [`extrude-process-stages.md`](extrude-process-stages.md) | Box loading stage reference for `EXTRUDE_PROCESS` and final verification. |
| [`load-retry-state-machine.md`](load-retry-state-machine.md) | Load failure recovery strategies and safe retry boundaries for independent wrappers. |
| [`auto-refill.md`](auto-refill.md) | Runout handling, same-material matching, replacement selection, slot remapping, ending-material flow, and troubleshooting. |
| [`hardware-validation-checklist.md`](hardware-validation-checklist.md) | Uncertain or hardware-specific behaviors that should be validated on real machines. |
| [`box-cfg-changes.md`](box-cfg-changes.md) | Explanation of the generated robust manual-purge `box.cfg`, what changed from the supplied profile, and how to validate it safely. |
| [`box-explicit-wrapper-flush.md`](box-explicit-wrapper-flush.md) | Explanation of `box-explicit-wrapper-flush.cfg`: explicit pre-cut retrude, cut, full retrude, wrapper load, controlled wrapper flush, remap mirror, and limitations. |

## Quick paths

| Goal | Start with |
|---|---|
| Implement or validate the box serial protocol | [`serial-protocol.md`](serial-protocol.md), then [`protocol-reference.md`](protocol-reference.md) and [`hardware-validation-checklist.md`](hardware-validation-checklist.md) |
| Edit or calibrate `box.cfg` | [`configuration.md`](configuration.md) |
| Understand status fields | [`state-model.md`](state-model.md) |
| Bring up sensors or cutter hardware | [`sensors-and-hardware.md`](sensors-and-hardware.md) |
| Find a G-code command and its parameters | [`klipper-macros.md`](klipper-macros.md) |
| Recover from a paused/error state | [`errors-and-recovery.md`](errors-and-recovery.md) |
| Understand a full material switch | [`material-change-flow.md`](material-change-flow.md) |
| Configure or debug automatic refill | [`auto-refill.md`](auto-refill.md) |
| Validate uncertain behavior on hardware | [`hardware-validation-checklist.md`](hardware-validation-checklist.md) |
| Build an independent wrapper | [`protocol-reference.md`](protocol-reference.md), [`material-change-flow.md`](material-change-flow.md), [`persistence-reference.md`](persistence-reference.md#what-this-means-for-a-new-wrapper), [`auto-refill.md`](auto-refill.md#what-this-means-for-a-new-wrapper), [`compatibility-caveats.md`](compatibility-caveats.md) |
| See the documentation dependency map | [`WIKI_PLAN.md`](WIKI_PLAN.md) |
