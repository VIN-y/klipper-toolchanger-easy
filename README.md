# klipper-toolchanger Easy Install
## Please Read this before starting!
The purpose of (yet another) fork of the Viesturz/klipper-toolchanger repo is to attempt to simplify the installation for StealthChanger users.  This repo combines the configuration, macros, and Python into a single repo.  It also outlines patterns for overriding configuration to ease installation and allow easier updates. 

All of the documentation for StealthChanger still applies and should be what you're following (except the Installation portion) https://github.com/DraftShift/StealthChanger/wiki

**This assumes you have a WORKING PRINTER and configuration**

# Installation

To install this plugin, run the installation script using the following command over SSH. This script will download this GitHub repository to your RaspberryPi home directory, and symlink the files in the Klipper extra folder.  It will also create copies of the user editable configuration and populate some example tools.  

This script does the following:
- symlinks the needed Python files into `klipper/klippy/extras`
- symlinks the gcode macros to `~/printer_data/config/stealthchanger/toolchanger`
- copies some example tools into `~/printer_data/config/stealthchanger/tools`
- copies user editible config into `~/printer_data/config/stealthchanger`

Running `install.sh` multiple times is a safe opearation, provided you follow the instructions below about configuration.
```
wget -O - https://raw.githubusercontent.com/jwellman80/klipper-toolchanger-easy/refs/heads/main/install.sh | bash
```

Add the following to your `printer.cfg`
```
[include stealthchanger/toolchanger-include.cfg]
```

Optional, but recommended, add the following to your `moonraker.conf` to enable automatic updates:
```
[update_manager klipper-toolchanger-easy]
type: git_repo
channel: dev
path: ~/klipper-toolchanger-easy
origin: https://github.com/jwellman80/klipper-toolchanger-easy.git
managed_services: klipper
primary_branch: main
```

## Updates that add new files

Note that if an update has new klipper files, they **will not** be automatically installed into Klipper.
You will need to run the intall script manualy to add them:
```commandline
bash ~/klipper-toolchanger-easy/install.sh
```

# VERY BASIC CONFIGURATION
At a minimum you will need to: 
- Set the dock position for each tool in the `stealthchanger/tools/Tx.cfg` file
- look through the toolchanger-config.cfg and configure the following:
  - homing_override_config (if used, if not then comment it out in the `toolchanger-include.cfg`)
  - _CALIBRATION_SWITCH if using a calibration probe set its position here.  Also update the `tools_calibrate`
- once things are working well you should uncomment the `[delayed_gcode startup]` block so that the INITIALIZE_TOOLCHANGER is available when Klipper starts

Tool configs must be named Tx.cfg (T0.cfg, T1.cfg, etc)

Follow the examples in the `example_T0.cfg` and `example_T1.cfg`.  The main difference between the tools is that T0 has a z_offset and other tools will not.  T1 and up will have gcode_offsets set and T0 will not.  Other than that the configuration between T0 and other tools is very similar.  

When starting up you will need a tool on the shuttle or Klipper will shut down.  You should always home, Quad Gantry Level, and bed mesh with T0.  

Highly recommend that you DO NOT EDIT the files in `~/printer_data/config/stealthchanger/toolchanger`
If you need to modify or config something in those files use the `~/printer_data/config/stealthchanger/toolchanger-config.cfg` file to override the macro or the setting in the macro.  `toolchanger-config.cfg` has examples of how this is accomplished. 

# Print_Start Macro
Edit your existing print start macro to have the following: 

Top of `print_start` macro
```
    {% set TOOL = params.TOOL | default(-1)| int %}
    M117 Initializing...
    INITIALIZE_TOOLCHANGER
    STOP_TOOL_PROBE_CRASH_DETECTION
    G28
    T0
    QUAD_GANTRY_LEVEL
    G28 Z
```

Bottom of `print_start` macro:
```
    {% if TOOL >= 0 %}
        M104 T0 S0 ; shutdown T0.  If it's up first it will be heated below.
        T{params.TOOL}
        {% set initialToolTemp = 'T' ~ params.TOOL|string ~ '_TEMP' %}
        M117 Waiting on T{params.TOOL} S{params[initialToolTemp]}C
        M109 S{params[initialToolTemp]}
    {% else %}
        M109 S{EXTRUDER}
    {% endif %}

    START_TOOL_PROBE_CRASH_DETECTION
```

Follow the Slicer configuration in the DraftShift Design Wiki:
https://github.com/DraftShift/StealthChanger/wiki/Slicers
