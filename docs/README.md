IGV ROS 2 Documentation
This folder contains project documentation for the IGV ROS 2 software, including notes about the previous team's prototype and plans for the current system.
Repositories
- Current working fork: AdrianGonzalezMech/igv_ros2
- Original legacy source snapshot: the uploaded igv_ros2-main archive supplied for this audit. The archive contains the source tree, but no Git remote metadata or upstream URL.
- Original legacy GitHub repository: Csun-TylerBernardino/igv_ros2
The working fork is the actively changing project copy. The uploaded archive preserves the legacy source snapshot reviewed during the audit. Use the upstream repository to inspect the original legacy files when a referenced path has changed or disappeared from the working fork.
Reading audit notes as the fork changes
Some audit documents describe the legacy code as it existed when it was reviewed. As the current fork is updated, files may be renamed, moved, replaced, or removed. A path mentioned in an audit may therefore no longer exist in the current checkout.

When following a legacy reference:
1. Start with the package name and relative path shown in the document.
2. Check the current fork first to see whether the file still exists or has a successor.
3. If it has changed or disappeared, use the upstream legacy repository link above to inspect the original version.
4. Use the Git history in the fork to trace changes when the original file was copied into this repository.
5. Treat audit findings as descriptions of the reviewed legacy snapshot, not as confirmation that the current fork still behaves that way.
Where possible, audit references should name files relative to their ROS package and distinguish legacy behavior, current implementation, and planned work. When a file is removed or replaced, update the relevant audit note with its status and, if known, the replacement path. Avoid silently rewriting historical findings as though the old code were still present.
