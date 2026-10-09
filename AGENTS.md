# Agent guide for the skills collection

This repository maintains reusable skills, examples, references and metadata.
Read [README.md](README.md) for the human catalog and installation guidance.
`CLAUDE.md` is a relative symbolic link to this canonical guide; maintain one source.

## Select and combine skills

Identify relevant skills by their names/descriptions and the actual task. Read
applicable `skills/<name>/SKILL.md` before applying it; read linked references only
when needed. Do not load unrelated guidance simply because it exists.

Combine language, UI, architecture and workflow skills when their scopes apply.
JavaScript + React covers web UI; JavaScript + React + native covers selected Expo
apps. Rust guides Rust source. Architecture skills apply only to selected
architectures: language/UI work alone does not authorize scaffolding, a mobile
workspace, a database or an architecture migration.

Preserve explicit precedence: React's responsibility-based component/hook
extraction overrides JavaScript's single-caller helper rule. Keep web DOM/CSS/mask
examples distinct from native components. Native applications do not inherit
backend bundling/listener rules. Keep source-only libraries and compiled
application services distinct.

## Instructions, configuration and authorization

Respect higher-priority execution-environment instructions, the user's explicit
decisions and the consuming project's applicable instructions. Reuse authorization
already given; do not invent or repeatedly request approval. Apply
[github-feature-workflow](skills/github-feature-workflow/SKILL.md) only where that
agreement is adopted. Implementation, commits, PRs, image/site publication and
merging have distinct authorization scopes.

Resolve paths, package scopes, app identifiers, environment settings, toolchain
versions, ports, services and Git remotes/base branches from the current checkout.
Inspect uncommitted work and local/remote differences; preserve unrelated changes
and resume a matching feature branch when continuing work.

When installed as `.agents`, entrypoints live at
`.agents/skills/<name>/SKILL.md`, with this guide at `.agents/AGENTS.md`.
A consuming project's own root `AGENTS.md` still governs that project. Do not
claim this submodule guide is automatically inherited; reference it explicitly
through the consuming instructions or task where needed. Within this collection,
paths in links are relative to the containing file.

## Shared execution contracts

For relevant development stacks, follow the canonical
[post-mount installation barrier](skills/npm-workspace-services/SKILL.md#post-mount-installation-barrier):
active mounts, successful locked installation in each effective environment,
initial builds/native preparation, watchers/Metro, then actual readiness.
Installation failure prevents subsequent startup. Serialize writes to shared
dependency paths and account for platform/ABI differences. Host tool dependencies
need a host installation; image-build or private-container installs do not provide them.

Follow the canonical [host-test/public-interface contract](skills/npm-workspace-services/SKILL.md#host-testing-and-public-interfaces)
and [E2E lifecycle](skills/npm-workspace-services/SKILL.md#e2e-run-lifecycle).
All test drivers run on the host. HTTP application assertions go through the public
gateway URL; native drivers use published ADB, and other targets use their public
protocols. Configure host/device addressing without manual Host headers or
container-IP discovery. Keep stack startup, migration application and teardown
outside tests/hooks/helpers. Missing prerequisites are failures/blockers.

Apply [workspace-ownership](skills/workspace-ownership/SKILL.md) to filesystem
changes: authored files/directories and touched Git metadata must have owner UID
`1000`. Preserve groups/modes and limit corrections to task-touched paths, including
symlinks without following unrelated targets. Generated artifacts may be excluded.
Report correction failures honestly.

## Maintain and verify the collection

Inspect current files before editing. Keep skill bodies, frontmatter, supporting
references and agent metadata agnostic: no personal provenance, product schemas,
fixed consuming identifiers or historical source-count claims. Use neutral
placeholders, official technology references and valid relative links. Keep chosen
conventions and meaningful exceptions intact. The UID `1000` ownership convention
is an explicit collection policy.

Maintain one clear source for each shared execution contract and link to it.
Review positive examples against the same rules they teach, including functional
React updaters, meaningful variable initialization, font failure rendering,
extensionless source imports and guards.
Do not silently settle unresolved policy decisions or copy task-specific approval/
status discussions into reusable skill policy.

For skill/documentation edits, validate frontmatter and directory names, local
file/heading links, metadata, symlink resolution and cross-skill consistency.
Use concise behavioral simulations where appropriate; do not scaffold a consuming
application solely for validation. Reading an instruction or simulating a snippet
is not proof of Docker, application, emulator or full E2E success. Run consuming
runtime checks only when the task and available environment support them.

Report actual changes, checks performed, missing/unverified runtime evidence and
unresolved points. Do not claim skipped or unavailable tests passed. Honor separate
commit/publication instructions and preserve unrelated user work.
