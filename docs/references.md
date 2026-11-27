# ROS 2 Humble and Nav2 Documentation References

This page collects the primary documentation for the IGV AutoNav software. The project targets **ROS 2 Humble** and **Nav2 Humble**. Use Humble-specific instructions and APIs when configuring or implementing the system; newer ROS 2 releases may differ.

## ROS 2 Humble

- [ROS 2 Humble documentation](https://docs.ros.org/en/humble/) — distribution-specific documentation landing page.
- [ROS 2 tutorials](https://docs.ros.org/en/humble/Tutorials.html) — beginner through intermediate tutorials, including nodes, topics, services, actions, parameters, launch, and tools.
- [ROS 2 packages for Humble](https://docs.ros.org/en/humble/p/) — API documentation for common ROS packages.
- [ROS 2 command line tools](https://docs.ros.org/en/humble/How-To-Guides/ROS-2-Command-Line-Tools.html) — inspect nodes, topics, services, and parameters from a terminal.

## Nav2 Humble

- [Nav2 Humble API documentation](https://api.nav2.org/nav2-humble/html/index.html) — API and package documentation generated for the Humble branch.
- [Nav2 Humble source repository](https://github.com/ros-navigation/navigation2/tree/humble) — source code and package configuration for the Humble release.
- [DWB controller](https://api.nav2.org/nav2-humble/html/md_nav2_dwb_controller_README.html) — trajectory generation and critic-based local controller used by the Nav2 Controller Server.
- [Nav2 waypoint follower](https://api.nav2.org/nav2-humble/html/md_nav2_waypoint_follower_README.html) — accepts an ordered list of map-frame poses and navigates them sequentially.
- [Nav2 action APIs for Humble](https://api.nav2.org/actions/humble/index.html) — actions including `NavigateToPose`, `FollowWaypoints`, `NavigateThroughPoses`, and `FollowPath`.
- [Nav2 controller server](https://api.nav2.org/nav2-humble/html/md_nav2_controller_README.html) — controller server action interface and role in path execution.

### GPS waypoint note

The Nav2 Humble waypoint follower accepts poses in the navigation frame; Humble does not provide the later `follow_gps_waypoints` action server. The IGV software will therefore need a GPS-to-map conversion/integration component that turns each competition GPS waypoint into a map-frame pose and submits the ordered poses to Nav2.

## Localization, coordinate frames, and messages

- [Nav2 state estimation concepts](https://docs.nav2.org/getting_started/navigation_concepts/state_estimation.html) — expected `map`, `odom`, and `base_link` transform relationships and the roles of localization and odometry.
- [`robot_localization` Humble branch](https://github.com/cra-ros-pkg/robot_localization/tree/humble-devel) — EKF/UKF state estimation package source for ROS 2 Humble.
- [`robot_localization` GPS transform documentation](https://github.com/cra-ros-pkg/robot_localization/blob/humble-devel/doc/navsat_transform_node.rst) — transforms GPS fixes into the robot’s world frame using odometry and heading inputs.
- [`nav_msgs`](https://docs.ros.org/en/humble/p/nav_msgs/) — navigation messages such as `Odometry`, `Path`, and `OccupancyGrid`.
- [`sensor_msgs`](https://docs.ros.org/en/humble/p/sensor_msgs/) — sensor messages, including `Imu`, `NavSatFix`, and `PointCloud2`.
- [`geometry_msgs`](https://docs.ros.org/en/humble/p/geometry_msgs/) — geometric messages, including `Twist` and `PoseStamped`.
- [`tf2_ros`](https://docs.ros.org/en/humble/p/tf2_ros/) — publishing and consuming coordinate transforms.

## Perception and costmaps

- [Nav2 costmap and environmental representation](https://docs.nav2.org/getting_started/navigation_concepts/index.html) — overview of costmaps and how navigation represents sensor-observed obstacles.
- [Nav2 obstacle layer configuration](https://docs.nav2.org/configuration/packages/costmap-plugins/obstacle.html) — obstacle layer input sources, including `PointCloud2`; verify parameters against the Humble branch before use.
- [Nav2 Humble costmap source](https://github.com/ros-navigation/navigation2/tree/humble/nav2_costmap_2d) — release-matched implementation and configuration references.

The vision/navigation interface is currently planned as one unified obstacle point cloud. Boundary lines must remain represented as obstacles; ramp-surface handling will be defined by the depth-based ramp detection design.

## Simulation and visualization

- [Gazebo Classic with ROS 2](https://github.com/ros-simulation/gazebo_ros_pkgs/tree/ros2) — ROS 2 integration packages for Gazebo Classic. Confirm the selected Gazebo version and Humble compatibility when setting up simulation.
- [Gazebo Classic tutorials](https://classic.gazebosim.org/tutorials) — world, model, sensor, and plugin concepts for Gazebo Classic.
- [RViz user guide](https://docs.ros.org/en/humble/Tutorials/Intermediate/RViz/RViz-User-Guide/RViz-User-Guide.html) — visualize transforms, point clouds, robot models, maps, paths, and sensor data.
- [URDF and robot state publishing tutorial](https://docs.ros.org/en/humble/Tutorials/Intermediate/URDF/URDF-Main.html) — robot description and visualization setup.

## Vehicle command and wheel interface

- [ROS 2 `Twist` message](https://docs.ros.org/en/humble/p/geometry_msgs/interfaces/msg/Twist.html) — linear and angular velocity command fields.
- [ROS 2 control documentation](https://control.ros.org/humble/index.html) — controller and hardware-interface framework, useful background for the base drive interface.

The planned command chain is **Nav2 DWB → `cmd_vel` → Arduino → left/right wheel speed commands**. The Arduino handles the vehicle-specific wheel conversion and motor interface; exact conversion parameters depend on the final drivetrain geometry and motor controller.

## Competition design input

- [IGVC 2026 competition rules PDF](https://www.igvc.org/rules.htm) — official competition rules. The team is targeting AutoNav only. Keep the applicable limits and requirements visible while setting measurable software requirements and test targets.
