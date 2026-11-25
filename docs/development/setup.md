# Development Environment Setup

This document describes the baseline developer environment for working on the IGV ROS 2 repository. It is intended to help team members prepare a machine, build the workspace, and verify that the ROS packages are discoverable.

## Supported baseline

| Item | Project baseline | Notes |
|---|---|---|
| Operating system | Ubuntu 22.04 LTS | ROS 2 Humble's target Ubuntu release. |
| ROS 2 distribution | Humble | Keep the project on Humble unless the team explicitly approves a migration. |
| Build system | `colcon` with `ament` packages | Build from the workspace root. |
| Source control | Git and the team GitHub fork | Work on a feature branch and use pull requests for review. |
| Editor | VS Code or another editor | Optional; use whichever editor the developer prefers. |

The repository has also been developed from Ubuntu under WSL. WSL can be used for code editing and builds, but camera hardware, GPU access, USB devices, GUI applications, and Gazebo behavior can differ from native Ubuntu. Validate hardware and simulation workflows on the team's target machine before relying on them.

## Before building

Each developer should have:

- Ubuntu 22.04 with ROS 2 Humble installed and sourced.
- `colcon` and `rosdep` available.
- Git access to the current project fork.
- A workspace directory with the repository checked out under `src/`.
- Network access for installing declared package dependencies.

The ZED SDK and ROS driver, camera-specific dependencies, and the exact Gazebo distribution are not specified here. Confirm their supported versions with the vision and simulation subteams before installing them; keep those instructions in the relevant subsystem documentation once decided.

## Create the workspace and clone the fork

Choose a workspace path and use it consistently. This example uses `~/igv_ros2_ws`:

```bash
mkdir -p ~/igv_ros2_ws/src
cd ~/igv_ros2_ws/src
git clone https://github.com/AdrianGonzalezMech/igv_ros2.git
```

The clone command creates a directory named `igv_ros2`. If the repository is already cloned, skip the clone step and use the existing checkout. Before starting work, switch to the intended feature branch and fetch the latest remote state according to the team's Git workflow.

## Source ROS 2 and install package dependencies

Open a terminal and source the ROS 2 Humble installation:

```bash
source /opt/ros/humble/setup.bash
```

From the workspace root, install dependencies declared by packages in `src/`:

```bash
cd ~/igv_ros2_ws
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

If `rosdep` reports a missing key or an unavailable system dependency, record the package and error for the team. Do not silently remove dependency declarations to make the command pass.

## Build the workspace

From the workspace root:

```bash
cd ~/igv_ros2_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install
```

After a successful build, source the workspace overlay in that terminal:

```bash
source install/setup.bash
```

Source the ROS installation before building, and source the workspace overlay in each new terminal used to run or inspect the built packages. A new terminal does not automatically know about packages built in this workspace unless the overlay is sourced or configured in the shell startup file.

## Verify the setup

From a terminal where both ROS and the workspace overlay are sourced:

```bash
ros2 pkg list | grep '^igv_'
```

The command should list the IGV packages that built successfully. You can also check a package directly:

```bash
ros2 pkg prefix igv_bringup
```

The package prefix should point into this workspace's `install/` directory. If a package is missing, review the `colcon build` output for build failures and verify that the package exists under `src/`.

## Rebuild after changes

Run a build from the workspace root after changing package code, metadata, launch files, or configuration that must be installed into the workspace:

```bash
cd ~/igv_ros2_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

`--symlink-install` helps during development by linking many package resources to the source tree. Python code changes may be visible without rebuilding in some setups, but rebuilding and sourcing after package or entry-point changes avoids confusion about which version is running.

For a focused build when working on one package and its dependencies:

```bash
colcon build --symlink-install --packages-up-to <package_name>
```

Replace `<package_name>` with the actual ROS package name.

## Common setup issues

| Symptom | Check |
|---|---|
| `ros2` or `colcon` command not found | Confirm ROS 2 Humble is installed and `/opt/ros/humble/setup.bash` has been sourced. |
| Package not found after building | Check the build result, source `install/setup.bash`, and confirm the package is under this workspace's `src/`. |
| Dependency cannot be resolved by `rosdep` | Confirm the dependency key is valid and available for Humble/Ubuntu 22.04; report missing or platform-specific dependencies. |
| Changes appear not to take effect | Confirm the terminal has sourced the intended workspace overlay and that another workspace is not taking precedence. Rebuild after metadata or entry-point changes. |
| Gazebo or camera driver cannot start | Check the version and installation steps agreed by the simulation or vision subteam; those dependencies are not yet standardized in this document. |

## Environment decisions to record

As the team finalizes its development environment, update this document with the confirmed details for:

- Gazebo distribution and ROS integration packages.
- ZED SDK, ZED ROS 2 wrapper, and supported camera firmware/driver configuration.
- Any system-specific GPU, USB, or udev requirements.
- Whether WSL is supported for simulation and hardware work, or only for code development.
- The team's agreed workspace path and branch/pull-request workflow.

Keep subteam deliverables and the legacy-to-target package migration sequence in [`workflow.md`](workflow.md). This file is for preparing and building a developer environment.
