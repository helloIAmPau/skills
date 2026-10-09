# Shared coding skills

Reusable coding-agent instructions for JavaScript, React, Rust, native Expo
applications and selected npm/Compose architectures. The collection provides
explicit coding conventions and an issue-first GitHub workflow. Choose the skills
that fit the task and architecture; using a language skill does not select a
database, add a mobile application or migrate an existing project.

Agents should read [AGENTS.md](AGENTS.md) for the collection's operating guide.
`CLAUDE.md` is a relative link to that same file. Consuming projects retain their
own root instructions and configuration.

## Available skills

| Skill | Choose it for |
| --- | --- |
| [javascript](skills/javascript/SKILL.md) | Functions, guards, promises, comments, ESM source and host-run E2E tests |
| [react](skills/react/SKILL.md) | Minimal components with one responsibility, JSX, state, hooks, contexts and selected server-document/browser-entry web structure |
| [rust](skills/rust/SKILL.md) | Two-space layout, separate imports, explicit matches/returns, domain errors, typed data and async work |
| [react-native-expo](skills/react-native-expo/SKILL.md) | Selected native apps, styling, font startup, containerized Android/Metro and host-native test drivers |
| [npm-workspace-services](skills/npm-workspace-services/SKILL.md) | Selected npm workspaces with source libraries, bundled services, Express/GraphQL, Compose and Caddy |
| [clickhouse](skills/clickhouse/SKILL.md) | Selected ClickHouse access, schemas derived from requirements and replay-safe numbered migrations |
| [github-feature-workflow](skills/github-feature-workflow/SKILL.md) | An adopted issue-first agreement with implementation/publication authorization and one feature commit/PR |
| [prerelease-review](skills/prerelease-review/SKILL.md) | Release change/issue tracing, test coherence and execution, code/security review, a GitHub report and owner-approved versioning/tagging |
| [workspace-ownership](skills/workspace-ownership/SKILL.md) | UID `1000` ownership for authored files and touched Git metadata in privileged development environments |

Combine JavaScript + React for web UI, and JavaScript + React + native for Expo
work. Add the workspace architecture skill for repositories using that structure,
ClickHouse for selected persistence and the GitHub workflow where adopted.
React responsibility-based extraction takes precedence over JavaScript's
single-caller helper rule: split independent concerns even with one caller.
JavaScript and React treat `null` and `undefined` as equivalent absence. Prefer
meaningful initial variable/state values; use `useState()` when state has no value
yet. See the
[initialization convention](skills/javascript/SKILL.md#absence-and-initialization).

## Install and initialize

From a consuming Git repository without an existing `.agents` directory:

```sh
git submodule add -b master https://github.com/helloIAmPau/skills.git .agents
git add .gitmodules .agents
```

Commit `.gitmodules` and the submodule reference through the project's authorized
workflow. The reference pins a specific skills commit; upstream changes do not
automatically update it. Inspect an existing `.agents` directory before choosing
a compatible installation path.

After cloning a project already using this submodule:

```sh
git submodule update --init --recursive .agents
```

Alternatively clone with `git clone --recurse-submodules`. Repository access is
required if the remote is restricted. Installed skill entrypoints are
`.agents/skills/<skill-name>/SKILL.md`; the collection guide is `.agents/AGENTS.md`.
Link to that guide from the consuming project's own instructions when appropriate;
its presence in a submodule does not establish automatic inheritance.

## Invoke skills

Select a skill through your agent's skill interface or name it in the task:

```text
Use $javascript and $react to review this component.
Use $javascript, $react and $react-native-expo to update this native screen.
Use $rust to review domain errors and asynchronous initialization.
Use $github-feature-workflow under our adopted agreement to propose a feature issue.
Use $prerelease-review to analyze this project and publish its prerelease report.
```

Read the selected `SKILL.md` before applying it. Follow relevant cross-skill links
and the consuming project's instructions; higher-priority environment instructions
and explicit user decisions govern conflicts. Actual paths, scopes, app identifiers,
ports, toolchains and remotes come from the consuming configuration.

## Development and verification

The shared [development installation contract](skills/npm-workspace-services/SKILL.md#post-mount-installation-barrier)
is: start containers with bind mounts active, install locked dependencies from
each effective workspace root, wait for success, build/prepare native apps, start
watchers/Metro, then confirm actual readiness. Image-build installation does not
replace installation after mounts. Coordinate installs sharing writable paths and
compatible native artifacts; failure prevents builds, watchers and readiness.
Manifest/lockfile changes require stopping affected processes, successful
installation and rebuild/restart. Host test dependencies are installed separately.

All test drivers and root `npm test` run on the host. Containers may run services,
emulators, apps and development servers. HTTP application assertions use the
configured public URL through Caddy; native tools use published ADB, and other
service tests use their documented public protocols. Publish only required
development endpoints and use loopback for local access. Host and device URLs
must be deliberately reachable; HTTP clients derive authority from the URL.
See the [public-interface rules](skills/npm-workspace-services/SKILL.md#host-testing-and-public-interfaces).

For every required E2E run, start a fresh complete applicable development stack,
wait for readiness, run selected migrations where applicable, run the complete
host suite, and tear down after success or failure while preserving data. Keep
this orchestration outside test hooks/helpers. Missing prerequisites are reported
as failures/blockers, never as passing skips. Documentation/link checks do not
establish that a consuming stack, emulator or E2E suite passed.

## GitHub workflow

Where the agreement is adopted: propose an issue, obtain user authorization,
implement on a matching feature branch, verify the complete E2E suite, then obtain
publication authorization for one feature commit and a PR. Resolve remotes/base
branches from the checkout. Reuse approvals already given; publication and merging
have their own scope. Details live in [github-feature-workflow](skills/github-feature-workflow/SKILL.md).

## Update a pinned version

With a clean submodule tree, fetch the configured tracking branch and review the
change from the consuming project's root:

```sh
git submodule update --remote .agents
git diff --submodule=log -- .agents
git add .agents
```

The installation above configures `master`; respect an intentionally different
tracking branch. Commit the updated reference through the consuming workflow.
`git submodule update --init --recursive .agents` restores the consuming project's
pinned commit instead of choosing the latest remote version.

## Contribute or maintain

Edit this repository directly or create a branch inside the submodule; submodule
checkouts can have a detached HEAD. After authorized publication of the skills
commit, update the consuming project's pin separately so others can fetch it.

Keep one package per `skills/<name>/`, with required YAML frontmatter:

```markdown
---
name: example-skill
description: Explain the capability and when it applies.
---

# Example skill

Instructions for its scope.
```

Keep examples project-agnostic and consistent with the chosen conventions. Link
shared execution contracts instead of duplicating recipes. Keep references inside
the package; optional `agents/openai.yaml` provides display/invocation metadata.
Check frontmatter, links and cross-skill consistency, and perform proportionate
behavioral simulations when useful. Apply the UID `1000` policy to authored files
and touched metadata. Report actual checks and unresolved decisions. Instructions
do not install a linter or replace a consuming project's required verification.
