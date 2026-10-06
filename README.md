# Shared coding skills

Reusable instructions for coding agents working on helloIAmPau's projects.
Each skill describes when it applies and the conventions to follow. Several
skills contain Hybrid-specific architecture and tooling decisions; read their
scope before applying them to another project.

This repository is installed as the consuming project's `.agents` directory.
Its skills live under `.agents/skills/<skill-name>/SKILL.md`.

## Install in a project

From the root of a Git repository that does not already have a `.agents`
directory:

```sh
git submodule add -b master https://github.com/helloIAmPau/skills.git .agents
git add .gitmodules .agents
```

Commit `.gitmodules` and the `.agents` submodule reference using the project's
publication workflow. The project pins a specific skills commit; subsequent
changes in this repository do not automatically change that pin.

After checking out a project that already references the submodule, run:

```sh
git submodule update --init --recursive .agents
```

Alternatively, clone the consuming project with `git clone --recurse-submodules`.
GitHub credentials with repository access are required if access is restricted.

## Use the skills

Open Codex in the consuming project. Codex discovers repository skills under
`.agents/skills` and can select one when your task matches its description.
In Codex CLI or the IDE extension, use `/skills` to select a skill or mention
its name with `$` in your prompt. See the [official skill documentation](https://learn.chatgpt.com/docs/build-skills).

For example:

```text
Use $react and $javascript to review this React component.

Use $react-native-expo, $react and $javascript to update the mobile screen.

Use $rust to implement or review Rust code in my pul.se and Olivia style.

Use $github-feature-workflow to propose the next feature issue.
```

Read the applicable `SKILL.md` before implementing or reviewing its subject.
Follow its linked skills where the task crosses their boundaries. Project
instructions belong in the consuming project's `AGENTS.md`; explicit owner
instructions take precedence over skill guidance. If a changed skill does not
appear in Codex, restart the session.

## Available skills

| Skill | Use it for |
| --- | --- |
| [javascript](skills/javascript/SKILL.md) | JavaScript functions, control flow, promises, formatting, comments, modules and tests. |
| [rust](skills/rust/SKILL.md) | Rust source conventions derived from pul.se and Olivia: two-space layout, separate imports, explicit matches and returns, domain errors, typed data and async execution. |
| [react](skills/react/SKILL.md) | React components, JSX, hooks, state, effects and contexts, including the mandatory single-responsibility and minimality rules. Web styling guidance applies only to web apps. Use alongside `javascript`. |
| [react-native-expo](skills/react-native-expo/SKILL.md) | Hybrid's mobile workspace, native components and styling, Expo startup, development builds and device E2E tests. Use alongside `react` and `javascript`. |
| [npm-workspace-services](skills/npm-workspace-services/SKILL.md) | Projects adopting npm workspaces, source-only libraries, bundled Express/GraphQL applications, Docker Compose and Caddy. |
| [clickhouse](skills/clickhouse/SKILL.md) | Hybrid's ClickHouse access, entries schema, container configuration and numbered SQL migrations. |
| [github-feature-workflow](skills/github-feature-workflow/SKILL.md) | Projects adopting the issue-first agreement: approve the issue plan, implement, run E2E tests, then approve publication of one feature commit and a PR. |
| [workspace-ownership](skills/workspace-ownership/SKILL.md) | Preserving the standard development user's ownership of authored files and Git metadata when working from a root container. |

Every React and React Native component must have exactly one responsibility and
be as minimal as possible. Split independent concerns into focused components
or hooks even when an extracted unit has only one caller. Enforce this during
implementation and review; the details live in the React and native skills.

## Update a project's pinned version

With a clean submodule working tree, fetch the configured `master` branch and
review the resulting change from the consuming project's root:

```sh
git submodule update --remote .agents
git diff --submodule=log -- .agents
git add .agents
```

Commit the updated reference through the project's publication workflow.
Use `git submodule update --init --recursive .agents` to restore the version
pinned by the consuming project instead of selecting the latest remote version.

## Edit or add skills

Work in this repository directly, or create a branch inside the `.agents`
submodule before editing. Submodule checkouts may otherwise have a detached
HEAD. Publish the skills commit first, then update and commit the consuming
project's `.agents` reference so other developers can fetch that commit.

Keep one directory per skill under `skills/`, with a `SKILL.md` containing YAML
frontmatter:

```markdown
---
name: example-skill
description: Describe what the skill does and when it applies.
---

# Example skill

Instructions for the task.
```

Use clear descriptions, preserve the intended scope and explain non-obvious
constraints. Keep supporting scripts, references or assets inside the relevant
skill directory, and link shared guidance rather than copying it. Optional
`agents/openai.yaml` files provide skill display and invocation metadata.

Review changed instructions for conflicts and broken relative links before
publishing. Skill files guide agents; they do not install a linter or replace
the consuming project's required verification.
