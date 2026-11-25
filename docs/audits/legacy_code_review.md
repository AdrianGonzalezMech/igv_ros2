# Legacy Code Audit: IGV ROS 2 Prototype

## Purpose and scope

This audit reviews the previous team’s ROS 2 prototype to identify what is useful as a reference, what is only a test stub, and what would need substantial work before supporting the current IGV navigation direction.

These findings describe the code in the repository snapshot. They do not confirm that the nodes build, run together, or are safe for vehicle operation. The future direction discussed for the IGV is local navigation using obstacle and track-boundary information to select traversable space, rather than following a lane centerline.

## High-level findings

- The repository contains useful ROS 2 examples for sensor subscriptions, message publishing, point-cloud handling, TF, RViz visualization, and a simple Nav2 costmap configuration.
- The current behavior node implements lane following and stop/hold behavior. It does not plan a route around obstacles or boundaries.
- The obstacle detector selects nearby depth-image regions and publishes points from those regions. It does not provide a complete scene representation or free-space map.
- A separate point-cloud filter and costmap configuration suggest an exploratory Nav2 path, but the main launch does not start that path.
- The robot description under `igv_description` is a minimal visual model. The separate differential-drive model is a more complete Gazebo example, but it is not an IGV-specific simulation.
- Several launch files and saved source variants reflect different stages of experimentation. Their names and contents should be considered before cleanup.

## Perception package: `igv_perception`

### Obstacle detector

Primary file: `igv_perception/obstacle_detector_node.py`.

#### Current data flow

1. The node subscribes to the ZED rectified RGB image (`/zed/zed_node/rgb/color/rect/image`) and registered point cloud (`/zed/zed_node/point_cloud/cloud_registered`).
2. `cloud_to_xyz_arrays()` decodes the `PointCloud2` fields into three aligned 2D arrays. `x[row, col]`, `y[row, col]`, and `z[row, col]` give the coordinates for the point at that organized-cloud pixel. Rows and columns are image-grid indices, not physical X and Y coordinates.
3. On each image callback, the node thresholds cloud Z values to 0.2–1.524 m. It optionally removes the upper part of the image and applies OpenCV morphology to create a binary mask. In this implementation, the mask is based on depth values, not RGB color or semantic object recognition.
4. The node finds connected image regions as contours and rejects regions below `min_box_area`. It maps each remaining rectangle into point-cloud array indices, selects valid XYZ points within the rectangle, and samples those points using `point_sample_step`.
5. It publishes sampled points on `/obstacles/points`, an obstacle marker, debug images, and an `ObstacleSummary`.

#### Useful concepts

- ROS 2 subscriptions for images and `PointCloud2`.
- `CvBridge` conversion between ROS image messages and OpenCV arrays.
- Decoding XYZ fields from an organized point cloud while preserving its grid layout.
- Mapping a selected image-grid region to corresponding 3D points, when the image and cloud are calibrated, aligned, and synchronized.
- Publishing point clouds, markers, and debug images for visualization.
- Using a compact summary message to communicate detections to another node.

#### Limitations

- **Depth-range blob detector, not semantic obstacle detection:** It groups nearby depth pixels but does not identify object classes or decide whether a surface is safe to traverse.
- **Reduced scene context:** `/obstacles/points` contains sampled points only from accepted blobs. It omits surrounding points and does not explicitly represent free space.
- **No image/cloud synchronization:** The image callback uses the latest cached cloud without matching timestamps or rejecting stale clouds. A time difference between streams could cause incorrect image-to-cloud region mapping.
- **Assumed grid correspondence:** If image and cloud dimensions differ, the node resizes the Z array for mask creation and proportionally maps rectangle bounds back to the cloud grid. This assumes compatible fields of view and registration.
- **Reported distance is median Z:** The reported obstacle distance is the median Z value of the valid points in a blob. It is not the nearest point’s range or full Euclidean distance from the camera.
- **No transform to `base_link`:** The output points and summary use the incoming cloud’s frame ID. This node does not transform them into `base_link`.
- **Summary flags do not reflect obstacle position:** Any qualifying blob sets `obstacle_ahead=true`, `lane_blocked=true`, and both left/right-open flags false, regardless of bearing or lateral position.
- **Legacy parameters are unused in this path:** `box_frame`, `min_x/max_x`, `min_y/max_y`, `min_z/max_z`, and `min_points` are declared for compatibility but are not used by the custom detector code.
- **Detection range and stop threshold:** The detector accepts Z through 1.524 m; the behavior node stops for reported distances under 3 m. Since the reported median Z cannot exceed the detector’s upper range, any accepted detection satisfies the behavior node’s stop-distance condition.

### Lane detector

Primary file: `igv_perception/segformer_lane_node.py`.

The node uses a SegFormer model to produce a lane mask from camera imagery, calculates lane-related estimates, and publishes `LaneEstimate` on `/lane_estimate`. It also publishes lane point-cloud and debug-image topics.

The current estimate is used by `igv_behavior/behavior_manager.py` for lane following. The lane width is set to a fixed value of 3.0 m, and the stop-bar fields are always published as “not detected” with zero distance. The lane estimate therefore does not currently provide measured lane width or stop-bar detection.

The node loads `models/best_modelV.2.pth` during startup. No `.pth` or `.pt` model file is present in the uploaded repository snapshot. The real-sensor lane launch depends on that model being supplied separately at the expected installed package path.

### Related point-cloud filter

File: `igv_perception/cloud_debug_filter_node.py`.

This is a separate processing path and is closer to a costmap input prototype:

- It subscribes to the registered ZED point cloud.
- It skips every Nth serialized point according to `sample_step` (default 3), ignores NaNs, applies a 3D range limit from the cloud’s source origin, transforms retained points into the configured target frame (default `base_link`), and filters by target-frame Z height.
- It publishes `/debug/filtered_cloud`.
- The transform lookup requests the latest available transform rather than explicitly using the cloud message timestamp. If the vehicle or sensor moves, this timing choice should be reviewed.
- This is **uniform point skipping**, not voxel-grid downsampling or obstacle clustering. The resulting sample density depends on the cloud’s organization and resolution.

Treat this filter as exploratory. Its transform timing, range and height limits, sampling behavior, costmap clearing configuration, and runtime wiring require validation before reuse.

## Behavior package: `igv_behavior`

Primary file: `igv_behavior/behavior_manager.py`.

The behavior node subscribes to `/lane_estimate` and `/obstacle_summary`, and publishes `BehaviorState` on `/behavior_state` at 10 Hz.

Its current behavior is:

1. If `obstacle_ahead` is true and the reported distance is under 3 m, publish a stop command.
2. After a reported obstacle clears, hold the stop for 0.75 seconds.
3. If a fresh, valid lane estimate is available, steer using weighted lateral and heading errors, with speed reduced as the requested turn value grows.
4. If lane data is invalid or stale, publish zero linear speed and a small positive turn request in `SEARCH_LANE`.

The search behavior asks for a turn while stopped. Whether that causes physical rotation depends on the downstream controller and vehicle.

This is lane-following logic with a simple obstacle stop. It does not choose a path around obstacles or boundaries, and it does not use the obstacle summary’s lane-open fields. There is also no obstacle-message freshness check: behavior depends on the last received obstacle state until another message arrives.

## Control package: `igv_control`

Primary file: `igv_control/cmd_vel_manager.py`.

This node subscribes to `BehaviorState`, copies its speed and turn fields, and publishes `geometry_msgs/Twist` on `/cmd_vel` every 0.1 seconds. This is a useful example of a ROS subscriber-to-publisher bridge.

Two limitations need review:

- It retains and republishes the last command indefinitely if `/behavior_state` stops publishing; there is no command timeout or watchdog.
- `BehaviorState.msg` names the turn field `target_steering_rad`, while this node copies it directly into `Twist.angular.z`, conventionally an angular velocity in rad/s. Confirm the intended units and vehicle kinematics before reusing this interface.

## Localization package: `igv_localization`

Primary file: `igv_localization/odom_stub.py`. Parameters: `igv_localization/config/odom_stub.yaml`.

The stub publishes `/odom` and an `odom → base_link` transform at a configured rate using fixed parameter values. It does not estimate motion from sensor data, wheel feedback, or `/cmd_vel`. It can provide a placeholder transform for visualization or early integration, but it is not vehicle localization.

If a simulator or another localization source also publishes odometry or the same transform, only one source should own that transform.

## Interfaces package: `igv_interfaces`

Message definitions are under `igv_interfaces/msg/`:

- `LaneEstimate.msg` carries lane validity, confidence, lateral and heading errors, width, and stop-bar fields.
- `ObstacleSummary.msg` carries obstacle status, distance, bearing, and lane-open flags.
- `BehaviorState.msg` carries mode/state/reason and target speed/turn values.

These messages document the current perception-to-behavior data flow. Their semantics should be reviewed against the planned costmap-based navigation architecture, especially the obstacle summary fields and the turn-command units.

## Bringup package: `igv_bringup`

### Launch files

The launch files are different startup configurations, not interchangeable versions of one complete system:

| Package-relative launch file | Main purpose and nodes |
|---|---|
| `igv_bringup/launch/full_system.launch.py` | Starts ZED and VectorNav drivers, `robot_state_publisher`, behavior manager, odometry stub, command manager, and RViz. It does not start lane perception, obstacle perception, the cloud filter, or the costmap. |
| `igv_bringup/launch/camera_only.launch.py` | Starts the ZED driver, `robot_state_publisher`, behavior and command nodes, odometry stub, and RViz. Despite its name, it starts more than the camera; it does not start perception nodes. |
| `igv_bringup/launch/lane_test.launch.py` | Starts the ZED driver and behavior/control nodes, plus `fake_lane_publisher`. It tests behavior using synthetic lane estimates rather than camera-derived estimates. |
| `igv_bringup/launch/real_lane_test.launch.py` | Starts the ZED driver, behavior/control nodes, lane detector, obstacle detector, point-cloud filter, and RViz. This is the most complete real-sensor test path in this launch group, subject to the lane model asset being available. |
| `igv_bringup/launch/obstacle_layer_view.launch.py` | Starts a Nav2 costmap node and lifecycle manager. It does not start the camera or point-cloud filter; those inputs must be provided by another process. |

`full_system.launch.py` starts the sensors and behavior nodes, but without perception publishers the behavior node does not receive fresh lane or obstacle estimates.

The costmap preview is not connected to a planner or controller in these launch files. Its launch file uses a hard-coded path under `~/ros2_ws/src/igv_bringup/config/obstacle_layer_params.yaml`. The costmap node is launched as `/local_costmap/local_costmap`, while the YAML parameters are keyed under `/costmap/costmap`; verify that the parameters are applied to the launched node.

`igv_bringup/setup.py` installs launch files and RViz configurations, but not the `config` directory. The costmap preview currently refers directly to a source-tree path instead of a package-installed configuration path.

### Costmap configuration

File: `igv_bringup/config/obstacle_layer_params.yaml`.

The configuration describes a rolling 12 m by 12 m costmap with 0.05 m resolution, an obstacle layer, and an inflation layer. It configures `/debug/filtered_cloud` as a marking source and the raw ZED cloud as a clearing source.

This provides a useful example of how sensor observations can mark occupied cells and how ray tracing can clear observed space. The configuration alone does not start its input nodes or create a complete navigation system. The main launch does not start this costmap path.

### RViz

In `igv_bringup/launch/full_system.launch.py`, the RViz node opens `igv_bringup/rviz/igv_default.rviz`. RViz is a human-facing ROS visualization tool for data published by nodes, such as robot models, transforms, point clouds, and markers. It does not simulate the environment, process perception data, plan routes, or control the vehicle.

When Gazebo is running, RViz can display ROS sensor data and other published representations from the simulated world for inspection.

## Robot description and simulation

### IGV visual model

File: `igv_description/urdf/igv.urdf`.

This URDF defines a minimal visual model with `base_link`, `imu_link`, and `zed_camera_link`. Fixed joints attach the sensor links to `base_link`. It contains simple visual boxes but no collision or inertial properties, wheel links, or drive joints. It is useful for displaying the robot and sensor frames in RViz, but it is not a complete physical model for Gazebo simulation.

The URDF names the camera frame `zed_camera_link`, while the RViz configuration uses `zed_left_camera_frame` as its fixed frame. The ZED driver may publish a transform connecting these frames; verify the live TF tree and confirm the URDF sensor pose against the actual mounting before relying on transformed sensor coordinates.

### Separate differential-drive Gazebo model

The separate model under `differential_drive_robot_model/Differential_drive_robot-Model/` is a more complete generic differential-drive Gazebo example:

- `differential_drive_robot_description/urdf/differential_drive_robot.xacro` defines chassis, wheel, caster, collision, and inertial elements.
- `differential_drive_robot_description/urdf/differential_drive_robot.gazebo` configures a Gazebo Classic differential-drive plugin that consumes `/cmd_vel` and publishes odometry.
- `differential_drive_robot_gazebo/launch/gazebo_world_visualize.launch.py` starts Gazebo and spawns the robot.
- `differential_drive_robot_gazebo/world/world.world` contains a ground plane and sun.

The `/cmd_vel` message type is compatible with `igv_control/igv_control/cmd_vel_manager.py`, but the simulation launch does not start the IGV perception or behavior nodes. The model has no ZED sensor plugin and does not represent the final IGV geometry or sensor configuration. It is useful as a reference for Gazebo setup and differential-drive control, not as a complete IGV simulation.

## Tools and saved variants

### Fake publishers

- `igv_tools/igv_tools/fake_lane_publisher.py` publishes a constant valid lane estimate and is used by `igv_bringup/launch/lane_test.launch.py`.
- `igv_tools/igv_tools/fake_obstacle_publisher.py` publishes a constant clear-path obstacle summary. It is registered as an executable, but the bringup launch files in this snapshot do not start it.

### Saved and backup source files

The repository contains saved or alternate variants, including:

- `igv_perception/igv_perception/segformer_lane_node.py.save`
- `igv_perception/igv_perception/segformer_lane_node_before_debug_upgrade.py`
- `igv_perception/igv_perception/segformer_lane_node_debug_backup.py`
- `igv_perception/igv_perception/obstacle_detector_node_before_custom_depth_boxes.py`
- `igv_bringup/launch/real_lane_test.launch.py.save`
- `igv_description/urdf/igv.urdf.save`

The normal setup entry points and launch-file installation select the named active files, not these saved variants. The backups may document earlier approaches. Compare their contents and consult version history before deciding whether to remove them.

## Dependencies and model assets

`igv_perception/igv_perception/segformer_lane_node.py` imports PyTorch, torchvision, Transformers, Pillow, NumPy, and OpenCV. These Python runtime libraries are not declared in `igv_perception/package.xml`. The package setup installs model files matching `models/*.pth`, but no `.pth` or `.pt` model file is present in the uploaded repository snapshot. The lane detector therefore depends on a model asset supplied separately at the expected installed package path.

`igv_bringup/package.xml` has no runtime dependencies declared, despite launch files referring to packages such as `zed_wrapper`, `vectornav`, `robot_state_publisher`, and `rviz2`. This limits dependency installation and environment reproducibility.

## Direction for the current IGV navigation design

The intended design discussed for this project is a 2D local obstacle representation:

1. Filter invalid and out-of-range sensor data.
2. Transform observations into `base_link`.
3. Reduce point density using a spatial method such as a configured voxel filter.
4. Combine observations from the three camera views.
5. Represent depth obstacles and RGB-detected track boundaries as unsafe regions in a shared local navigation representation.
6. Feed obstacle information to a rolling local costmap and use a planner/controller to select and follow a safe path.

This differs from the current lane-following behavior and from the legacy blob detector, which reports selected nearby points and a coarse stop summary. The range, voxel size, ground and slope handling, boundary representation, and costmap settings require vehicle geometry and sensor testing; this audit does not prescribe final values.

## Audit status

The repository has useful examples of ROS 2 plumbing, visualization, point-cloud handling, and a generic differential-drive simulation. The current navigation behavior and perception outputs are prototype components, not a complete local navigation stack. The main integration gaps are sensor-to-navigation representations, validated transforms and timing, a working model asset and reproducible dependencies, a costmap connected to a planner/controller, and an IGV-specific simulation.

No code or configuration changes are made by this audit draft.
