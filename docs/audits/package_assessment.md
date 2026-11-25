# Package Assessment

## Purpose

This document records what each ROS 2 package does, how it connects to the rest of the system, and whether it should be kept, changed, archived, or removed.

Assessments are based on repository contents. No runtime tests were performed, so “prototype” and “placeholder” describe what the source code appears to do, not verified runtime behavior.

## Assessment summary

| Package or package group | Role | Key nodes or files | Current status | Recommendation |
|---|---|---|---|---|
| `differential_drive_robot_model` directory | Groups three standalone ROS packages that demonstrate a generic differential-drive robot and Gazebo setup | `differential_drive_robot_description`, `differential_drive_robot_gazebo`, `differential_drive_robot_control` | Reference prototype; not an IGV simulation | Keep as reference; adapt only if useful |
| `igv_behavior` | Chooses lane-following, stop, hold, or lane-search behavior from perception messages | `igv_behavior/behavior_manager.py` | Prototype | Replace for the planned local-navigation approach |
| `igv_bringup` | Defines launch configurations, RViz settings, and a costmap parameter file | `launch/`, `rviz/`, `config/obstacle_layer_params.yaml` | Prototype; launch paths are incomplete or inconsistent | Keep and revise |
| `igv_control` | Converts behavior commands into `geometry_msgs/Twist` on `/cmd_vel` | `igv_control/cmd_vel_manager.py` | Prototype | Keep as a ROS interface example; revise command semantics and timeout behavior |
| `igv_description` | Provides a minimal IGV visual model and sensor frames | `urdf/igv.urdf` | Placeholder visual description | Replace or expand for simulation |
| `igv_interfaces` | Defines custom messages used between perception and behavior nodes | `msg/*.msg` | Active interface definitions in source; semantics are prototype-level | Keep conceptually; revise with navigation architecture |
| `igv_localization` | Publishes fixed odometry and an `odom → base_link` transform | `igv_localization/odom_stub.py` | Placeholder | Replace with simulator or vehicle odometry |
| `igv_perception` | Contains lane segmentation, obstacle detection, and point-cloud filtering nodes | `igv_perception/*.py` | Prototype; lane model asset is absent from the uploaded repository snapshot | Keep reusable processing patterns; replace or redesign outputs |
| `igv_tools` | Provides fake lane and obstacle publishers for integration experiments | `igv_tools/fake_*_publisher.py` | Test utilities | Keep for testing; do not use as real perception |

> The directory `differential_drive_robot_model/` is not itself a ROS package in this snapshot. It contains three separate ROS packages listed below.

## Package details

### `differential_drive_robot_description`

**Purpose:**  
Defines a generic differential-drive robot using Xacro/URDF, including visual, collision, and inertial elements, and provides an RViz viewing launch.

**Key contents:**
- Nodes or executables: None in the package itself.
- Launch files: `launch/rviz_visualize.launch.py`.
- Config, message, or robot-description files: `urdf/differential_drive_robot.xacro`, `urdf/differential_drive_robot.gazebo`, `urdf/xacro_variables.xacro`, `urdf/materials.xacro`, `rviz/urdf.rviz`.

**Connections to other packages:**
- Subscribes to: None.
- Publishes to: None directly.
- Depends on: `robot_state_publisher`, `xacro`, RViz, and joint-state publisher packages for visualization.

**Current status:**  
Reference model. The Xacro includes chassis, wheels, casters, collision geometry, and inertial values. Its geometry and dimensions are generic and should not be assumed to match the IGV.

**Recommendation:**  
Keep as a reference; adapt only if useful.

**Reasoning:**  
It demonstrates a more complete robot description than `igv_description/urdf/igv.urdf`. It can help with Gazebo model structure, but replacing the IGV description with this model would not represent the actual vehicle.

**Open questions:**
- Does the IGV use the same differential-drive geometry and wheel arrangement?
- Which model should become the authoritative IGV simulation description?

**Evidence reviewed:**
- `differential_drive_robot_description/urdf/differential_drive_robot.xacro`
- `differential_drive_robot_description/urdf/differential_drive_robot.gazebo`
- `differential_drive_robot_description/urdf/xacro_variables.xacro`
- `differential_drive_robot_description/README.md`

### `differential_drive_robot_gazebo`

**Purpose:**  
Launches a Gazebo Classic world and spawns the generic differential-drive model.

**Key contents:**
- Nodes or executables: Launches Gazebo, `robot_state_publisher`, and `gazebo_ros/spawn_entity.py`.
- Launch files: `launch/differential_drive_robot_gazebo.launch.py`, `launch/gazebo_world_visualize.launch.py`, `launch/gazebo_visualize.launch.py`, `launch/gazebo_world_visualize.launch.py`, `launch/differential_drive_robot_rviz_gazebo.launch.py`.
- Config, message, or robot-description files: `world/world.world`; robot description comes from `differential_drive_robot_description`.

**Connections to other packages:**
- Subscribes to: The Gazebo differential-drive plugin consumes `/cmd_vel`.
- Publishes to: The plugin publishes odometry and TF for its configured frames.
- Depends on: `differential_drive_robot_description`, Gazebo Classic packages, `robot_state_publisher`, Xacro, and launch packages.

**Current status:**  
Reference simulation prototype. The supplied world contains a ground plane and sun, but no IGV track, obstacles, or ZED sensor plugin.

**Recommendation:**  
Keep as a reference; adapt for IGV simulation if appropriate.

**Reasoning:**  
The launch pattern and differential-drive plugin are useful examples. The model, sensor setup, and world need to be made IGV-specific before they can validate the planned navigation software.

**Open questions:**
- Is Gazebo Classic the intended simulator and ROS distribution for the final project environment?
- What track, ramps, boundaries, and obstacles should be represented in simulation?
- What sensor simulation plugins or simulated data sources will replace the ZED hardware?

**Evidence reviewed:**
- `differential_drive_robot_gazebo/launch/gazebo_world_visualize.launch.py`
- `differential_drive_robot_gazebo/launch/differential_drive_robot_gazebo.launch.py`
- `differential_drive_robot_gazebo/world/world.world`
- `differential_drive_robot_description/urdf/differential_drive_robot.gazebo`

### `differential_drive_robot_control`

**Purpose:**  
Provides keyboard teleoperation for the generic differential-drive simulation.

**Key contents:**
- Nodes or executables: `nodes/teleop.py`.
- Launch files: `launch/teleop.launch.py`.
- Config, message, or robot-description files: None.

**Connections to other packages:**
- Subscribes to: Keyboard input from the terminal.
- Publishes to: `/cmd_vel` using `geometry_msgs/Twist`.
- Depends on: `rclpy`, `geometry_msgs`, and ROS launch packages.

**Current status:**  
Utility/reference control package, separate from the IGV behavior-to-command path.

**Recommendation:**  
Keep as a reference or testing utility.

**Reasoning:**  
It demonstrates a manual command source for the generic simulation. It does not implement autonomous navigation or connect to `igv_behavior`.

**Open questions:**
- Should keyboard control remain available as a separate simulation test tool?
- How will manual control be arbitrated against autonomous commands in the IGV stack?

**Evidence reviewed:**
- `differential_drive_robot_control/nodes/teleop.py`
- `differential_drive_robot_control/launch/teleop.launch.py`
- `differential_drive_robot_control/package.xml`

### `igv_behavior`

**Purpose:**  
Combines lane and obstacle summaries into a behavior state and target motion command.

**Key contents:**
- Nodes or executables: `igv_behavior/behavior_manager.py`, executable `behavior_manager`.
- Launch files: Started by launch files in `igv_bringup`.
- Config, message, or robot-description files: Uses `LaneEstimate`, `ObstacleSummary`, and publishes `BehaviorState`.

**Connections to other packages:**
- Subscribes to: `/lane_estimate` (`LaneEstimate`), `/obstacle_summary` (`ObstacleSummary`).
- Publishes to: `/behavior_state` (`BehaviorState`).
- Depends on: `rclpy`, `igv_interfaces`.

**Current status:**  
Prototype lane-following and stop/hold logic. It stops for a reported obstacle within 3 m, holds briefly after a reported obstacle clears, follows a valid fresh lane estimate, and requests a turn while stopped when lane data is missing or invalid.

**Recommendation:**  
Replace the lane-following decision logic for the planned local-navigation approach; retain only useful ROS patterns if desired.

**Reasoning:**  
The node does not choose a path around obstacles or track boundaries. It ignores the left/right-open fields in `ObstacleSummary` and only uses obstacle status and distance. It also has no obstacle-message timeout. Its behavior depends on the most recently received obstacle state.

**Open questions:**
- What node or navigation framework will select a safe path from the costmap?
- What should happen when obstacle or perception messages stop arriving?
- Should behavior remain a separate node once a planner/controller is introduced?

**Evidence reviewed:**
- `igv_behavior/behavior_manager.py`
- `igv_interfaces/msg/BehaviorState.msg`
- `igv_interfaces/msg/LaneEstimate.msg`
- `igv_interfaces/msg/ObstacleSummary.msg`

### `igv_bringup`

**Purpose:**  
Provides launch files for hardware, test, and visualization configurations, plus a costmap parameter file.

**Key contents:**
- Nodes or executables: No standalone console nodes.
- Launch files: `launch/full_system.launch.py`, `launch/camera_only.launch.py`, `launch/lane_test.launch.py`, `launch/real_lane_test.launch.py`, `launch/obstacle_layer_view.launch.py`.
- Config, message, or robot-description files: `config/obstacle_layer_params.yaml`, `rviz/igv_default.rviz`.

**Connections to other packages:**
- Subscribes to: None directly.
- Publishes to: None directly; launch files start other publishers.
- Depends on: Launch files reference `zed_wrapper`, `vectornav`, `robot_state_publisher`, `igv_behavior`, `igv_control`, `igv_localization`, `igv_perception`, `igv_tools`, RViz, and Nav2 packages.

**Current status:**  
Prototype bringup package with several different launch configurations. `full_system.launch.py` starts ZED and VectorNav, robot-state publishing, behavior/control nodes, an odometry stub, and RViz, but does not start the perception nodes or costmap. `real_lane_test.launch.py` starts the lane detector, obstacle detector, and cloud filter as well. `lane_test.launch.py` uses a fake lane publisher.

**Recommendation:**  
Keep and revise.

**Reasoning:**  
The launch files are useful records of how the prototype was assembled, but they do not yet provide one coherent, complete navigation launch. The costmap preview is separate and does not start its sensor inputs. The costmap launch also uses a hard-coded workspace path, and its parameter YAML node key differs from the launched node name. The package setup installs launch and RViz files but not the config directory.

**Open questions:**
- Which launch should become the supported hardware bringup?
- Which launch should become the simulation bringup?
- Should the costmap, sensor filters, and planner/controller be composed in one navigation launch?
- Do all external package dependencies match the intended ROS distribution and machine setup?

**Evidence reviewed:**
- `igv_bringup/launch/full_system.launch.py`
- `igv_bringup/launch/camera_only.launch.py`
- `igv_bringup/launch/lane_test.launch.py`
- `igv_bringup/launch/real_lane_test.launch.py`
- `igv_bringup/launch/obstacle_layer_view.launch.py`
- `igv_bringup/config/obstacle_layer_params.yaml`
- `igv_bringup/setup.py`
- `igv_bringup/package.xml`

### `igv_control`

**Purpose:**  
Converts `BehaviorState` commands into a `Twist` velocity command.

**Key contents:**
- Nodes or executables: `igv_control/cmd_vel_manager.py`, executable `cmd_vel_manager`.
- Launch files: Started by launch files in `igv_bringup`.
- Config, message, or robot-description files: Uses `BehaviorState`.

**Connections to other packages:**
- Subscribes to: `/behavior_state` (`BehaviorState`).
- Publishes to: `/cmd_vel` (`geometry_msgs/Twist`).
- Depends on: `rclpy`, `geometry_msgs`, `igv_interfaces`.

**Current status:**  
Prototype command bridge. It republishes the last received command every 0.1 seconds and does not expire stale commands.

**Recommendation:**  
Keep as an interface example; revise for the final controller and vehicle command contract.

**Reasoning:**  
The subscriber/publisher pattern is reusable. However, `BehaviorState.target_steering_rad` is copied directly into `Twist.angular.z`, conventionally an angular velocity in rad/s. The intended units and vehicle kinematics need clarification. The node also has no command watchdog.

**Open questions:**
- Will the final vehicle accept `Twist`, wheel commands, or another interface?
- Is the turn quantity intended to be steering angle or yaw rate?
- What safety behavior should occur if the command source stops publishing?

**Evidence reviewed:**
- `igv_control/igv_control/cmd_vel_manager.py`
- `igv_interfaces/msg/BehaviorState.msg`
- `igv_control/package.xml`
- `igv_control/setup.py`

### `igv_description`

**Purpose:**  
Provides a minimal robot model and fixed sensor frames for visualization.

**Key contents:**
- Nodes or executables: No standalone console nodes; consumed by `robot_state_publisher`.
- Launch files: No package-specific launch file used by the bringup launch group.
- Config, message, or robot-description files: `urdf/igv.urdf`.

**Connections to other packages:**
- Subscribes to: None.
- Publishes to: No topics directly; `robot_state_publisher` publishes TF from the URDF.
- Depends on: `robot_state_publisher`.

**Current status:**  
Placeholder visual description. It defines `base_link`, `imu_link`, and `zed_camera_link` with simple visual boxes and fixed joints. It lacks collision/inertial properties and wheel/drive elements needed for a physical simulation.

**Recommendation:**  
Keep as a frame/visualization reference; replace or expand for the IGV simulation.

**Reasoning:**  
It provides a starting point for displaying the IGV and defining sensor mounting frames. The sensor pose must be checked against the real vehicle. The URDF camera link is named `zed_camera_link`, while the RViz config uses `zed_left_camera_frame`; verify the live TF connection.

**Open questions:**
- What are the actual sensor mounting positions and orientations?
- Which component publishes the transform between `zed_camera_link` and the ZED driver’s frame?
- What are the vehicle’s geometry, inertial properties, collision geometry, and wheel arrangement?

**Evidence reviewed:**
- `igv_description/urdf/igv.urdf`
- `igv_description/setup.py`
- `igv_description/package.xml`
- `igv_bringup/rviz/igv_default.rviz`

### `igv_interfaces`

**Purpose:**  
Defines custom ROS messages used to exchange lane, obstacle, and behavior information.

**Key contents:**
- Nodes or executables: None.
- Launch files: None.
- Config, message, or robot-description files: `msg/LaneEstimate.msg`, `msg/ObstacleSummary.msg`, `msg/BehaviorState.msg`.

**Connections to other packages:**
- Subscribes to: None.
- Publishes to: None.
- Depends on: `std_msgs`; generated messages are used by `igv_perception`, `igv_behavior`, `igv_control`, and `igv_tools`.

**Current status:**  
The message definitions are built with `rosidl` and form the current interfaces between prototype nodes. Their existence in source was reviewed; runtime builds were not tested.

**Recommendation:**  
Keep the interface package; revise message semantics as the architecture changes.

**Reasoning:**  
Custom interfaces are a normal way to define ROS data contracts. `ObstacleSummary` contains left/right-open fields that the current behavior node does not use. `BehaviorState.target_steering_rad` is passed to `Twist.angular.z` without a unit conversion. The messages should match the final navigation and control design.

**Open questions:**
- Will obstacle data be represented by Nav2 costmap topics, custom messages, or both?
- What command type and units should the controller interface use?
- Which message fields remain necessary after moving away from lane following?

**Evidence reviewed:**
- `igv_interfaces/msg/LaneEstimate.msg`
- `igv_interfaces/msg/ObstacleSummary.msg`
- `igv_interfaces/msg/BehaviorState.msg`
- `igv_interfaces/CMakeLists.txt`
- `igv_interfaces/package.xml`

### `igv_localization`

**Purpose:**  
Provides an odometry placeholder and an `odom → base_link` transform.

**Key contents:**
- Nodes or executables: `igv_localization/odom_stub.py`, executable `odom_stub`.
- Launch files: Started by launch files in `igv_bringup`.
- Config, message, or robot-description files: `config/odom_stub.yaml`.

**Connections to other packages:**
- Subscribes to: None.
- Publishes to: `/odom` (`nav_msgs/Odometry`) and TF (`odom → base_link`).
- Depends on: `rclpy`, `nav_msgs`, `geometry_msgs`, `tf2_ros`.

**Current status:**  
Placeholder. It repeatedly publishes fixed parameter values and does not estimate motion from sensors, wheel feedback, or `/cmd_vel`.

**Recommendation:**  
Replace with simulator odometry for simulation and an appropriate vehicle localization source for hardware.

**Reasoning:**  
A stable placeholder transform can support early visualization, but it cannot provide changing vehicle pose. If another node also publishes the same odometry transform, duplicate TF publishers may conflict.

**Open questions:**
- What odometry source will be used in Gazebo?
- What sensors and localization method will provide hardware odometry?
- Which node should own the `odom → base_link` transform?

**Evidence reviewed:**
- `igv_localization/igv_localization/odom_stub.py`
- `igv_localization/config/odom_stub.yaml`
- `igv_localization/package.xml`
- `igv_localization/setup.py`

### `igv_perception`

**Purpose:**  
Processes ZED camera images and point clouds to produce lane estimates, obstacle summaries, point clouds, and debug visualizations.

**Key contents:**
- Nodes or executables:
  - `igv_perception/segformer_lane_node.py`
  - `igv_perception/obstacle_detector_node.py`
  - `igv_perception/cloud_debug_filter_node.py`
- Launch files: Perception nodes are started through `igv_bringup/launch/real_lane_test.launch.py`.
- Config, message, or robot-description files: Loads `models/best_modelV.2.pth` for lane segmentation; no such model file is present in the uploaded repository snapshot.

**Connections to other packages:**
- Subscribes to: ZED RGB image, registered point cloud, and camera information.
- Publishes to: `/lane_estimate`, `/obstacle_summary`, `/obstacles/points`, `/debug/filtered_cloud`, markers, and debug images.
- Depends on: `rclpy`, sensor and visualization messages, `cv_bridge`, `sensor_msgs_py`, `tf2_ros`, `igv_interfaces`, and Python libraries including PyTorch, torchvision, Transformers, Pillow, NumPy, and OpenCV.

**Current status:**  
Prototype. The lane node uses a learned segmentation model. The obstacle node performs depth-range thresholding and image-region grouping. The cloud filter uses uniform point skipping, TF transformation, and height/range filtering. These nodes do not produce one fused obstacle-and-boundary representation for local navigation.

**Recommendation:**  
Keep useful ROS integration and visualization patterns; redesign the perception outputs for the local costmap approach.

**Reasoning:**  
The code demonstrates sensor subscriptions, XYZ extraction, TF use, debug outputs, and ROS message publishing. The obstacle detector omits free-space context and its image/cloud pairing is not synchronized. The model asset and Python dependency declarations need resolution before the lane node is reproducible.

**Open questions:**
- Where is `best_modelV.2.pth` stored for the intended runtime?
- Are image and point-cloud timestamps synchronized by the camera driver, and should the node enforce pairing?
- What filtering and downsampling method should be used for the final cloud?
- How will depth obstacles and RGB-detected boundaries be represented together?
- How will sloped traversable surfaces be distinguished from unsafe geometry?

**Evidence reviewed:**
- `igv_perception/igv_perception/segformer_lane_node.py`
- `igv_perception/igv_perception/obstacle_detector_node.py`
- `igv_perception/igv_perception/cloud_debug_filter_node.py`
- `igv_perception/package.xml`
- `igv_perception/setup.py`
- `igv_bringup/launch/real_lane_test.launch.py`

### `igv_tools`

**Purpose:**  
Provides simple fake publishers for testing behavior-node inputs.

**Key contents:**
- Nodes or executables:
  - `igv_tools/fake_lane_publisher.py`
  - `igv_tools/fake_obstacle_publisher.py`
- Launch files: `fake_lane_publisher` is started by `igv_bringup/launch/lane_test.launch.py`; no bringup launch starts `fake_obstacle_publisher`.
- Config, message, or robot-description files: Uses `LaneEstimate` and `ObstacleSummary`.

**Connections to other packages:**
- Subscribes to: None.
- Publishes to: `/lane_estimate` and `/obstacle_summary`.
- Depends on: `rclpy`, `igv_interfaces`.

**Current status:**  
Test utilities. The fake lane node publishes a constant valid lane estimate; the fake obstacle node publishes a constant clear-path summary.

**Recommendation:**  
Keep for controlled software tests; do not treat outputs as real sensor data.

**Reasoning:**  
Fake publishers can isolate behavior logic from camera hardware and help test message wiring. Their constant values do not exercise realistic perception changes unless extended.

**Open questions:**
- Should additional test inputs cover changing lane errors, invalid/stale messages, and obstacle appearances?
- Should a dedicated test launch include the fake obstacle publisher?

**Evidence reviewed:**
- `igv_tools/igv_tools/fake_lane_publisher.py`
- `igv_tools/igv_tools/fake_obstacle_publisher.py`
- `igv_tools/setup.py`
- `igv_bringup/launch/lane_test.launch.py`

## Cross-package findings

- `igv_bringup/launch/full_system.launch.py` starts ZED and VectorNav, robot-state publishing, behavior/control nodes, an odometry stub, and RViz, but it does not start lane perception, obstacle perception, the cloud filter, or the costmap.
- `igv_bringup/launch/real_lane_test.launch.py` is the most complete real-sensor test path in the launch group, but it depends on the missing SegFormer model asset and does not start a planner/controller.
- The costmap configuration in `igv_bringup/config/obstacle_layer_params.yaml` uses `/debug/filtered_cloud` for marking and raw ZED cloud for clearing. The separate costmap launch does not start the camera or filter.
- The current behavior path consumes lane and obstacle summaries rather than a shared spatial representation of occupied and traversable areas.
- `igv_control` copies a quantity named as steering angle directly into `Twist.angular.z`; confirm command units and vehicle kinematics.
- `igv_bringup/package.xml` does not declare runtime dependencies referenced by its launch files. `igv_perception/package.xml` does not declare several imported Python libraries.
- Backup and saved files exist alongside active files. Compare them and check version history before deciding whether they should be archived or removed.

## Overall recommendation

Retain the repository as a source of ROS 2 integration examples and historical design context. Treat the current lane behavior, obstacle summary, odometry stub, and minimal IGV robot description as prototype components that require replacement or redesign for the intended system.

The planned direction is a local navigation representation in which filtered depth obstacles and RGB-detected track boundaries constrain traversable space, with a rolling costmap and planner/controller selecting motion. Final transforms, sensor synchronization, filtering, downsampling, costmap parameters, simulation geometry, and safety behavior require implementation and testing.

## Packages not yet assessed

None in this source-level package review. Runtime behavior, build success, hardware integration, and simulation validity remain unverified.
