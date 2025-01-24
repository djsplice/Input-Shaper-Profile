## What is this?
A macro and supporting configuration to dynamically select Input Shaper configuration and per-axis acceleration at print start time.
### Prerequisites:
- Kalico (Bleeding Edge v2 if you want to use Extruder PA Sync - recommended)
	- Patch to `toolhead.py` to enable setting of per-axis accels using `SET_VELOCITY_LIMIT` macro
- Ability to pass slicer profile name into Gcode macro
- Input Shaper recommendations (I use ShakeTune)
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
### Use [Extruder Pressure Advance / Input Shaper Sync](https://github.com/KalicoCrew/kalico/blob/bleeding-edge-v2/docs/Bleeding_Edge.md#extruder-pa-synchronization-with-input-shaping)
A Kalico bleeding edge feature, this aims to reduce artifacts by synchronizing filament extrusion with modified toolhead motion due to effects of Input Shaping.
### Use`limited_corexy` kinematics in Kalico
Kalico allows you to set a `max_acceleration` for each axis independently using the `limited_corexy` kinematics setting in `printer.cfg`.  Now we can set acceleration limits that align to ShakeTune recommendations and no longer need to choose between higher of the 2 recommended input shaper accelerations and diminish quality on one axis!
### Create and Test  'Quality' and 'Performance' Input Shaper configurations
Run ShakeTune `AXES_SHAPER_CALIBRATION` process, and test recommended Quality and Performance settings.

Once validated, enter the values into the profiles defined in`SHAPER_PROFILES` gcode macro.
### Integrate the `SET_SHAPER_PROFILE` macro into your `START_PRINT` routine
Configure your slicer to expose print profile names, and pass that name when you call `START_PRINT`

In Orca Slicer, I had to update the 'Template Custom G-Code' section in my Printer profile, and make the `print_preset` variable available for use.
![[Pasted image 20250124093425.png]]
![]("./imgPasted image 20250124093425.png")
Update the 'Start Machine G-Code' to pass along the `print_preset` data:
`START_PRINT EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single] PROFILE="[print_preset]"`
### Update your `START_PRINT` routine to call `SET_SHAPER_PROFILE`
```
  # Set Input Shaper configuration based on Slicer Profile
  {% if 'Draft' in PROFILE %}
    {action_respond_info("Draft profile detected - Removing Y Acceleration Limits and applying performance input shaper settings .")}
    SET_SHAPER_PROFILE PROFILE=performance
  {% else %}
    {action_respond_info("Configuring for Quality acceleration speeds and input shaper settings.")}
    SET_SHAPER_PROFILE PROFILE=quality
  {% endif %} 
```
> [!Default Behavior]
> If a profile name isn't passed, or there isn't a match, no changes to base input shaper or max_accel configuration will be made.
> 
> Make sure to specify sane values in your `printer.cfg` as they are they will be used by default!

### Update your Slicer Print Profile
Make sure it's got the key word HQ (for quality input shaper profile) or Draft (for performance input shaper profile) in the Profile Name.

Example - HQ for Quality Input Shaper profile
![[Pasted image 20250124094249.png|400]]
Example - Draft for Performance Input Shaper profile
![[Pasted image 20250124094514.png|400]]
### Set max accel values in your slicer profile
Use the highest acceleration values based on input shaper recommendations for Inner/Outer walls acceleration limits. This will set upper limits for the entire print, `SET_SHAPER_PROFILE` will set additional limits on each axis as defined in performance or quality variables set in the `SHAPER_PROFILE` macro. 

As an example, if your ShakeTune Performance profile recommended a max accel of 13300 for X axis and a max accel of 5120 for the Y axis - use the value of 13300 (OR LESS) as a starting point. You can always increase or decrease this value as you see fit, you can also use this value in other features outside of Inner/Outer walls.

As an example, even though my shaper recommendations say I can use a max accel of 13300, for my quality profile, I limit wall accelerations to 6000 to further reduce artifacts.

![[Pasted image 20250124101355.png|400]]
### Putting it all together
If you have the following Quality configuration set in the `SHAPER_PROFILE` macro as such:
```
variable_quality:                             #
  {                                  #
    'shaper_type_x': 'smooth_ei',    #
    'smoother_freq_x': 67.0,         #
    'shaper_type_y': 'smooth_ei',    #
    'smoother_freq_y': 41.2,         #
    'max_y_accel': 5320,             #
  }
```
The resultant behavior would be to enforce all speed limits based on slicer config above:
- Normal Printing - 15000mm/s
- Inner/Outer walls - 6000mm/s
- Travel - 25000mm/s
- etc..
Per axis limits would then be applied to further limit accelerations based on `max_y_accel` value defined in the `quality` profile
* Limit Y axis movements to 5320mm/s
