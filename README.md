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
T{next_extruder} LEN={if purge_in_prime_tower}0{else}{second_flush_volume/2.405279844}{endif}  RETRACT={new_retract_length_toolchange} PRERETRACT={old_retract_length_toolchange}
```
6. Under *Printer-settings -> Multimaterial* disable **Enable filament ramming** and if you want to purge in the hopper then also disable **purge in prime tower**. Note that disabling the latter won't allow to purge into objects or infill (in general, not only for this mod).
7. I advise you to use a small **prime tower** anyway and also a few skirts around the first printed object to get a stable pressure shortly before the actual print. This is not necessary but will improve quality.

8. Adjust the config for your needs. My mod provides these custom configurations
```
[gcode_macro pre_cut_box]
variable_pre_cut_retrusion: 37 # mm, heatbreak filament length, so there is approx 65-37 = 28 mm filament left in the nozzle after cutting
variable_pre_cut_retrusion_speed: 33 
variable_max_wait_for_preload: 30 # sec, timout for preload
variable_extrude_before_purge: 34 # mm, how much to extrude before moving to the purge tower. We need some pressure there to not print have large holes in the purge tower
variable_purge_speed: 15 # volumetric speed mm3/s
```
Most of it is self explanatory but `variable_max_wait_for_preload` is just a timeout for a very special case that usually never happens. You can just ignore it. If it interests you, check my blog.

Note, the settings `[box]` have the same meaning as without the mod.

# Contribution/Issues

If you want to contribute please make a pull request or if you have questions about reverse engineering their binary you can contact me. If you find an issue, please report it and i may fix it.

# Explanation

See [creality-k1-improved-purging](https://frederickalt.github.io/blog/creality-k1-improved-purging/)
