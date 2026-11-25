# IGV Navigation Subteam Responsibilities and Integration Plan

## Purpose

This document describes the work and interfaces for the three navigation subteams:

1. Local navigation and DWB controller
2. State estimation and IMU
3. GPS and global heading

The plan accounts for the legacy ROS 2 prototype and the intended direction for the IGV: use depth data to identify physical obstacles, use RGB data to identify track boundaries, and select a safe route through traversable space rather than follow a lane centerline.

This is an integration plan, not a claim that the existing code is ready for vehicle use. Findings about legacy code are based on repository contents; runtime behavior, sensor performance, and safety have not been validated.

## Working assumptions

- The target development environment is **ROS 2 Humble**.
- Nav2 is the expected basis for local navigation, subject to confirming package availability and compatibility in the project environment.
- A planner node is expected to generate a path or route for DWB to follow. The specific planner and its inputs remain to be selected.
- The vehicle has three ZED cameras, an IMU, and a GPS/global-heading source. Sensor topics, frames, mounting positions, and performance need to be confirmed.
- The intended local representation combines depth-obstacle information with RGB-detected track boundaries as regions to avoid.
- The final simulator has not been selected. The legacy differential-drive example targets Humble with Gazebo Classic 11, but that does not by itself establish Gazebo Classic as the project choice.

## System overview

~~~text
GPS / global heading ──> global pose, heading, and navigation goal
                                  │
                                  ▼
                     Planner node generates path
                                  │
                                  ▼
Depth obstacles ──┐      Local costmap + DWB ──> velocity command
                  ├────>        ▲
RGB boundaries ───┘             │
                       local pose and velocity
                                  ▲
                       State estimation / IMU
~~~

The team should decide whether the planner generates a global path from a map or waypoints, or whether it generates a shorter route from track geometry and local goals. DWB is the local controller; it needs a path or local target to follow.

## Subteam summary

| Subteam | Main responsibility | Principal outputs |
|---|---|---|
| Local navigation and DWB controller | Represent obstacles and boundaries locally, obtain a path from the planner, and generate motion commands | Local costmap, DWB controller output, vehicle command interface |
| State estimation and IMU | Estimate continuous local motion and orientation | Odometry, IMU data, odom → base_link transform |
| GPS and global heading | Estimate global position and heading and relate global goals to the local navigation frame | Global pose/heading, navigation goals, and possibly map → odom |

Each transform and command interface should have one agreed owner. Avoid multiple nodes publishing the same TF transform.

## 1. Local navigation and DWB controller

### Mission

The local navigation subteam turns sensor observations into a local representation of occupied and unsafe space. It receives a path from the planner node and uses DWB to generate motion commands that follow the path while responding to nearby obstacles and track boundaries.

DWB does not replace the planner. The planner is expected to generate the path or route; DWB evaluates motion locally using that path, the local costmap, and the vehicle state.

### Work to accomplish

#### Sensor inputs and geometry

- Confirm the topic names, message types, rates, frame IDs, timestamp behavior, and synchronization of all three ZED cameras.
- Measure each camera’s mounting position and orientation on the vehicle.
- Transform observations into a common vehicle frame, expected to be base_link.
- Define the field of view and sensor range needed for the vehicle’s expected speed, braking, and track dimensions.

#### Obstacles and boundaries

- Filter invalid and out-of-range depth points.
- Reduce point density with a spatial method such as voxel-grid downsampling. Select the resolution to preserve obstacles that matter to the vehicle’s footprint.
- Combine the three camera views after transforming them into the shared vehicle frame.
- Detect physical obstacles from depth data.
- Detect white track boundaries from RGB data and convert them into a representation the navigation system can use.
- Represent both obstacles and boundaries as unsafe regions.
- Preserve enough free-space information for navigation to compare routes. A list of detected obstacle points by itself may not describe where the vehicle can safely travel.
- Decide how to treat ground, ramps, and slopes so a traversable ramp is not automatically treated as an obstacle while edges and unsafe drops remain blocked.

#### Costmap and planner/DWB integration

- Configure the local costmap frame, vehicle footprint, dimensions, resolution, update rate, obstacle limits, and clearing behavior.
- Connect depth-obstacle and RGB-boundary outputs to the costmap.
- Confirm that sensor observations are transformed at appropriate timestamps.
- Select the planner node or planner framework that generates the path for DWB.
- Define the planner’s goal input, path output, coordinate frame, and replanning behavior.
- Connect the planner output to DWB and confirm that the controller follows the path while avoiding costmap obstacles and boundaries.
- Define behavior when no safe trajectory is available, such as stopping and reporting that the path is blocked.
- Connect the controller output to the vehicle command interface. Document command units, velocity limits, timeout behavior, and any command arbitration.
- Visualize sensor data, transforms, boundaries, costmap layers, path, and DWB trajectories in RViz.

### Deliverables

- A documented navigation input/output contract.
- Validated camera extrinsics and TF configuration for all three cameras.
- Depth-obstacle and RGB-boundary processing paths.
- A local costmap configuration with inspectable sensor layers.
- A planner configuration or implementation that produces a path for DWB.
- A DWB configuration matched to the vehicle’s motion limits and kinematics.
- A command interface with documented units, limits, and stale-command behavior.
- RViz views for points, boundaries, costmap, planned path, and controller output.
- Simulation scenarios covering obstacles, boundaries, narrow passages, ramps, and sensor loss.

### Legacy code to review

- igv_perception/obstacle_detector_node.py demonstrates ROS subscriptions, XYZ decoding, markers, debug images, and publishing selected point-cloud regions. Its depth-range blob detector loses free-space context and should not define the final navigation representation.
- igv_perception/cloud_debug_filter_node.py demonstrates point-cloud filtering, TF transformation, and a costmap-oriented output. It uses every-Nth-point sampling rather than spatial voxel downsampling and should be treated as exploratory.
- igv_behavior/behavior_manager.py implements lane following, stop/hold behavior, and a stopped lane-search turn. It does not plan a path around obstacles or boundaries.
- igv_control/cmd_vel_manager.py demonstrates a subscriber-to-publisher bridge, but copies target_steering_rad directly into Twist.angular.z. Confirm and document the final command units rather than inheriting this ambiguity.
- igv_bringup/config/obstacle_layer_params.yaml and igv_bringup/launch/obstacle_layer_view.launch.py are costmap examples, but their inputs, parameter path, and node naming need review before reuse.

## 2. State estimation and IMU

### Mission

The state-estimation subteam provides a continuous estimate of local vehicle motion and orientation. This lets navigation relate sensor observations, the vehicle footprint, and controller commands as the vehicle moves.

An IMU provides angular-rate and acceleration measurements and may provide an orientation estimate. It does not, by itself, provide reliable long-term position. Identify the wheel-odometry or other motion source that will complement it.

### Work to accomplish

- Identify the VectorNav topics, message types, rates, frame IDs, axis conventions, timestamps, and covariance fields.
- Confirm the IMU’s mounting orientation and define its fixed transform to base_link.
- Identify available wheel encoder, motor, or drivetrain feedback for estimating translation and rotation.
- Document units, sign conventions, calibration, bias, covariance, and filtering assumptions.
- Develop or configure an estimator that combines IMU data with wheel odometry or other local motion sources.
- Publish nav_msgs/Odometry with documented frames, timestamps, pose, velocity, and uncertainty.
- Own the odom → base_link transform unless the simulator or hardware driver is explicitly assigned that role.
- Define behavior when IMU or wheel data becomes stale or unavailable.
- In simulation, use simulated drivetrain odometry and IMU data instead of the fixed odometry stub.
- Verify that local navigation receives current pose and velocity estimates in the expected frames.

### Deliverables

- Sensor and frame specification for the IMU and local motion sensors.
- Estimator implementation or configuration.
- Documented /odom and odom → base_link contracts.
- Calibration and validation results for stationary, straight-line, turning, and ramp motion.
- A documented response to data loss or estimator failure.

### Legacy code to review

igv_localization/odom_stub.py publishes fixed parameter values on /odom and broadcasts odom → base_link at a set rate. It is useful for understanding odometry and TF publishing, and may support early visualization. It does not integrate sensor data or represent changing vehicle motion. Replace it with the actual estimator or simulator odometry before testing navigation that depends on vehicle motion.

## 3. GPS and global heading

### Mission

The GPS/global-heading subteam provides global position and heading information for navigation tasks that need a global reference, such as locating the vehicle in a world frame or relating goals to the local navigation system.

The team should identify whether heading comes from a dedicated heading-capable sensor, a fused orientation estimate, or course over ground. These sources behave differently, particularly when the vehicle is stopped or moving slowly.

### Work to accomplish

#### Global sensors and reference

- Identify the GNSS receiver, antenna arrangement, heading source, message types, update rates, frame IDs, and expected accuracy.
- Define the global coordinate reference and how GPS coordinates will be converted into a local Cartesian frame.
- Document the relationship between global heading, map axes, vehicle forward direction, and sensor axes.
- Determine what heading data is valid while stationary and at low speed.

#### Integration with local estimation and planning

- Agree with state estimation on which estimate is continuous and local, and which source provides global correction.
- Define ownership of map → odom if a map/world frame is used.
- Ensure global corrections do not create sudden local pose jumps that destabilize the controller.
- Publish global pose and heading with timestamps and uncertainty.
- Define how GPS or heading uncertainty and loss affect goals, path generation, or vehicle speed.
- Define how global goals or waypoints are converted into the planner’s goal input.
- Coordinate with local navigation on the planner’s required start pose, goal format, and frame.

### Deliverables

- GPS and heading sensor specification.
- Global-frame and coordinate-conversion specification.
- Global pose/heading interface with uncertainty and timestamps.
- A clearly assigned map → odom transform publisher, if that transform is used.
- A goal/waypoint handoff contract to the planner.
- Tests for stationary heading, low-speed motion, turns, GPS degradation, and signal recovery.

### Legacy code to review

The legacy launch files start a VectorNav driver, but the reviewed prototype does not contain a complete GPS/global-heading integration path. The odometry stub is not a substitute for GNSS localization or global heading. Confirm the intended hardware topics and frame contracts before using the existing launch as the basis for this subteam’s work.

## Cross-subteam interfaces and ownership

| Interface or transform | Proposed owner | Agreement needed |
|---|---|---|
| Camera frames → base_link | Local navigation/perception, coordinated with robot-description owner | Measured extrinsics, frame names, timestamp handling |
| IMU frame → base_link | State estimation/IMU | Mounting transform and axis conventions |
| IMU topic | State estimation/IMU | Message type, rate, frame, covariance, timestamp |
| /odom and odom → base_link | State estimation/IMU, or simulator drivetrain in simulation | One publisher at a time; pose and velocity semantics |
| Global pose and heading | GPS/global-heading | Reference frame, conversion, uncertainty |
| map → odom, if used | GPS/global-heading or an explicitly selected localization component | Single publisher and correction policy |
| Depth-obstacle and boundary inputs | Local navigation | Topic/message types, frames, range, rates, failure reporting |
| Planner goal input | GPS/global-heading supplies global goals; local navigation integrates planner | Goal type and coordinate frame |
| Planner path output → DWB | Planner component | Path message/action, frame, update/replanning behavior |
| Local costmap and DWB command | Local navigation | Footprint, limits, stop behavior, command units |
| /cmd_vel to drivetrain | Local navigation plus drivetrain/control integration | Message type, units, timeout, arbitration, emergency stop |

If Nav2’s standard map, odom, and base_link frames are used, the team should preserve the connected chain from the global frame through odometry and the vehicle base to sensor frames. Assign a single publisher to each transform.

## Integration sequence

### Stage 1: Agree on contracts

- Confirm camera, IMU, and GNSS topics and frames.
- Agree on the TF tree and transform ownership.
- Define the vehicle command interface, units, limits, and timeout behavior.
- Define the planner’s goal input and path output to DWB.

### Stage 2: Establish the simulation base

- Adapt the generic Gazebo differential-drive example to the IGV geometry and drive system if it fits the chosen simulator.
- Add the track, boundaries, obstacles, ramps, and sensors required for testing.
- Replace the fixed odometry stub with simulator odometry.
- Confirm that the simulated drivetrain responds to commands and publishes the expected odometry and TF.

The legacy model under differential_drive_robot_model/ is a generic reference. Its world contains a ground plane and sun, and the robot description does not include ZED sensors. Its README targets Humble with Gazebo Classic 11; that is evidence about the example’s intended environment, not confirmation that the project has selected Gazebo Classic.

### Stage 3: Build the local navigation path

- Feed simulated depth data into obstacle processing.
- Feed simulated RGB data into boundary detection.
- Verify that both outputs appear in the correct vehicle frame.
- Mark obstacles and boundaries in the local costmap.
- Configure a planner node to generate a path from the current pose to a navigation goal.
- Connect that path to DWB and verify that DWB follows it while responding to costmap obstacles and boundaries.

### Stage 4: Integrate local and global state

- Add state-estimated local odometry and IMU data.
- Add global position and heading interfaces.
- Test the complete TF tree and goal-to-planner handoff.
- Confirm that global corrections do not destabilize local motion or path following.

### Stage 5: Validate degraded and edge cases

- Test missing, stale, or invalid camera data.
- Test IMU, wheel-odometry, and GNSS degradation.
- Test obstacles that partly obscure boundaries.
- Test narrow passages, turns, ramps, and edges.
- Verify the agreed stop or fault response when no safe path or reliable state estimate is available.

## Completion criteria

The navigation system is ready for integrated simulation testing when:

- Each subteam has documented topics, message types, units, frames, rates, timestamps, and uncertainty.
- The TF tree is connected and every transform has one authoritative publisher.
- The selected Humble environment can launch the required packages.
- The simulator produces the required sensor data and vehicle odometry.
- Depth obstacles and track boundaries appear in the local costmap in the correct locations.
- The planner generates a path from the agreed goal input.
- DWB follows that path while avoiding unsafe costmap regions.
- The drivetrain interprets the controller commands using the intended units and limits.
- Loss or staleness of critical perception or state data leads to the agreed safe response.
- RViz displays the robot, sensor data, transforms, costmap, planned path, and controller output for review.

## Open decisions

- Which planner node or Nav2 planner plugin will generate the path for DWB?
- What will the planner use as its map, track representation, or goal source?
- What simulator and ROS 2 Humble-compatible sensor plugins will be used?
- Which localization component owns map → odom, if that transform is needed?
- What are the final sensor frames, camera extrinsics, vehicle footprint, and motion limits?
- What command type will the physical drivetrain accept, and who owns the final conversion and watchdog?

## Reference note

Use ROS 2 Humble-compatible Nav2 documentation and package versions when selecting planner, controller, costmap, and launch configuration. Newer ROS 2 documentation may describe parameter conventions that differ from Humble.

