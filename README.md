# Sonic-Pad-Adaptive-Meshing-Setup
#---------------------------------------------------------------------------
#This line of coding will go into your printer.cfg under the macros section.
#---------------------------------------------------------------------------



[gcode_macro PRINT_START]
gcode:
  {% set BED = params.BED_TEMP|int %}
  {% set EXTRUDER = params.EXTRUDER_TEMP|int %}
  M190 S{BED}  ;Set bed temperature and wait
  G28  ;Home all axes if not homed
  M109 S{EXTRUDER}  ;Set extruder temperature and wait


[gcode_macro PRINT_END]
gcode:
  _MOVE_AWAY  ;Move away from print
  G1 E-10 F1800  ;Retract filament
  PARK_CENTER_REAR  ;Park central rear
  TURN_OFF_HEATERS  ;Turn off heaters
  M84  ;Disable motors


[gcode_macro M190]
rename_existing: M190.1
gcode:
  {% if printer["gcode_macro status_heating"] != null %}
    status_heating
  {% endif %}
    M190.1 { rawparams }
  {% if printer["gcode_macro status_ready"] != null %}
    status_ready
  {% endif %}


[gcode_macro _CHOME]
gcode:
  {% if printer["gcode_macro status_homing"] != null %}
    status_homing
  {% endif %}
  {% if printer.toolhead.homed_axes != "xyz" %}
  G28
  {% endif %}
  {% if printer["gcode_macro status_ready"] != null %}
    status_ready
  {% endif %}

[gcode_macro M109]
rename_existing: M109.1
gcode:
  {% if printer["gcode_macro status_heating"] != null %}
    status_heating
  {% endif %}
    M109.1 { rawparams }
  {% if printer["gcode_macro status_ready"] != null %}
    status_ready
  {% endif %}


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


[gcode_macro _MOVE_AWAY]
gcode:
    {% set th = printer.toolhead %}
    {% set x_safe = th.position.x + 20 * (1 if th.axis_maximum.x - th.position.x > 20 else -1) %}
    {% set y_safe = th.position.y + 20 * (1 if th.axis_maximum.y - th.position.y > 20 else -1) %}
    {% set z_safe = [th.position.z + 2, th.axis_maximum.z]|min %}
      
    G90                                      ; absolute positioning
    G0 X{x_safe} Y{y_safe} Z{z_safe} F20000  ; move nozzle to remove stringing


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
    
    [gcode_macro BED_MESH_CALIBRATE]
rename_existing: BED_MESH_CALIBRATE_BASE
; gcode parameters
variable_parameter_AREA_START : 0,0
variable_parameter_AREA_END : 0,0
; the clearance between print area and probe area 
variable_mesh_area_offset : 5.0
; number of sample per probe point
variable_probe_samples : 2
; minimum probe count
variable_min_probe_count : 3
; scale up the probe count, should be 1.0 ~ < variable_max_probe_count/variable_min_probe_count
variable_probe_count_scale_factor : 1.0
gcode:
    {% if params.AREA_START and params.AREA_END %}
        {% set bedMeshConfig = printer["configfile"].config["bed_mesh"] %}
        {% set safe_min_x = bedMeshConfig.mesh_min.split(",")[0]|float %}
        {% set safe_min_y = bedMeshConfig.mesh_min.split(",")[1]|float %}
        {% set safe_max_x = bedMeshConfig.mesh_max.split(",")[0]|float %}
        {% set safe_max_y = bedMeshConfig.mesh_max.split(",")[1]|float %}

        {% set area_min_x = params.AREA_START.split(",")[0]|float %}
	{% set area_min_y = params.AREA_START.split(",")[1]|float %}
	{% set area_max_x = params.AREA_END.split(",")[0]|float %}
	{% set area_max_y = params.AREA_END.split(",")[1]|float %}

        {% set meshPointX = bedMeshConfig.probe_count.split(",")[0]|int %}
        {% set meshPointY = bedMeshConfig.probe_count.split(",")[1]|int %}
	
	{% set meshMaxPointX = meshPointX %}
	{% set meshMaxPointY = meshPointY %}


        {% if (area_min_x < area_max_x) and (area_min_y < area_max_y) %}
            {% if area_min_x - mesh_area_offset >=  safe_min_x %}
                {% set area_min_x = area_min_x - mesh_area_offset %}
            {% else %}
                {% set area_min_x = safe_min_x %}
            {% endif %}

            {% if area_min_y - mesh_area_offset >=  safe_min_y %}
                {% set area_min_y = area_min_y - mesh_area_offset %}
            {% else %}
                {% set area_min_y = safe_min_y %}
            {% endif %}

            {% if area_max_x + mesh_area_offset <=  safe_max_x %}
                {% set area_max_x = area_max_x + mesh_area_offset %}
            {% else %}
                {% set area_max_x = safe_max_x %}
            {% endif %}

            {% if area_max_y + mesh_area_offset <=  safe_max_y %}
                {% set area_max_y = area_max_y + mesh_area_offset %}
            {% else %}
                {% set area_max_y = safe_max_y %}
            {% endif %}

            {% set meshPointX = (meshPointX * (area_max_x - area_min_x) / (safe_max_x - safe_min_x) * probe_count_scale_factor)|round(0)|int %}
            {% if meshPointX < min_probe_count %}
                {% set meshPointX = min_probe_count %}
            {% endif %}
	    {% if meshPointX > meshMaxPointX %}
                {% set meshPointX = meshMaxPointX %}
            {% endif %}

            {% set meshPointY = (meshPointY * (area_max_y -area_min_y ) / (safe_max_y - safe_min_y) * probe_count_scale_factor )|round(0)|int %}
            {% if meshPointY < min_probe_count %}
                {% set meshPointY = min_probe_count %}
            {% endif %}
	    {% if meshPointY > meshMaxPointY %}
                {% set meshPointY = meshMaxPointY %}
            {% endif %}

            BED_MESH_CALIBRATE_BASE mesh_min={area_min_x},{area_min_y} mesh_max={area_max_x},{area_max_y} probe_count={meshPointX},{meshPointY} samples={probe_samples|int}
        {% else %}
            BED_MESH_CALIBRATE_BASE
        {% endif %}
    {% else %}
        BED_MESH_CALIBRATE_BASE
    {% endif %}
    
 #------------------------------------------------------------------------------------
 #------------------------------------------------------------------------------------
 #------------------------------------------------------------------------------------
 
#Starting GCODE in my Slicer (Orca Slicer) But is the same in every one 
 
M117
G28
PRINT_START EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[first_layer_bed_temperature]
BED_MESH_CALIBRATE AREA_START={first_layer_print_min[0]},{first_layer_print_min[1]} AREA_END={first_layer_print_max[0]},{first_layer_print_max[1]}
_PURGE_LINE
 
 
 #Ending GCODE in my Slicer (Orca Slicer) But is the same in every one
 
 PRINT_END
 
 
#Before layer Changing GCODE in my Slicer (Orca Slicer) But is the same in every one (This is for the Timelapse on sonic pad not needed for mesh but here to provide info where it goes)
 
 Timelapse_Take_Frame
 
 
 
 
 
 
 
 
    
    
