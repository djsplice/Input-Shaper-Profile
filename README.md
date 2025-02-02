 ## Input Shaper Profiles
Wouldn't it be nice if we didn't have to make print quality tradeoffs based on global toolhead acceleration values? Wouldn't it also be nice to automatically apply the best input shaper configuration to meet print speed or quality goals (eg: Draft Mode, High Quality Mode).

Let me introduce you to what I'm calling Input Shaper Profiles, built using:
- Per Axis Accelerations enabled in `limited_corexy` kinematics
- Input Shaper recommendations stored in variables in the `printer.cfg`
- Custom gcode macro that parses the slicers `Process` profile name used to slice the model to determine what Input Shpaer profile to apply

## Tuning for High Quality prints and High Performance prints
Choosing input shaper parameters has always been rather heavy handed and not very flexible. You usually ended up choosing an input shaper config that was 'good enough' for general purpose printing, the trade off being at the expense of speed or quality (or both!). This 'One Size Fits All' approach is problematic because the Y axis on most CoreXY machines is heavier needing much lower accelerations than the X axis to stay within bounds of Input Shapers ability to most effectively correct vibrations.
### Problem 1 - Choosing an appropriate `max_accel` value
Even though resonance tests would give you per-axis recommendations for both shaper type and suggested `max_accel`, there was no way to specify a `max_accel` per axis. This would leave you in a bit of a bind... 

Your choices were either to:
1. Slow the overall top end to match the `max_accel` recommendations of the slowest axis (usually Y, usually very slow)
2. Choose the higher overall `max_accel` recommendations and sacrifice print quality on the slower axis (usually Y) resulting in more ringing/ghosting
### Problem 2 - Choosing the best general purpose Input Shaper algorithms
ShakeTune will typically provide 2 sets of recommendations and you had to choose to optimize for either:
1. **Performance** for higher acceleration speeds at the expense of more smoothing and higher remaining vibrations
2. **Quality** trading print speed (lower accelerations) for improved print quality by reducing the impact of Input Shaper smoothing, and lower remaining vibration percentage
### Problem 3 - Changing Input Shaper configuration based on print goals
While you could define Quality and Performance `[input shaper]` configurations in your `printer.cfg` you would have to remember to go into the config and enable the desired profile at the start of every print. This is way too much cognitive load for me. I would always forget which profile the printer was currently using, and prints would come out with less quality than was actually achievable... 
## Implementation details

### Prerequisites:
- Kalico (Bleeding Edge v2 if you want to use Extruder PA Sync - recommended)
  - `limited_corexy` kinematics supports per-axis acceleration config  
	- Patch to `toolhead.py` to enable setting of per-axis accels using `SET_VELOCITY_LIMIT` macro
- Input Shaper recommendations (I use ShakeTune) for Quality and Speed 
- Ability to pass slicer profile name into `START_PRINT` Gcode macro
- A `saved_variables` configured for storing Input Shaper profile data.
  See: [Configuration reference - Klipper documentation](https://www.klipper3d.org/Config_Reference.html?h=macro#save_variables)


### Use [Extruder Pressure Advance / Input Shaper Sync](https://github.com/KalicoCrew/kalico/blob/bleeding-edge-v2/docs/Bleeding_Edge.md#extruder-pa-synchronization-with-input-shaping)
A Kalico bleeding edge feature, this aims to reduce artifacts by synchronizing filament extrusion with modified toolhead motion due to effects of Input Shaping.

### Use`limited_corexy` kinematics in Kalico
Kalico allows you to set a `max_acceleration` for each axis independently using the `limited_corexy` kinematics setting in `printer.cfg`.  Now we can set acceleration limits that align to ShakeTune recommendations and no longer need to choose between higher of the 2 recommended input shaper accelerations and diminish quality on one axis!

### Create and Test  'Quality' and 'Performance' Input Shaper configurations
Run ShakeTune `AXES_SHAPER_CALIBRATION` process, and test recommended Quality and Performance settings.

Once validated, enter the values into the profiles defined in the `saved_variables.cfg` file.

### Integrate the `SET_SHAPER_PROFILE` macro into your `START_PRINT` routine
Configure your slicer to expose print profile names, and pass that name when you call `START_PRINT`

In Orca Slicer, I had to update the 'Template Custom G-Code' section in my Printer profile, and make the `print_preset` variable available for use.

<img src="/img/20250124093425.png" width="400">

Update the 'Start Machine G-Code' to pass along the `print_preset` data:
```
START_PRINT EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single] PROFILE="[print_preset]"
```
### Update your `START_PRINT` routine to call `SET_SHAPER_PROFILE`
This is where you can specify what Input Shaper profile to use based on key words in your Slicer Profile. Using the logic in the macro below

* Profile Name: `0.2 - Draft` would use the 'performance' Input Shaper Profile
* Profile Name: `0.2 - HQ` would use the 'quality' Input Shaper Profile
* Profile Name: `0.2 - Test` would not mach any custom profiles, and would load the base parameters from `printer.cfg`, that is, the values defined in `[printer]` and `[input_shaper]` config sections.

```
  # Set Input Shaper configuration based on Slicer Profile
  {% if 'Draft' in PROFILE %}
    {action_respond_info("Draft profile detected - Removing Y Acceleration Limits and applying performance input shaper settings .")}
    SET_SHAPER_PROFILE PROFILE=performance
  {% else %}
    {% if 'HQ' in PROFILE %}
      {action_respond_info("Configuring for Quality acceleration speeds and input shaper settings.")}
      SET_SHAPER_PROFILE PROFILE=quality
    {% endif %} 
    SET_SHAPER_PROFILE PROFILE=load-printer-default
  {% endif %} 
```
> [!Default Behavior]
> If a profile name isn't passed, or there isn't a match, no changes to base input shaper or max_accel configuration will be made.
> 
> Make sure to specify sane values in your `printer.cfg` as they are they will be used by default!

### Update your Slicer Print Profile
Make sure it's got the key word HQ (for quality input shaper profile) or Draft (for performance input shaper profile) in the Profile Name.

Example - HQ for Quality Input Shaper profile:

<img src="/img/20250124094249.png" width="400">

Example - Draft for Performance Input Shaper profile:

<img src="/img/20250124094514.png" width="400">

### Set max accel values in your slicer profile
Use the highest acceleration values based on input shaper recommendations for Inner/Outer walls acceleration limits. This will set upper limits for the entire print, `SET_SHAPER_PROFILE` will set additional limits on each axis as defined in performance or quality variables set in the `saved_variables.cfg` file. 

As an example, if your ShakeTune Performance profile recommended a max accel of 13300 for X axis and a max accel of 5120 for the Y axis - use the value of 13300 (OR LESS) as a starting point. You can always increase or decrease this value as you see fit, you can also use this value in other features outside of Inner/Outer walls.

As an example, even though my shaper recommendations say I can use a max accel of 13300, for my quality profile, I limit wall accelerations to 6000 to further reduce artifacts.

<img src="/img/20250124101355.png" width="400">

### Putting it all together
If you have the following configuration set in the `saved_variables.cfg` file as such:
```
quality = {'shaper_type_x': 'smooth_ei', 'smoother_freq_x': 67.0, 'shaper_type_y': 'smooth_ei', 'smoother_freq_y': 41.2, 'max_y_accel': 5320}
performance = {'shaper_x': 'mzv', 'shaper_freq_x': 68.4, 'damping_ratio_x': 0.068, 'shaper_y': 'mzv', 'shaper_freq_y': 42.6, 'damping_ratio_y': 0.078}
```
Using the slicer configuration above as an example:

The name of the profile is `0.20mm - HQ` so it will match the `HQ` filter in the `START_PRINT` macro and load the `Quality` Input Shaper Profile.

The `0.20mm - HQ` Slicer profile will set overall limits:
- Normal Printing - 15000mm/s
- Inner/Outer walls - 6000mm/s
- Travel - 25000mm/s
- etc..

The Quality Input shaper profile will apply additional limits:
- Set `smooth_ei` shapers for x and y
- Limit `max_y_accel` to 5320mm/sec
