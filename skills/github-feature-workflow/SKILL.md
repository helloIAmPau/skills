---
name: github-feature-workflow
description: Coordinate development in repositories with a `docs/README.md`. Read it before any work, write or update a feature document and obtain user approval before implementation, run the mandatory E2E suite, and obtain user approval after passing tests before publishing one feature commit and a PR. Use for any work in such a repository.
---

# GitHub feature workflow

This is the shared development agreement for developers and coding agents. The repository's `docs/` folder is the specification: `docs/README.md` describes the project and each feature has its own document. Code and E2E tests must reflect the documentation. GitHub hosts the feature branch and its pull request. The user approves feature documents and decides when implementations are ready to commit and submit for review.

Read higher-priority environment instructions, the project's documentation and existing user authorization before acting. A feature document already supplied as approved authorizes its implementation; do not reopen settled approval gates. Document approval and commit approval are separate. Never infer approval from silence, elapsed time, passing tests, or approval of another task. Honor approvals already given without asking for them again unless the relevant document changes materially.

The feature lifecycle is: feature document → user approval of the document → implementation → E2E tests. Failed tests return the work to implementation and testing. Passing tests lead to a second user approval, then publication of the feature commit and pull request.

## 0. Read the project documentation

Before any work, read `docs/README.md` and the feature documents relevant to the task. A new project starts by discussing it with the user and writing `docs/README.md` together. Do not create root `AGENTS.md` or `CLAUDE.md` files; `docs/README.md` is the project's instructions.

## Repository documentation

Maintain these files:

- `README.md` (root): a user-targeted description of the app, its purpose, features, and usage. Distinguish planned capabilities from implemented ones.
- `docs/README.md`: the project overview, architecture, development conventions, setup/verification commands, the domain glossary (a "Language" section), a link to this skill, and an index of the feature documents.
- `docs/<feature>.md`: one document per capability, named after the capability, such as `docs/record-search.md`. It records requirements, behaviour, acceptance criteria, relevant data/API contracts and E2E scenarios. Keep feature documents directly in `docs/`; decision records live in `docs/adr/`.

Documentation describes the current system. Do not keep task lists, execution steps, approval/status ledgers, progress or per-change history in `docs/`; those belong in the conversation and the PR.

Read existing documentation before editing. Preserve relevant existing content. A request to draft initial documentation authorizes that drafting, not application implementation or commits. Keep documentation and applicable installed skills current within approved feature work, respecting their actual location and separate repository boundaries. Skills record durable definitions and conventions, not status.

## 1. Write or update the feature document

For every feature under this agreement, create `docs/<feature>.md` or edit the existing document that covers the behaviour, unless existing instructions already authorize the work. When continuing existing work, reuse its approved document. Every feature must have an E2E suite that validates its acceptance criteria; describe the E2E scenarios and expected observable outcomes in the document so the user approves implementation and test scope together. Add new documents to the index in `docs/README.md`.

Use this structure:

````markdown
# <Feature name>

## Requirements
The user need, scope, and constraints.

## Behaviour
How the feature works from the user's and the system's point of view,
including failure cases.

## Acceptance criteria
- Observable, verifiable outcomes.

## Contracts
Relevant data, API, or request/response shapes, when the feature defines any.

## E2E scenarios
- The scenarios the E2E suite runs and the outcome each one expects.
````

Propose the ordered execution steps in the conversation alongside the document: implementation steps, the E2E suite to add or extend, the test run, and documentation updates. Raise unresolved decisions there as well.

Choose the Conventional Commit type for the change and use it for the branch prefix and the commit. The working vocabulary is:

| Type | Purpose |
| --- | --- |
| `feat` | New capability |
| `fix` | Behaviour correction |
| `docs` | Documentation |
| `refactor` | Code restructuring without behaviour changes |
| `perf` | Performance improvement |
| `test` | Test changes |
| `build` | Build tooling or dependencies |
| `ci` | Continuous integration |
| `chore` | Other maintenance |
| `style` | Code formatting without behaviour changes |
| `revert` | Reverting an earlier change |

Conventional Commits defines `feat` and `fix` and permits additional types; this vocabulary is the project's convention. Use `feat` for a feature even when it includes tests and documentation.

Changes that do not alter behaviour, contracts or acceptance criteria, such as refactors and chores, need no feature document; present their plan in the conversation and obtain approval the same way.

Present the document and wait for explicit approval. Before approval, the feature document is the only file you may change; read-only investigation is allowed. Application code, scaffolding, dependency changes, migrations, and implementation tests must wait. Never fabricate an approval.

## 2. Implement on a separate branch

When continuing work, inspect the current branch, its feature document and
local/remote differences; resume an existing matching feature branch. Do not
restart approved work or inherit unrelated local commits accidentally.

For a new feature, resolve the appropriate configured remote and base branch from
checkout settings, upstream/default-branch information and the user's instructions.
Fetch and inspect differences. Preserve uncommitted work, then switch to the local
base, update it from the selected remote with a fast-forward pull, and only then
create the feature branch. Do not substitute branching directly from the remote
ref for this sequence. If the local base contains unrelated commits or cannot
fast-forward, resolve the divergence without discarding work before proceeding.

```text
git switch <base>
git pull --ff-only <remote> <base>
git switch -c <type>/<feature-slug>
```

Derive the slug from the feature document's name: lowercase it, replace spaces with hyphens, remove invalid punctuation, and collapse repeated hyphens while retaining its meaning.

Example: a `feat` change documented in `docs/record-search.md`:

```text
feat/record-search
```

Keep the feature on its own branch and preserve unrelated user changes. Do not implement directly on the default branch.

Implement the approved document while sharing progress so the user can review the code and provide insights during development. Incorporate feedback within the approved scope. If feedback or discoveries materially change the scope or solution, update the feature document and obtain approval for the revised portion before implementing it.

Add or extend the feature's E2E suite alongside the implementation. Update the documentation and decision records where needed. Leave the implementation and tests uncommitted through testing and user review. Do not make WIP, checkpoint, automatic, or intermediate commits, including to enable remote review.

## 3. Run E2E tests and return to implementation on failure

Every feature requires an E2E suite that checks the feature's validity through observable behaviour in the running stack. Cover every acceptance criterion in the feature document, including relevant failure cases. Extend existing suites where appropriate; a generic health check alone does not validate unrelated feature behaviour.

Keep the E2E test layout flat: all test files live directly in `tests/e2e/`, without nested feature or platform directories. Distinguish features through filenames.

This selected workflow uses E2E tests only, through the real system's public
interfaces without mocked transports or in-process application imports. Follow
the shared [host-test contract](../npm-workspace-services/SKILL.md#host-testing-and-public-interfaces)
and [E2E run lifecycle](../npm-workspace-services/SKILL.md#e2e-run-lifecycle):
a fresh complete applicable development stack, post-mount installation/builds/
watchers/readiness, successful selected migrations where applicable, root
`npm test` on the host, then teardown even on failure while preserving data.
Keep stack startup, migration application and teardown outside tests/hooks/helpers.
Containers run the targets, not Node/Maestro/recording/migration verification drivers.
Do not substitute a partial stack or production runtime for required development
verification. Tests may inspect containers or temporarily inject and restore real
failures to validate an acceptance criterion. Missing host tools or services are
failures/blockers, never skipped passing tests.

Always run the complete E2E suite with `npm test`, including every feature and existing regression coverage. Do not add or use feature-specific test scripts such as `test:mobile`. If tests fail, return to implementation, correct the cause, and repeat the complete startup, migration, test and teardown sequence. Continue within the approved scope without requesting renewed permission for routine fixes. Do not skip, remove, or weaken valid assertions merely to obtain a passing result.

Tests that could not run are not passing tests. Report any environment or access blocker and resolve it within the available authorization; do not advance to publication while required E2E verification remains incomplete.

## 4. Wait for user approval after tests pass

Once the required E2E suites pass against the current implementation, present the concrete changes and test results for user review. Wait for explicit approval to publish the feature. Feature-document approval, general satisfaction, passing tests, or permission to continue coding is not publication approval.

If the user requests changes, return to implementation and E2E testing before presenting the revised result for approval. Approval of the tested feature for publication authorizes the single feature commit and PR described below; do not request a redundant confirmation for those steps.

## 5. Commit and open the pull request after approval

Once authorized:

1. Inspect the final diff and passing E2E results. Include only the approved feature's changes, and ensure the results cover the implementation being published.
2. Create exactly one commit for the feature, including its documentation, code and tests. Use the matching Conventional Commit type.
3. Describe the implemented solution in the commit body.
4. Add a `Docs: docs/<feature>.md` trailer for every feature document the commit creates or changes. A behaviour change without a matching document change is incomplete.
5. Push the feature branch and open the PR against the intended base. Describe the final solution, E2E coverage, commands and results, and list the same `Docs:` references in the PR body. Honor any narrower authorization, such as a request to commit only.

Example commit message:

```text
feat(search): add record search

Expose the requested search through the public API and show matching
records. Preserve entered filters after a failed request.

Docs: docs/record-search.md
```

Verify that the branch contains exactly one feature commit relative to its base before publishing the PR.

A request to commit alone does not authorize opening a PR. A request to open a PR for existing committed work does not authorize new commits. Opening a PR does not authorize merging it.

If further changes are requested after the feature commit, keep those edits uncommitted until explicitly authorized to amend the existing commit. Preserve the one-commit rule rather than adding a second feature commit. Obtain authorization before rewriting a published branch. Preserve `Docs:` trailers if the final commit message is edited during merging.

## Handoffs

State the actual feature document and branch when available, E2E coverage and results, and the current stage: awaiting document approval, implementing, testing, fixing failed tests, awaiting publication approval after passing tests, or authorized commit/PR completed. Identify any pending approval without repeating requests already satisfied. Never claim unperformed checks or unpublished GitHub actions.

## References

- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
- [Git trailers](https://git-scm.com/docs/git-interpret-trailers)
