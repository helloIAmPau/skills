---
name: workspace-ownership
description: Preserve UID 1000 ownership of authored project files and touched Git metadata when working from a privileged development environment. Apply during filesystem-changing work; generated artifacts may be excluded.
---

# Workspace ownership

Leave authored project files and directories created or modified during the task
owned by numeric UID `1000`. Use the numeric UID directly; do not depend on an
account name being present or matching across environments. Preserve existing
group ownership and file modes unless the user explicitly requests a group/mode change.

## Apply ownership

- Track paths created or modified, including files written indirectly, new parent
  directories and touched Git metadata. When running with sufficient privilege,
  apply `chown 1000 -- <paths>` after writes and before handing back the work.
- Include changes to this collection's own instructions and supporting files.
  Correct artifacts created by earlier operations in the same task when needed.
- Generated dependencies, build outputs, caches, coverage and runtime data may be
  excluded. Determine purpose from the actual path; authored fixtures still qualify.
- Limit changes to touched paths. Recurse only through a directory whose entire
  contents are in scope. Do not recursively change the whole checkout, unrelated
  files, system paths or bundled tools.
- Do not follow symlinks outside scope. Use `chown -h 1000 -- <link>` for a touched
  symlink. Changing its owner does not authorize changing its target outside scope.
- Ownership correction does not authorize staging, committing or publication.
- Verify numeric ownership with `stat` or equivalent. If permissions prevent
  correction, report the actual error and affected paths; do not claim success.
