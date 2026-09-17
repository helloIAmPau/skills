# skills

Skills for working on helloIAmPau's repositories, kept in one place so a change
to how the work is done arrives as a diff rather than as folklore.

A skill is a set of instructions that loads only when it is relevant. Nothing
here is read on every session; each file states when it applies, and that is
what decides whether it is loaded at all.

## What is here

| Skill | Load it when |
|---|---|
| [`repo-workflow`](repo-workflow/SKILL.md) | Before creating a branch, committing, pushing, force-pushing, or opening a pull request. Issue first, one branch per issue, squash to one commit before pushing, open the PR and stop — review and merge belong to the repository owner. |
| [`architecture`](architecture/SKILL.md) | Before adding a service or a workspace, naming either, deciding what belongs in a library, changing the ingress, touching the Dockerfile, or building the web app. A service per path prefix; the data API is GraphQL and is called `graphql`; `auth` is not GraphQL and the reason matters; the web app is the `web` client, a server shell and a hydrated client built from one bundle each. |
| [`javascript`](javascript/SKILL.md) | Before writing or reviewing **JavaScript**, and before calling a diff's style wrong in review. No arrow functions, no space before a `function`'s parens, no `else`, no ternaries, no optional chaining and no `??`, at most two arguments, a single-use function written where it is used, template literals for anything spanning lines, promise chains in services and `async`/`await` only in tests. There is no linter; this is the whole of the enforcement. |
| [`agent-container`](agent-container/SKILL.md) | At the start of any session that will run the stack, use docker or compose, touch an `.env` file, run migrations or the suite, or edit an issue body. The docker CLI is missing, `/workdir` is a different path to the daemon, and `sleep` is blocked. |

## How a repository gets them

As a submodule, at the path the tooling already looks in:

```sh
git submodule add https://github.com/helloIAmPau/skills.git .claude/skills
```

A fresh checkout has to initialise it, and a checkout that skips this gets an
empty directory and no error — the worst way to be missing something:

```sh
git submodule update --init .claude/skills
```

The repository is private, so cloning it needs credentials. `gh` supplies them
through git's credential helper; CI needs its own answer.

Each consuming repository pins a commit. That is the point: a repository says
which version of the working agreement it is on, and moving to a newer one is a
deliberate act with a diff attached.

## The layout

```
skills/
  README.md
  repo-workflow/
    SKILL.md
  agent-container/
    SKILL.md
  architecture/
    SKILL.md
  javascript/
    SKILL.md
```

**One directory per skill, holding a `SKILL.md`.** The directory name is the
skill's name and matches the `name:` in the frontmatter. A flat `.md` at the
root is not a skill — it is tracked and never loaded, which fails by looking
like nothing happened.

Anything a skill needs beyond its own text — a reference, a script, a template
— sits in that directory beside it. That is the reason for the directory: a
sibling file at the root is associated with nothing.

## Adding one

```markdown
---
name: <directory-name>
description: <what it covers, and when to load it>
---

# Title

...
```

**The `description` is the most important line in the file**, and the one most
often written as an afterthought. It is all that is read when deciding whether
to load the skill, so it has to say both *what the skill covers* and *the
moments it applies to* — "before opening a pull request", "at the start of a
session that will run the stack". A description that only names a topic is a
skill that loads too late to help.

Some things that keep these useful:

- **Say why, not only what.** A rule with its reason attached survives contact
  with a case it did not anticipate; a bare instruction does not.
- **Write what is true here**, not what is true in general. The value is in the
  specifics — this container, these repositories, this failure that actually
  happened.
- **One subject per skill.** Two subjects in one file means loading both to get
  either, and the wrong one is noise at the moment it is least wanted. A
  language counts as a subject, and the name should say which — `javascript`,
  not `code-style`, so that a second language can sit beside it rather than
  look like a subset of it.
- **Point at siblings** rather than repeating them. Each of the two skills here
  ends up in the other's territory occasionally; each says so and moves on.
- **The repository's own README wins.** A skill describes how work is done; a
  project's README is the contract for what is being built. Where they
  disagree, the README is right and the skill needs a change.

Changes here follow the workflow the `repo-workflow` skill describes, which
includes this repository.
