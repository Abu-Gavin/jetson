# ROS2 Humble API Interfaces (Jetson Orin NX / Ubuntu 22.04)

This document lists the migrated packages and the exact ROS interfaces required to run them.

> Current-use note (2026-04-18): the car's active drive stack is the `turn_on_wheeltec_robot + velpub` chain described in `docs/当前主链运行逻辑总览.md`.  
> This file is an interface index for the whole workspace, so it also includes alternate or incomplete packages such as `wheeltec_yolo_action`, `camera`, and `largemodel`.

## Migrated packages

- `wheeltec_yolo_action` (ROS2 Python nodes + custom msgs)
- `turn_on_wheeltec_robot` (ROS2 C++ base serial driver)
- `simple_follower` (ROS2 interface package, `Position.msg`)
- `darknet_ros_msgs` (ROS2 interface package, `BoundingBox/BoundingBoxes`)
- `velpub` (ROS2 Python lane-keeping bridge + velocity bridge)
- `interfaces` (ROS2 interface package aligned to `jetson/yahboom_ws` API names)

ROS1-only script `turn_on_wheeltec_robot/scripts/send_mark.py` has been removed from active code because it depends on `move_base` APIs that are not available in Humble/Nav2.
`turn_on_wheeltec_robot` now uses Linux `termios` directly for serial I/O (no ROS1 `serial` package dependency).
`velpub` ROS1 launch files (`*.launch`) and dynamic reconfigure file (`cfg/detect_line.cfg`) were removed from active code and replaced by ROS2 launch files (`*.launch.py`).
`jetson/yahboom_ws/src/interfaces` currently contains zero-byte source interface files in this synced copy, so `talos_ws/src/interfaces` uses temporary placeholder fields with identical interface names for integration.

## Launch entry points

- Line-follow stack (decision + execution):
  - `ros2 launch wheeltec_yolo_action dp_drive.launch.py`
- Full stack (base + line-follow):
  - `ros2 launch wheeltec_yolo_action full_stack.launch.py`
- Base serial only:
  - `ros2 launch turn_on_wheeltec_robot turn_on_wheeltec_robot.launch.py`
- Local lane bridge only (`velpub`, current default):
  - `ros2 launch velpub detect_line.launch.py`
- PID velocity bridge only (`velpub`):
  - `ros2 launch velpub AIcar_move.launch.py`
- Sign/traffic-rule fusion (`velpub`):
  - `ros2 launch velpub get_yolo_rst.launch.py`
- Current base + local lane-follow order:
  - 1) `ros2 launch turn_on_wheeltec_robot turn_on_wheeltec_robot.launch.py`
  - 2) `ros2 launch velpub detect_line.launch.py`
  - 3) `ros2 launch velpub AIcar_move.launch.py`
- Intended traffic-rule fusion order:
  - 1) `ros2 launch turn_on_wheeltec_robot turn_on_wheeltec_robot.launch.py`
  - 2) `ros2 launch velpub detect_line.launch.py`
  - 3) `ros2 launch velpub AIcar_move.launch.py cmd_vel_topic:=/cmd_vel_1`
  - 4) `ros2 launch velpub get_yolo_rst.launch.py`
- Important current-code note:
  - `AIcar_move.launch.py` currently defaults `cmd_vel_topic` to `/cmd_vel`
  - `velpub.py` currently defaults `input_topic` to `/cmd_vel_1`
  - Therefore `velpub.py` only joins the main control loop when `cmd_vel_topic:=/cmd_vel_1` is used

## Topic APIs

### `turn_on_wheeltec_robot` / node `wheeltec_robot`

- Subscribed topics:
  - `/cmd_vel` : `geometry_msgs/msg/Twist`
- Published topics:
  - `/odom` : `nav_msgs/msg/Odometry`
  - `/imu1` : `sensor_msgs/msg/Imu`
  - `/PowerVoltage` : `std_msgs/msg/Float32`

### `wheeltec_yolo_action` / node `drive_execution_node`

- Subscribed topics:
  - `/usb_cam/image_raw` : `sensor_msgs/msg/Image`
  - `/ctrl_data` : `wheeltec_yolo_action/msg/CtrlData`
  - `/decelerate_data` : `std_msgs/msg/Int8`
- Published topics:
  - `/cmd_vel` : `geometry_msgs/msg/Twist`
  - `/control_flag` : `std_msgs/msg/Int8`

### `wheeltec_yolo_action` / node `drive_decision_node`

- Subscribed topics:
  - `/usb_cam/image_raw` : `sensor_msgs/msg/Image`
  - `/object_tracker/current_position` : `simple_follower/msg/Position`
  - `/darknet_ros/bounding_boxes` : `darknet_ros_msgs/msg/BoundingBoxes`
  - `/control_flag` : `std_msgs/msg/Int8`
- Published topics:
  - `/ctrl_data` : `wheeltec_yolo_action/msg/CtrlData`
  - `/decelerate_data` : `std_msgs/msg/Int8`
  - `/line_judgment` : `std_msgs/msg/Int8`
  - `/cmd_vel` : `geometry_msgs/msg/Twist`

### `wheeltec_yolo_action` / node `no_map_action`

- Subscribed topics:
  - `/darknet_ros/bounding_boxes` : `darknet_ros_msgs/msg/BoundingBoxes`
- Published topics:
  - `/cmd_vel` : `geometry_msgs/msg/Twist`

### `velpub` / node `detect_line` (script: `detect_line3.py`, current launch default)

- Published topics:
  - `/lane/steering_angle` : `std_msgs/msg/Float32`
  - `/lane/turn_severity` : `std_msgs/msg/Float32`
  - `/cmd_vel` : `geometry_msgs/msg/Twist` (only when `publish_cmd_vel=true`)

### `velpub` / node `detect_line` (script: `detect_line2.py`, alternate remote-inference path)

- Published topics:
  - `/lane/steering_angle` : `std_msgs/msg/Float32`
  - `/lane/turn_severity` : `std_msgs/msg/Float32`
  - `/cmd_vel` : `geometry_msgs/msg/Twist` (only when `publish_cmd_vel=true`)

### `velpub` / node `vth2ros` (script: `vth2ros.py`)

- Published topics:
  - `/cmd_vel_1` : `geometry_msgs/msg/Twist` (script default topic)
  - `/cmd_vel` : `geometry_msgs/msg/Twist` when launched via current `AIcar_move.launch.py` defaults

### `velpub` / node `velpub` (script: `velpub.py`)

- Subscribed topics:
  - `/cmd_vel_1` : `geometry_msgs/msg/Twist` (configurable by `input_topic`)
- Published topics:
  - `/cmd_vel` : `geometry_msgs/msg/Twist` (configurable by `output_topic`)

## Message APIs

### `wheeltec_yolo_action/msg/CtrlData`

- `float32 mask_x_left`
- `float32 mask_x_right`
- `float32 mask_y_top`
- `float32 mask_y_bot`
- `float32 center_target`
- `float32 vel_x`
- `float32 vel_y_p`
- `float32 vel_y_d`
- `float32 vel_z_p`
- `float32 vel_z_d`
- `int8 en`

### `simple_follower/msg/Position`

- `float32 angle_x`
- `float32 angle_y`
- `float32 distance`

### `darknet_ros_msgs/msg/BoundingBox`

- `float64 probability`
- `int64 xmin`
- `int64 ymin`
- `int64 xmax`
- `int64 ymax`
- `int16 id`
- `string class_name`

### `darknet_ros_msgs/msg/BoundingBoxes`

- `std_msgs/Header header`
- `std_msgs/Header image_header`
- `darknet_ros_msgs/BoundingBox[] bounding_boxes`

### `interfaces/srv/*` (temporary placeholders)

- `Audio.srv`:
  - Request: `string request_json`
  - Response: `string response_json`
- `Audio2.srv`:
  - Request: `string request_json`
  - Response: `string response_json`
- `LargeScaleModel.srv`:
  - Request: `string request_json`
  - Response: `string response_json`
- `Qwen25.srv`:
  - Request: `string request_json`
  - Response: `string response_json`
- `TextToImage.srv`:
  - Request: `string request_json`
  - Response: `string response_json`
- `Vision.srv`:
  - Request: `string request_json`
  - Response: `string response_json`

### `interfaces/action/Rot` (temporary placeholder)

- Goal:
  - `string goal_json`
- Result:
  - `bool success`
  - `string result_json`
- Feedback:
  - `string feedback_json`

## Parameter APIs

### `turn_on_wheeltec_robot` / node `wheeltec_robot`

- `usart_port_name` (string, node default `/dev/ttyUSB0`)
- `serial_baud_rate` (int, default `115200`)
- `odom_frame_id` (string, node default `odom`, launch default `odom_combined`)
- `robot_frame_id` (string, default `base_footprint`)
- `gyro_frame_id` (string, default `gyro_link`)
- `loop_rate_hz` (int, default `200`)

### `wheeltec_yolo_action` / node `drive_execution_node`

- `show_debug` (bool)
- `image_topic` (string)
- `cmd_vel_topic` (string)
- `ctrl_topic` (string)
- `decelerate_topic` (string)
- `control_flag_topic` (string)

### `wheeltec_yolo_action` / node `drive_decision_node`

- `loop_rate_hz`
- `out_l_center_target`, `out_l_vel_z_p`, `out_l_vel_z_d`
- `out_r_center_target`, `out_r_vel_z_p`, `out_r_vel_z_d`
- `in_l_center_target`, `in_l_vel_y_p`, `in_l_vel_y_d`, `in_l_vel_z_p`, `in_l_vel_z_d`
- `in_r_center_target`, `in_r_vel_y_p`, `in_r_vel_y_d`, `in_r_vel_z_p`, `in_r_vel_z_d`
- `left_stop_xmin`, `left_stop_ymin`, `left_stop_xmax`, `left_stop_ymax`
- `right_stop_xmin`, `right_stop_ymin`, `right_stop_xmax`, `right_stop_ymax`
- `road_con_par_rig_min`, `road_con_par_rig_max`
- `road_con_par_left_min`, `road_con_par_left_max`

### `wheeltec_yolo_action` / node `no_map_action`

- `trigger_count`
- `angular_speed`
- `linear_speed`
- `turn_duration_sec`
- `move_duration_sec`

### `velpub` / node `detect_line` (`detect_line3.py`, current launch default)

- `video_device` (string, node default `/dev/video0`, launch default `/dev/video3`)
- `frame_width` (int, default `640`)
- `frame_height` (int, default `480`)
- `loop_hz` (double, default `20.0`)
- `note_file` (string, default `<install-prefix>/lib/velpub/note.txt` from launch)
- `write_note_file` (bool, default `true`)
- `weights_path` (string, default `<script-dir>/params/min_loss.pth`)
- `steering_topic` (string, default `/lane/steering_angle`)
- `severity_topic` (string, default `/lane/turn_severity`)
- `cmd_vel_topic` (string, default `/cmd_vel`)
- `publish_cmd_vel` (bool, default `false` in launch file)
- `linear_speed` (double, default `0.15`)
- `steering_gain` (double, launch default `0.005`)
- `stream_host` (string, default `0.0.0.0`)
- `stream_port` (int, default `8090`)
- `stream_quality` (int, default `75`)
- `enable_stream_server` (bool, default `true`)

### `velpub` / node `detect_line` (`detect_line2.py`, alternate remote-inference path)

- `laptop_ip` (string, script default `10.139.124.70`)
- `port` (int, default `8000`)
- `video_device` (string, default `/dev/video0`)
- `frame_width` (int, default `640`)
- `frame_height` (int, default `480`)
- `jpeg_quality` (int, default `80`)
- `loop_hz` (double, default `20.0`)
- `reconnect_interval_sec` (double, default `2.0`)
- `steering_topic` (string, default `/lane/steering_angle`)
- `severity_topic` (string, default `/lane/turn_severity`)
- `cmd_vel_topic` (string, default `/cmd_vel`)
- `publish_cmd_vel` (bool, default `false` in launch file)
- `linear_speed` (double, default `0.15`)
- `steering_gain` (double, default `0.01`)
- `note_file` (string, default `<install-prefix>/lib/velpub/note.txt`)
- `write_note_file` (bool, default `true`)
- `response_floats` (int, default `2`)

### `velpub` / node `vth2ros` (`vth2ros.py`)

- `Speed_stright` (double, script default `0.3`, launch default `0.16`)
- `K_curve` (double, script default `0.1`, launch default `0.05`, kept for compatibility and not consumed in control loop)
- `Max_angular_curve` (double, script default `4.0`, launch default `2.6`)
- `P` (double, script default `0.1`, launch default `0.0200`)
- `I` (double, script default `0.01`, launch default `0.0008`)
- `D` (double, script default `0.01`, launch default `0.0030`)
- `DeadArea` (double, script default `4.0`, launch default `2.5`)
- `publish_hz` (double, default `30.0`)
- `note_file` (string, default `<install-prefix>/lib/velpub/note.txt`)
- `TurnSlowdownGain` (double, default `0.70`)
- `TurnAngularBoost` (double, default `0.40`)
- `MinSpeedRatio` (double, default `0.40`)
- `cmd_vel_topic` (string, script default `/cmd_vel_1`, launch default `/cmd_vel`)

### `velpub` / node `velpub` (`velpub.py`)

- `input_topic` (string, default `/cmd_vel_1`)
- `output_topic` (string, default `/cmd_vel`)

## External APIs you still need to provide

- Camera driver publishing `sensor_msgs/msg/Image` on `/usb_cam/image_raw` is still required for the alternate `wheeltec_yolo_action` stack
- Detector publishing `darknet_ros_msgs/msg/BoundingBoxes` on `/darknet_ros/bounding_boxes` is still required for the alternate `wheeltec_yolo_action` stack
- Tracker publishing `simple_follower/msg/Position` on `/object_tracker/current_position` is still required for the alternate `wheeltec_yolo_action` stack
- TCP lane inference server for `velpub/detect_line2.py`:
  - Socket endpoint: `<laptop_ip>:<port>` (script default `10.139.124.70:8000`)
  - Protocol: first 4 bytes big-endian `uint32` frame length, then JPG bytes; server replies with 1 or 2 `float32` values depending on `response_floats`
- `detect_line2.py` and `detect_line3.py` both write `deviation,severity` text into `note_file`, consumed by `vth2ros.py`
- `velpub.py` additionally depends on DeepStream-Yolo writing sign results into `internal_memory.txt`

## Backup paths

- `backup/ros1_removed_backup/wheeltec_yolo_action_20260318_174029`
- `backup/ros1_removed_backup/turn_on_wheeltec_robot_20260318_175437`
- `backup/ros1_removed_backup/velpub_20260318_181156`
