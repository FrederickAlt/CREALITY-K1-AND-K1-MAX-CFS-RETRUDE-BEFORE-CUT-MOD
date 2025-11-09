This Repository implements a pre-cut retrusion for clreality k1 and k1 max. I have reverse engineered their cythonized binary with the source `box_wrapper.py` to be able to make this mod. The mod was made in conjunction with **Orca Slicer**. If you use a different slicer you might need to figure out how to configure thinks there.

# Instructions

1. Backup you **box.cfg** file.
2. Replace the **box.cfg** file with the provided one.
3. From you backup **box.cfg** copy everything in the `[box]` section and replace this in the provided **box.cfg**
4. In **Orca Slicer** go to *Printer-settings -> Machine G-code -> Machine start G-code* and **append**
```
T{initial_extruder} LEN=40 RETRACT=0 PRERETRACT=0
```
5. Set *Change filament G-code* to 
```
T{next_extruder} LEN={if purge_in_prime_tower}0{else}{second_flush_volume/2,405279844}{endif} RETRACT={new_retract_length_toolchange} PRERETRACT={old_retract_length_toolchange} {endif}
```
6. Under *Printer-settings -> Multimaterial* disable **Enable filament ramming** and if you want to purge in the hopper then also disable **purge in prime tower**. Note that disabling the latter won't allow to purge into objects or infill (in general, not only for this mod).
7. I advise you to use a small **prime tower** anyway and also a few skirts around the first printed object to get a stable pressure shortly before the actual print. This is not necessary but will improve quality.

# Contribution/Issues

If you want to contribute or have questions about reverse engineering their binary you can contact me. If you find an issue, please report it and i may fix it.

# Explanation

See https://frederickalt.github.io/

# Disclaimer