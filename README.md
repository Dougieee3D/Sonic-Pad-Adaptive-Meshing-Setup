Please Download the Txt File and READ IT do not just skip around its very straight forward and no im not around all the time for questions but leave them and ill get back as soon as i can, Thanks and happy printing!


If you happen to slam the front of the bed when the print ends change the 230 <--  -2 F3600  in the marco

 [gcode_macro PARK_CENTER_REAR]
gcode:
    {% if printer["gcode_macro status_busy"] != null %}
      status_busy
    {% endif %}
    {% set th = printer.toolhead %}
    {% set x_safe = th.position.x + 20 * (1 if th.axis_maximum.x - th.position.x > 20 else -1) %}
    {% set y_safe = th.position.y + 20 * (1 if th.axis_maximum.y - th.position.y > 20 else -1) %}

    G0 X{th.axis_maximum.x//2} Y{230 - 2} F3600  
    {% if printer["gcode_macro status_ready"] != null %}
    status_ready
    {% endif %}
    
Also Please mess or import your pruge line cordinations if you do not want mine into the printer.cfg 
    
    [gcode_macro _PURGE_LINE]
gcode:
  {% if printer["gcode_macro status_cleaning"] != null %}
    status_cleaning
  {% endif %}
  SAVE_GCODE_STATE NAME=Pre_Prime
        
  G90
  G92 E0 ;Reset Extruder

  G1 Z10.0 F3000 ;Move Z Axis up
  G1 X0 Y0;
  G1 E10.0 F1800
  G1 Z0.28 F5000.0 ;Move to start position
  G1 X220 Y0 Z0.28 F1500.0 E30 ;Draw the first line
  G92 E0 ;Reset Extruder
  G1 Z10 F3000 ;Move Z Axis up
  RESTORE_GCODE_STATE NAME=Pre_Prime

  {% if printer["gcode_macro status_printing"] != null %}
    status_printing
  {% endif %}
