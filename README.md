# Shake-Tune-AWD-modification
an attempt to have clear IS testing for AWD printers



Shake&Tune-friendly AWD input-shaper test suite that:

runs resonance tests with REAR motors only,

then FRONT motors only,

then ALL motors,
…for X and Y, saving each run with a distinct name so you can compare them in Shake&Tune (or auto-plot via shell if you want).

It’s designed for CoreXY-AWD style rigs (e.g., stepper_x, stepper_x1, stepper_y, stepper_y1). If your names differ, just change the variables at the top.

1) Drop these into your printer.cfg
```
#####################################################################
# AWD Input Shaper Suite (REAR → FRONT → ALL) for CoreXY AWD rigs
# Requires: ADXL + TEST_RESONANCES + (optional) gcode_shell_command
#####################################################################

# -----------------------
# Tunables / Defaults
# -----------------------
[gcode_macro AWD_IS_SUITE]
description: Run Shake&Tune-style IS tests (Rear, Front, All) on X & Y
variable_front_steppers: "stepper_x,stepper_y"        # FRONT pair
variable_rear_steppers:  "stepper_x1,stepper_y1"      # REAR pair
variable_test_current:   0.80                          # A during test
variable_idle_current:   0.05                          # A for "disabled" group (keeps drivers awake)
variable_freq_start:     20
variable_freq_end:       150
variable_accel_per_hz:   150
variable_scv:            10                            # Square Corner Velocity during sweep (optional)
variable_travel_f:       6000
gcode:
  {% set FRONT = params.FRONT|default(front_steppers)|string %}
  {% set REAR  = params.REAR|default(rear_steppers)|string %}
  {% set ITEST = params.ITEST|default(test_current)|float %}
  {% set IIDLE = params.IIDLE|default(idle_current)|float %}
  {% set FSTART = params.FSTART|default(freq_start)|int %}
  {% set FEND   = params.FEND|default(freq_end)|int %}
  {% set APH    = params.ACCEL_PER_HZ|default(accel_per_hz)|int %}
  {% set SCV    = params.SCV|default(scv)|int %}
  {% set VF     = params.VF|default(travel_f)|int %}

  # Safety: Home first and move up a bit
  G28
  G1 Z10 F{VF}

  # Set SCV if available (ignore if your setup doesn't use it)
  {% if SCV %}
    SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY={SCV}
  {% endif %}

  # ---------- Phase 1: REAR ONLY ----------
  {action_respond_info("AWD-IS: Phase 1 - REAR only")}
  _AWD_SET_GROUP ACTIVE={REAR} INACTIVE={FRONT} ITEST={ITEST} IIDLE={IIDLE}
  TEST_RESONANCES AXIS=X FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=rear
  TEST_RESONANCES AXIS=Y FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=rear

  # ---------- Phase 2: FRONT ONLY ----------
  {action_respond_info("AWD-IS: Phase 2 - FRONT only")}
  _AWD_SET_GROUP ACTIVE={FRONT} INACTIVE={REAR} ITEST={ITEST} IIDLE={IIDLE}
  TEST_RESONANCES AXIS=X FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=front
  TEST_RESONANCES AXIS=Y FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=front

  # ---------- Phase 3: ALL MOTORS ----------
  {action_respond_info("AWD-IS: Phase 3 - ALL motors")}
  _AWD_ENABLE_ALL LIST={FRONT + "," + REAR} ITEST={ITEST}
  TEST_RESONANCES AXIS=X FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=all
  TEST_RESONANCES AXIS=Y FREQ_START={FSTART} FREQ_END={FEND} ACCEL_PER_HZ={APH} OUTPUT=raw_data NAME=all

  # Restore everything
  _AWD_RESTORE_ALL LIST={FRONT + "," + REAR}
  {action_respond_info("AWD-IS: Done. Files saved as /tmp/resonances_x_[rear|front|all].csv and ..._y_...")}

# ---------------------------------------------------
# Helpers: enable/disable groups & set TMC currents
# ---------------------------------------------------
[gcode_macro _AWD_SET_GROUP]
description: Enable ACTIVE group, idle INACTIVE group (low current + disable)
gcode:
  {% set ACTIVE   = params.ACTIVE|string %}
  {% set INACTIVE = params.INACTIVE|string %}
  {% set ITEST    = params.ITEST|float %}
  {% set IIDLE    = params.IIDLE|float %}

  # Normalize lists (strip spaces)
  {% set act = ACTIVE.replace(" ","").split(",") %}
  {% set ina = INACTIVE.replace(" ","").split(",") %}

  # Active group: enable + set test current
  {% for s in act %}
    {% if s %}
      SET_STEPPER_ENABLE STEPPER={s} ENABLE=1
      {% if printer.configfile.settings["tmc2209 " ~ s] is defined or
            printer.configfile.settings["tmc5160 " ~ s] is defined or
            printer.configfile.settings["tmc2240 " ~ s] is defined %}
        SET_TMC_CURRENT STEPPER={s} CURRENT={ITEST}
      {% endif %}
    {% endif %}
  {% endfor %}

  # Inactive group: drop current and disable to isolate
  {% for s in ina %}
    {% if s %}
      {% if printer.configfile.settings["tmc2209 " ~ s] is defined or
            printer.configfile.settings["tmc5160 " ~ s] is defined or
            printer.configfile.settings["tmc2240 " ~ s] is defined %}
        SET_TMC_CURRENT STEPPER={s} CURRENT={IIDLE}
      {% endif %}
      SET_STEPPER_ENABLE STEPPER={s} ENABLE=0
    {% endif %}
  {% endfor %}

[gcode_macro _AWD_ENABLE_ALL]
description: Enable all given steppers at test current
gcode:
  {% set LIST  = params.LIST|string %}
  {% set ITEST = params.ITEST|float %}
  {% set lst = LIST.replace(" ","").split(",") %}
  {% for s in lst %}
    {% if s %}
      SET_STEPPER_ENABLE STEPPER={s} ENABLE=1
      {% if printer.configfile.settings["tmc2209 " ~ s] is defined or
            printer.configfile.settings["tmc5160 " ~ s] is defined or
            printer.configfile.settings["tmc2240 " ~ s] is defined %}
        SET_TMC_CURRENT STEPPER={s} CURRENT={ITEST}
      {% endif %}
    {% endif %}
  {% endfor %}

[gcode_macro _AWD_RESTORE_ALL]
description: Re-enable all and let your normal driver currents take back over
gcode:
  {% set LIST = params.LIST|string %}
  {% set lst = LIST.replace(" ","").split(",") %}
  {% for s in lst %}
    {% if s %}
      SET_STEPPER_ENABLE STEPPER={s} ENABLE=1
      # No explicit current restore here; next movement/PRINT_START should re-apply your normal currents
    {% endif %}
  {% endfor %}
```

Notes

Defaults assume a common AWD naming (stepper_x, stepper_y = front; stepper_x1, stepper_y1 = rear). If yours are swapped, just swap the two lists at the top or override them at call time.

This uses TEST_RESONANCES ... OUTPUT=raw_data NAME=<tag>, so you’ll get files like:

/tmp/resonances_x_rear.csv, /tmp/resonances_y_rear.csv

/tmp/resonances_x_front.csv, /tmp/resonances_y_front.csv

/tmp/resonances_x_all.csv, /tmp/resonances_y_all.csv

2) How to run it (in your console)

Basic (use defaults):
AWD_IS_SUITE

If your stepper names differ, override on the fly (example):
```
AWD_IS_SUITE FRONT="stepper_xA,stepper_yA" REAR="stepper_xB,stepper_yB" ITEST=0.9 IIDLE=0.03 FSTART=20 FEND=160 ACCEL_PER_HZ=150
```
3) View & compare the data
Option A — Use Shake&Tune UI (easiest)

Open Shake&Tune in Mainsail/Fluidd, go to Input Shaper → Compare, and load:

rear vs front vs all for X

same for Y

You’ll get exactly the one-graph overlay you asked for per axis with clear labels.

Option B — Auto-generate combined PNGs (optional)

If you have gcode_shell_command enabled (Moonraker), add these two shell helpers to printer.cfg:
```
[gcode_shell_command awd_plot_x]
command: bash -lc 'python3 ~/klipper/scripts/graph_accelerometer.py -o ~/printer_data/config/AWD_COMPARE_X.png /tmp/resonances_x_rear.csv /tmp/resonances_x_front.csv /tmp/resonances_x_all.csv'
timeout: 120
verbose: True

[gcode_shell_command awd_plot_y]
command: bash -lc 'python3 ~/klipper/scripts/graph_accelerometer.py -o ~/printer_data/config/AWD_COMPARE_Y.png /tmp/resonances_y_rear.csv /tmp/resonances_y_front.csv /tmp/resonances_y_all.csv'
timeout: 120
verbose: True
```

Then run:
```
RUN_SHELL_COMMAND CMD=awd_plot_x
RUN_SHELL_COMMAND CMD=awd_plot_y
```

Your combined graphs will be saved to ~/printer_data/config/ so they show up in your web UI file list.

If your Klipper path differs (e.g., KIAUH installs), adjust the python/script paths accordingly.
