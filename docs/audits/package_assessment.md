# Package Assessment

## Purpose

This document records what each ROS 2 package does, how it connects to the rest
of the system, and whether it should be kept, changed, archived, or removed.

Assessments are based on repository contents unless runtime testing or team
confirmation is explicitly noted.

## Assessment summary

| Package | Role | Key nodes or files | Current status | Recommendation |
|---|---|---|---|---|
| `package_name` | What responsibility it appears to have | Main nodes, launch files, or resources | Verified / Prototype / Placeholder / Unknown | Keep / Replace / Archive / Remove candidate / Investigate |
| `differential_drive_robot_model` |  |  |  |  |
| `igv_behavior` |  |  |  |  |
| `igv_bringup` |  |  |  |  |
| `igv_control` |  |  |  |  |
| `igv_description` |  |  |  |  |
| `igv_interfaces` |  |  |  |  |
| `igv_localization` |  |  |  |  |
| `igv_perception` |  |  |  |  |
| `igv_tools` |  |  |  |  |

## Package details

### `package_name`

**Purpose:**  
What does this package do?

**Key contents:**
- Nodes or executables:
- Launch files:
- Config, message, or robot-description files:

**Connections to other packages:**
- Subscribes to:
- Publishes to:
- Depends on:

**Current status:**  
Is it active, a prototype, a placeholder, or unclear? What evidence supports that?

**Recommendation:**  
Keep / Replace / Archive / Remove candidate / Investigate

**Reasoning:**  
Why does this recommendation fit? Note what depends on this package and what
could break if it changes.

**Open questions:**
- What needs runtime testing or confirmation from the previous team?
- Is any required hardware, model, configuration, or external package missing?

**Evidence reviewed:**
- `path/to/relevant/file`
- `path/to/another/file`

---

## Packages not yet assessed

- `package_name`