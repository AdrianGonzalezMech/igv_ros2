# Development Workflow and Legacy Migration Plan

## Purpose

This document describes how the team will move from the legacy ROS 2 package structure toward the IGV system planned for ROS 2 Humble. It records migration steps, expected work products, and provisional decisions about legacy packages.

This is a working plan. Package actions and ownership must be confirmed by the relevant subteams before implementation. The legacy audit describes the uploaded source snapshot; it does not guarantee that a referenced file still exists in the changing fork.

## Working principles

- Use the audit as a map of the legacy code, not as a requirement to preserve its structure.
- Agree on ROS interfaces before multiple teams implement connected components.
- Prefer small, reviewable changes on feature branches and merge through pull requests.
- Preserve useful behavior and evidence until a replacement is demonstrated.
- Do not remove a package or node until its dependents, launch references, configuration, and required assets are accounted for.
- Label documentation statements as **legacy**, **current**, **planned**, or **verified** when the distinction matters.
- Keep package and file references package-relative so they remain understandable when repository folders move.

## Target direction

The planned system has the following responsibilities:

- **Vision:** configure the ZED 2i camera group, integrate camera data, and provide navigation-usable obstacle point-cloud output. RGB track-boundary detection must have a documented owner and agreed output.
- **Local navigation:** maintain the local obstacle representation/costmap, receive a path from a planner node, run DWB as the local controller, and connect its velocity command to the vehicle interface.
- **State estimation and IMU:** provide local pose, velocity, and the agreed odometry transform using IMU and available motion measurements.
- **GPS and global heading:** provide global position/heading inputs and the agreed global-frame relationship or goal interface.
- **Simulation and robot description:** provide a model and test environment that represent the IGV closely enough to exercise the navigation interfaces.

The use of ROS 2 Humble is the project baseline. Nav2 is the expected navigation basis, and a planner node is expected to provide a path to DWB; confirm the exact planner, package boundaries, and simulator before treating them as final decisions.

## Provisional legacy package disposition

These recommendations are starting points based on the repository inventory and audit notes. “Retain” means keep the responsibility or useful implementation, not necessarily the existing package unchanged.

| Legacy package | Provisional action | Reason / follow-up |
|---|---|---|
| `differential_drive_robot_model` | **Evaluate and adapt or replace** | Generic differential-drive Gazebo example. Confirm drivetrain and simulator fit before using it as the IGV model. |
| `igv_behavior` | **Replace its navigation role** | Legacy lane-following and stop/search behavior does not provide the intended planner-to-DWB path following. Preserve any reusable state or safety ideas only after review. |
| `igv_bringup` | **Retain and refactor** | Launch/configuration package can remain the system entry point as its launches are updated to start the target stack. Remove obsolete launch references only after replacements work. |
| `igv_control` | **Inspect and define the command boundary** | Review the current `/cmd_vel` or vehicle command bridge, units, limits, timeout, and drivetrain ownership. Adapt or replace after the interface is agreed. |
| `igv_description` | **Retain and update** | Robot description is needed for geometry and TF, but placeholder sensor mounts must be replaced by measured IGV geometry and camera extrinsics. |
| `igv_interfaces` | **Review before retaining** | Keep only custom messages/actions/services that are still required. Prefer standard ROS messages where they express the agreed interface adequately. |
| `igv_localization` | **Replace placeholder estimator behavior** | The audited odometry stub is useful for learning and early visualization, but does not estimate motion from sensors. Develop the estimator and TF ownership deliberately. |
| `igv_perception` | **Retain useful patterns; redesign outputs** | ROS image/cloud handling and debug visualization are useful references. The legacy blob detector and single-camera filters do not satisfy the planned multi-camera, costmap-oriented interface. |
| `igv_tools` | **Assess individually** | Keep utilities with a current user; retire unused tools only after searching scripts, launch files, and documentation for references. |
| `vectornav` | **Verify dependency and role** | Determine whether this is the required driver/source for the selected IMU and its Humble compatibility. Confirm topics, frames, and ownership with state estimation. |
| `zed-ros2-wrapper` | **Verify dependency and version strategy** | Confirm the Humble-compatible driver version, SDK requirements, and whether it is maintained in this repository or installed/built separately. Coordinate with vision. |

Do not implement the table as a bulk deletion list. Update each row when its owner confirms the choice and records dependencies and replacement work.

## Migration workflow

### Stage 1: Establish the baseline

**Work:**

- Identify the current branch and repository revision being changed.
- Build the unmodified working fork in the agreed Humble environment.
- Record which packages build and which launch files can be inspected or run.
- Separate build failures from runtime failures and from features that have not been tested.
- Capture the legacy package inventory and relevant package-relative paths in the audit documents.

**Deliverables:**

- Baseline build notes, including environment and command output summary.
- Updated package assessment showing evidence, unknowns, and initial recommendations.
- List of unavailable assets, hardware, or dependencies that prevent verification.

**Review gate:** The team can identify what was present and buildable before structural changes begin.

### Stage 2: Agree on system contracts

Before moving packages, the subteams agree on the interfaces between them. Record at least:

| Contract | Decisions to record |
|---|---|
| Camera outputs | Per-camera topics, message types, frame IDs, timestamps, rates, and data-health behavior. |
| Vision to local navigation | Obstacle representation, common frame, filtering semantics, and how clearing/free-space observations are supplied. Define boundary output and owner. |
| State estimation | Odometry topic, `odom` and `base_link` ownership, IMU source and frame, timestamp/covariance expectations. |
| Global position and heading | Source topics, global frame, heading convention, and who publishes any global-to-local transform. |
| Planner to DWB | Goal source, path message/frame, path refresh behavior, and what happens when no path is available. |
| DWB to vehicle | Command topic/type, linear and angular units, limits, timeout, arbitration, and stop behavior. |
| Robot and sensor frames | TF tree, transform authority for each edge, and measured sensor extrinsics. |

**Deliverables:**

- Reviewed ROS interface table and TF ownership map.
- Confirmed responsibility owner for each output and each transform.
- Decision log for unresolved planner, simulator, camera-driver, and boundary-detection choices.

**Review gate:** No two nodes are assigned the same TF edge or command output, and each cross-team interface has an owner.

### Stage 3: Define target package boundaries

Choose package boundaries based on stable ROS responsibilities and dependencies. The names below are illustrative, not commitments:

- robot description and vehicle geometry;
- sensor drivers/configuration;
- perception and multi-camera processing;
- state estimation;
- local navigation and costmap configuration;
- planner integration;
- vehicle control/command interface;
- bringup and simulation.

For each proposed package, record its purpose, owned nodes/configuration, public topics/actions/services, dependencies, and the legacy package(s) it replaces or draws from. Avoid splitting packages solely to match subteam labels; a package should have a coherent runtime or build responsibility.

**Deliverables:**

- Target package map.
- Legacy-to-target mapping with action (`reuse`, `adapt`, `replace`, `retire`, or `investigate`), owner, dependency notes, and status.
- Agreed source-control/ownership plan for packages that multiple subteams touch.

**Review gate:** Each legacy package has a disposition or an explicit “investigate” owner, and every target responsibility has a home.

### Stage 4: Migrate in vertical slices

Build and verify a thin, end-to-end capability before migrating every legacy feature. A proposed order is:

1. **Robot model and simulation base:** describe the IGV geometry, drive system, sensors, and essential TF; start the model in the selected simulator.
2. **Command and motion feedback:** send a bounded test command through the chosen control interface and verify simulated motion and odometry.
3. **State estimation interface:** establish the agreed odometry, IMU, and TF outputs; initially use simulator sources where appropriate.
4. **Single-camera perception:** validate image/depth/cloud topics and camera transforms before adding multi-camera fusion.
5. **Multi-camera obstacle input:** combine filtered observations in a common frame and inspect them in RViz; agree how costmap marking and clearing work.
6. **Local costmap and planner/controller:** mark obstacles and boundaries, produce a path with the selected planner, pass it to DWB, and inspect the resulting commands/trajectories.
7. **Global inputs and mission goals:** integrate GPS/global heading and the agreed global planning or goal interface.
8. **Integrated scenarios:** test combinations of sensors, state estimation, planner, controller, and vehicle model.

The sequence can change when a dependency requires it. Record the reason and preserve a working build at each milestone.

### Stage 5: Retire legacy components safely

For each package or node proposed for removal:

1. Search source, launch, configuration, scripts, and documentation for references.
2. Record what the component currently provides and which consumers depend on it.
3. Confirm that a replacement exists or that the responsibility is intentionally removed.
4. Update consumers and launch files, then build and perform the relevant integration check.
5. Remove the old component in a focused change and update the package assessment and repository documentation.
6. Keep links to the upstream legacy repository and note the former package-relative path when historical context remains useful.

Retain experiments only when they have a clear use or are needed for comparison. If archival is preferred, confirm the repository’s agreed archival approach instead of leaving abandoned code looking active.

### Stage 6: Review and maintain the migration record

At each milestone, review the code and docs together. Update:

- package disposition and status in `audits/package_assessment.md`;
- findings and historical path references in `audits/legacy_code_review.md`;
- package and subsystem architecture in the relevant documentation;
- interfaces, ownership, and unresolved choices in the decision log;
- setup instructions when build dependencies or environment assumptions change.

Use package-relative paths in documentation, for example `igv_perception/igv_perception/obstacle_detector_node.py`. State whether a cited path refers to the legacy snapshot or the current fork if those differ.

## Team deliverables

| Team | Workflow deliverables |
|---|---|
| Local navigation / DWB | Costmap data contract, planner choice and path interface, DWB configuration, vehicle command contract, RViz integration view, and navigation scenario results. |
| State estimation / IMU | Sensor/frame inventory, estimator and odometry contract, TF ownership, calibration/validation notes, and stale-data behavior. |
| GPS / global heading | Global pose/heading contract, frame and transform ownership, goal handoff, and validation under expected signal conditions. |
| Vision | Three-camera driver/configuration plan, camera extrinsics, multi-camera fusion output, obstacle/boundary contracts, and perception health/debug outputs. |
| Integration / simulation | Selected simulator and model, reproducible launch entry points, scenario definitions, and milestone-level integration results. |

These are expected work products, not claims that the code or interfaces are already implemented.

## Pull request and change workflow

For each change:

1. Create a focused feature branch from the agreed project branch.
2. Make one coherent change, such as documenting an interface or migrating one package responsibility.
3. Update affected package manifests, launch files, configuration, and docs together.
4. Build the relevant package(s) and run the agreed verification for the change; report what was and was not verified.
5. Open a pull request that describes the purpose, files/packages affected, migration-table updates, evidence, and remaining questions.
6. Review and merge using the team’s GitHub process; then update the local branch before starting the next task.

Do not label code “tested” if only inspection or compilation was performed. Distinguish static review, successful build, simulation check, and hardware check in the pull-request notes.

## Completion criteria for leaving the legacy structure

The package migration is ready to be considered complete when:

- every legacy package has a recorded final disposition;
- active target packages have clear responsibilities and documented interfaces;
- obsolete launch and configuration references have been removed or intentionally retained with explanation;
- the Humble workspace builds reproducibly from the documented setup;
- the integrated simulation can run the agreed navigation path from planner through DWB to the vehicle command interface;
- state estimation, camera frames, and TF ownership are consistent;
- obstacle and boundary inputs reach the local costmap with documented semantics;
- key failure/degraded cases are addressed or explicitly listed as limitations;
- the docs distinguish legacy findings from current implementation and planned work.

## Open decisions

Track these decisions until the relevant owners confirm them:

- Exact Gazebo distribution and ROS integration for the project.
- Exact Nav2/planner configuration and whether any custom planner component is needed.
- Target package names and whether Nav2 packages remain external dependencies or are wrapped by project-specific packages.
- ZED driver/SDK version and repository/install strategy.
- Boundary detector ownership and representation.
- Vehicle command ownership and drivetrain interface.
- Source and authority for GPS/global heading, and global-to-local TF ownership.
- Whether WSL is supported for simulation and hardware bringup or only for development/build work.
