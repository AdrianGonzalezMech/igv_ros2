# IGV Vision Subteam Responsibilities and Integration Plan

## Purpose

This document describes the vision subteam’s responsibilities and integration points with local navigation. The vision subteam is responsible for bringing up and configuring the ZED cameras, integrating the multi-camera arrangement, and producing perception outputs that local navigation can use.

The current design baseline discussed for the IGV is three ZED 2i cameras, each with approximately 110 degrees of horizontal field of view, arranged to provide roughly 220–240 degrees of coverage relative to the front of the vehicle. The final mounting positions, overlap, and usable coverage must be verified on the vehicle.

This plan reflects the intended ROS 2 Humble environment and the project’s local-navigation direction: depth data identifies physical obstacles, RGB data identifies track boundaries, and local navigation uses those outputs to represent unsafe regions and select a route.

This is a planning document. It does not claim that the existing perception code has been validated on the final cameras or vehicle.

## Vision subteam mission

Provide reliable, time-stamped, correctly framed camera data and navigation-oriented perception outputs for the IGV:

1. Bring up and configure each ZED camera in ROS 2 Humble.
2. Calibrate and document the camera mounting geometry and frame relationships.
3. Integrate the three camera views into a consistent sensor system.
4. Produce a filtered obstacle point cloud from depth data.
5. Produce a navigation-usable representation of track boundaries from RGB data, or coordinate and document who owns that detector.
6. Work with local navigation to define topics, message semantics, coordinate frames, update rates, and behavior when camera data is missing or stale.

The ZED driver supplies camera images, depth, and point-cloud data. The vision subteam’s central integration work is to configure the drivers, validate their data, transform and combine the camera observations, and produce outputs suited to navigation. Reimplementing stereo depth generation is not the goal unless the selected driver cannot provide the required data.

## Interfaces with local navigation

The vision and local-navigation subteams should agree on these contracts before integrating the detector output:

| Interface | Vision subteam responsibility | Local navigation responsibility |
|---|---|---|
| Camera topics | Configure and document RGB, depth, point-cloud, and camera-info topics for each camera | Identify required inputs and confirm compatibility |
| Camera frames and timestamps | Publish correct frame IDs and measurement timestamps; document extrinsics | Define target frame and required transform timing |
| Combined obstacle data | Filter, transform, and combine camera depth observations into an agreed point-cloud output | Decide how it marks obstacles and clears observed free space in the costmap |
| Track-boundary detections | Detect boundaries from RGB and provide detections in an agreed form, if assigned to vision | Interpret boundary output as unsafe regions in the local costmap |
| Confidence and health | Provide relevant confidence/validity information and report missing or stale sensor data | Define controller behavior when data is invalid, stale, or unavailable |
| Debugging outputs | Provide images, masks, point clouds, and markers that expose perception decisions | Visualize the outputs with costmap and planner data in RViz |

Possible topic names, such as a combined obstacle cloud or a boundary-detection topic, are examples only. The teams should agree on names and message types rather than treating these as established interfaces.

## Work to accomplish

### 1. Camera bringup and configuration

- Confirm the exact ZED 2i units, serial numbers, mounting positions, viewing directions, and available device connections.
- Configure the ROS 2 Humble ZED wrapper for all three cameras, including unique namespaces, topic names, and frame names.
- Document each camera’s image, camera-info, depth, and point-cloud topics.
- Agree on image resolution, frame rate, depth mode, depth range, confidence settings, and any required point-cloud settings.
- Check resource constraints such as USB bandwidth, compute load, memory, and thermal behavior while all cameras operate together.
- Ensure launch files can start the three cameras consistently and that each camera can be identified in logs and RViz.

### 2. Camera mounting, calibration, and transforms

- Measure each camera’s position and orientation relative to the vehicle.
- Define the TF relationship from each camera frame to base_link.
- Confirm that camera optical frames and vehicle frames use the expected axis conventions.
- Validate transforms visually and numerically using RViz and known test points or targets.
- Record the final extrinsics, calibration settings, frame names, and measurement procedure in project documentation.
- Recheck the extrinsics after physical mounting changes.

The legacy robot description uses a single zed_camera_link, while the ZED driver and RViz configuration refer to camera-specific frames such as zed_left_camera_frame. The final three-camera system will need explicit, verified frame connections for all camera units.

### 3. Multi-camera integration

- Validate each camera independently before combining outputs.
- Transform each camera’s observations into the common vehicle frame using the transform valid at the observation timestamp.
- Decide how to handle overlap between cameras, duplicate observations, different update rates, and temporary disagreement between depth measurements.
- Combine observations into an agreed output while preserving timestamps and the common frame ID.
- Confirm that the integrated system provides the intended forward and side coverage, with no unexpected blind regions caused by camera placement or filtering.
- Test the camera group on the real vehicle, not only from individual camera specifications.

The nominal field of view of the cameras does not by itself guarantee the desired vehicle coverage. Mounting angles, the vehicle body, occlusion, lens behavior, camera overlap, and valid depth range all affect usable coverage.

### 4. Depth obstacle point cloud

- Start from the ZED point-cloud or depth output and confirm its coordinate frame, organization, units, timestamp, and invalid-point representation.
- Filter invalid or low-confidence depth measurements according to the selected camera configuration.
- Apply a range of interest appropriate for the vehicle and local-navigation update loop.
- Reduce point density with a spatial method, such as voxel-grid filtering, if needed to manage bandwidth and processing cost while retaining relevant obstacle shape.
- Transform the filtered observations into the common vehicle frame before fusion.
- Produce a documented, navigation-oriented obstacle output for local navigation.
- Preserve access to raw or minimally processed camera data for debugging and comparison.
- Provide visualization of the raw cloud, filtered cloud, and combined cloud.

A filtered obstacle cloud and a complete point cloud do not have identical meaning. A cloud containing only detected obstacle points can support costmap marking, but it may not show which areas were observed and found clear. Coordinate with local navigation on whether separate camera or depth observations are needed for costmap clearing and free-space reasoning.

Do not remove ground or ramp points using an unvalidated flat-ground assumption. Test the selected filtering against the project’s ramps and slopes so a traversable surface is not automatically reported as an obstacle.

### 5. RGB track-boundary perception

- Agree whether the vision subteam owns the RGB boundary detector or whether it is assigned elsewhere.
- If owned by vision, define how the white boundaries are detected under expected lighting, shadows, glare, surface changes, and partial occlusion.
- Publish boundary detections in a representation local navigation can transform and use, such as projected points, a boundary point cloud, or another agreed message.
- Include timestamps, coordinate frames, validity, and confidence or quality indicators where available.
- Preserve useful debug outputs such as the RGB image, segmentation mask, and projected boundary detections.
- Test continuous and dashed lines and track boundaries on ramps or bridges.

Vision should provide the detected boundary geometry and its quality information. Local navigation should decide how that geometry is converted into costmap costs and how the planner/controller treats it.

### 6. Synchronization and data health

- Measure the update rates and timestamp behavior of each camera stream.
- Decide how the system pairs RGB and depth data for each camera.
- Reject or flag stale data rather than silently pairing unrelated measurements.
- Define what happens when one camera disconnects, publishes invalid depth, or falls behind the others.
- Report whether a combined output has partial camera coverage or a complete sensor set.
- Confirm QoS settings and message transport behavior in the actual Humble deployment environment.

### 7. Integration and visualization

- Provide a launch configuration for the three-camera group with consistent namespaces and parameters.
- Publish the required static or robot-description transforms and verify them against the vehicle.
- Provide RViz displays for each camera, the combined cloud, obstacle output, boundary output, and relevant TF frames.
- Work with local navigation to show the perception layers together with the local costmap and planner path.
- Document startup requirements, camera identification, expected topics, parameter meanings, and common failure symptoms.

## Legacy code to review

- igv_perception/obstacle_detector_node.py subscribes to one RGB image and one registered point cloud, thresholds nearby Z values, groups image regions, samples points from accepted regions, and publishes debug outputs and a compact obstacle summary. It is useful as a reference for ROS image/cloud handling, but it does not fuse three cameras, preserve free-space context, enforce RGB/cloud synchronization, or produce a complete navigation representation.
- igv_perception/cloud_debug_filter_node.py demonstrates point-cloud filtering and TF transformation into base_link. It samples every Nth point rather than using spatial voxel filtering and is configured for a single ZED input.
- igv_perception/segformer_lane_node.py demonstrates learned image segmentation and publishes LaneEstimate plus debug outputs. It reports a fixed lane width and does not currently implement stop-bar detection. The uploaded repository snapshot does not contain the model file it attempts to load, so the node requires that asset to be supplied separately.
- igv_bringup/launch/real_lane_test.launch.py starts the single-camera lane detector, obstacle detector, and cloud filter. It can help identify existing topic names and bringup patterns, but it is not a three-camera integration launch.
- igv_description/urdf/igv.urdf provides a single camera link with a placeholder pose. It should not be treated as the calibrated three-camera transform model.

## Deliverables

The vision subteam should provide:

1. A camera configuration and launch setup for all three ZED cameras in ROS 2 Humble.
2. A topic/frame inventory for each camera’s RGB, camera-info, depth, and point-cloud outputs.
3. Measured camera extrinsics from each camera frame to base_link.
4. A tested multi-camera transform and fusion process.
5. A filtered obstacle point-cloud output with documented frame, timestamp, range, filtering, and sampling behavior.
6. A boundary-detection output or a documented ownership handoff if boundary detection belongs to another team.
7. Debug and visualization outputs sufficient for local navigation to inspect the sensor data and detections.
8. A sensor-health and stale-data policy agreed with local navigation.
9. Test results for individual cameras and the combined camera system under expected track conditions.

## Suggested integration sequence

### Stage 1: Agree on interfaces

- Confirm Humble-compatible ZED driver packages and configuration.
- Agree on camera namespaces, topic names, frame IDs, and timestamp policy.
- Agree on the combined obstacle-cloud format and boundary-detection output.
- Agree with local navigation how free-space clearing information will be represented.

### Stage 2: Bring up cameras individually

- Start and inspect each camera by itself.
- Confirm RGB, camera-info, depth, point cloud, frame IDs, timestamps, and expected rates.
- Check depth validity and range on representative obstacles and surfaces.

### Stage 3: Validate mounting and transforms

- Measure the physical mounting of all cameras.
- Publish and visualize camera-to-vehicle transforms.
- Verify common-frame point locations using known targets and vehicle geometry.

### Stage 4: Integrate and filter camera observations

- Transform each cloud into the common vehicle frame.
- Apply agreed validity, confidence, and range filtering.
- Evaluate spatial downsampling and its effect on small obstacles.
- Combine camera observations and inspect overlap, blind areas, and duplicate measurements.

### Stage 5: Produce navigation outputs

- Publish the obstacle cloud and boundary detections with agreed semantics.
- Confirm outputs are usable by the local costmap and planner/controller pipeline.
- Test stale data, camera loss, and partial sensor coverage.

### Stage 6: Validate in simulation and on the vehicle

- Use Humble-compatible simulated sensors if available in the selected simulator.
- Compare simulated outputs with real camera behavior.
- Test track boundaries, obstacles, ramps, bridges, lighting changes, and camera occlusion.
- Record limitations and supported operating conditions.

## Completion criteria

The vision package is ready for integration testing when:

- All three cameras launch consistently in the Humble environment.
- Each camera’s topics, frames, timestamps, and parameters are documented.
- Camera-to-vehicle transforms are measured and visually validated.
- Combined point-cloud data uses a common frame and preserves correct timestamps.
- Filtering and downsampling retain relevant obstacles at the chosen operating range.
- Local navigation can distinguish obstacle marking data from data used for free-space clearing.
- Boundary detections are available in the agreed format, or ownership is explicitly assigned.
- Invalid, stale, disconnected, or partial camera data is reported according to the agreed policy.
- RViz can show individual camera data, the fused cloud, obstacle output, boundaries, and TF.
- Results have been reviewed with local navigation using the planner and costmap integration path.

## Open decisions

- What are the final physical locations, orientations, and serial-number assignments for the three cameras?
- Which exact ROS 2 Humble ZED wrapper version and configuration will be used?
- What are the agreed topics and message types for the combined obstacle cloud and boundary detections?
- Which component will provide costmap clearing or observed-free-space information?
- Does the vision subteam own RGB boundary detection, or will another team provide it?
- What range, confidence, and downsampling settings preserve the smallest obstacle that matters to the vehicle?
- What output should local navigation receive when one camera is unavailable?
- Which simulator and simulated camera implementation will be used?

