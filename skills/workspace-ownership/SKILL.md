---
name: workspace-ownership
description: Preserve standard-user ownership when creating or modifying project files from a root container. Apply during filesystem-changing work; generated dependencies, build outputs, caches, and runtime data may be excluded.
---

# Workspace ownership

When running as root in a development container, leave every project file and directory you create or modify owned by the standard development user and the `users` group. This includes source, configuration, documentation, skills, and Git metadata created or changed by your commands.

## Select the owner

- In this workspace, use `node:users`: `node` is the standard user (UID 1000), and existing project files use this ownership.
- In another container, infer the standard user from the workspace and existing non-generated project files, and verify it with `id` or `getent`. Do not assume the invoking root account is the intended owner. If the evidence is ambiguous, ask for the username before changing ownership.

## Apply ownership

- Track paths created or modified during the task, including files written indirectly by tools and any new parent directories. Apply `chown node:users -- <paths>` after writes and before handing the work back. Substitute the verified account in other containers.
- Include this skill's own files when creating or updating it. Correct root-owned artifacts from earlier work in the same task when identified.
- Generated content may be skipped: for example `dist/`, `dists/`, `build/`, `node_modules/`, `target/`, caches, coverage reports, and runtime `data/`. Use the directory's actual purpose; authored source or fixtures still need the intended ownership.
- Scope changes to touched paths. Recurse only through a directory whose entire contents are in scope, such as a newly created `.git` directory. Do not recursively chown the entire workspace, unrelated files, system paths, or bundled tools.
- Do not follow symlinks outside the intended scope. When the touched path is a symlink, use `chown -h` to change the link itself.
- Preserve file modes. Ownership correction does not authorize staging, committing, or pushing, and does not replace required filesystem permission approval.
- Verify ownership of the affected non-generated paths with `stat` or an equivalent check. If correction is blocked, report the affected paths and the actual error.
