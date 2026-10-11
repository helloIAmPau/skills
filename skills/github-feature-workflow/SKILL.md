---
name: github-feature-workflow
description: Coordinate development in repositories adopting an issue-first GitHub workflow, with user-approved plans, mandatory E2E suites, one feature commit and an agent-opened PR after checks pass; merging belongs to the owner. Use under this selected agreement; documentation edits alone do not adopt it. Do not impose this workflow on unrelated projects.
---

# GitHub feature workflow

Apply only where this issue-first agreement has been selected. This is the shared development agreement for developers and coding agents. GitHub issues coordinate the work. The user approves plans and merges PRs. Agents complete approved work, verify it, then publish one feature commit and an open PR without another approval gate.

Read higher-priority environment instructions, the consuming project's applicable instructions and existing user authorization before acting. An approved plan authorizes implementation and, once the approved work and required checks are complete, its single commit, branch push and PR. Honor narrower instructions such as local work only. Never infer plan approval from silence, elapsed time, passing tests or approval of another task. Do not reopen settled approvals unless the plan changes materially. Merging remains the owner's action; do not merge or enable auto-merge.

The feature lifecycle is: issue → user approval of the plan → implementation → passing E2E tests → one feature commit → branch push and open PR → owner merges. Failed tests return the work to implementation and testing; there is no second approval gate before opening the PR.

## Repository documentation

Maintain these files at the repository root:

- `README.md`: a user-targeted description of the app, its purpose, features, and usage. Distinguish planned capabilities from implemented ones.
- `AGENTS.md`: durable architecture, development conventions, and setup/verification commands. Link this skill to preserve the working agreement. Do not keep an issue list, per-issue implementation history, approval/status ledger, or feature progress here; retrieve those through the GitHub API and record task-specific evidence on the issue or PR.

Read existing repository instructions before editing. Preserve relevant existing content. A request to draft initial documentation authorizes that drafting, not application implementation or commits. Keep documentation and applicable installed skills current within approved feature work, respecting their actual location and separate repository boundaries. Skills record durable definitions and conventions; use GitHub for issue status, approvals and execution evidence rather than duplicating a status ledger in skills.

## 1. Open an issue before implementation

Identify the intended repository from its checkout, remote, or the user's instructions. If the repository is unknown or inaccessible, present the proposal in the conversation and request the missing information or access. Do not invent issue numbers or claim an issue was published.

Keep issue proposals in the conversation or on GitHub. Never create local issue-draft files or directories, including copies of published issues.

For every feature under this agreement, open a GitHub issue before implementation unless existing instructions already authorize the work. Reuse the matching issue when continuing existing work. Include requirements, a proposed solution, meaningful code examples, and an execution plan. Every feature must have an E2E test suite that validates its acceptance criteria. Describe the E2E scenarios and expected observable outcomes in the issue so the user approves implementation and test scope together.

Use this structure:

````markdown
## Requirements
Describe the user need, expected behaviour, scope, and acceptance criteria.

## Proposed solution
Explain the approach, affected components, relevant data/API changes,
tradeoffs, dependencies, and unresolved decisions.

## Code examples
```text
Illustrative code, pseudocode, or request/response examples specific to
this solution. Mark them as proposals rather than existing implementation.
```

## Execution plan
1. Concrete, ordered implementation steps.
2. Add or extend the feature's E2E suite to validate its acceptance criteria against the running stack.
3. Run E2E tests; return to implementation and rerun tests on failure.
4. Update documentation and technical decisions, then publish one feature commit and an open PR with the passing results. Leave merging to the owner.

## Approval
Awaiting explicit user approval of this plan.
````

Label each issue with a Conventional Commit type. Use the same primary type for the issue label, branch prefix, and eventual commit. The working vocabulary is:

| Label | Purpose |
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

Conventional Commits defines `feat` and `fix` and permits additional types; this vocabulary is the project's convention. Use `feat` for a feature even when it includes tests and documentation. Apply type labels to subissues as well. Create a missing type label in the intended repository as part of the requested issue workflow, subject to available permissions; do not change unrelated repository settings.

Present the completed issue and wait for explicit plan approval. Read-only investigation and illustrative issue examples are allowed before approval. Application code, scaffolding, dependency changes, migrations, and implementation tests must wait.

Approval may be given in the working conversation or on GitHub. Record the approval reference and approved plan revision in the issue when possible. Never fabricate an approval.

## 2. Implement on a separate branch

When continuing work, inspect the current branch, matching issue and local/remote
differences; resume an existing matching feature branch. Do not restart approved
work or inherit unrelated local commits accidentally.

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
git switch -c <label>/<issue-number>-<issue-title>
```

Use the real issue number. Convert the title to a branch-safe slug: lowercase it, replace spaces with hyphens, remove invalid punctuation, and collapse repeated hyphens while retaining its meaning.

Example: issue `#42`, titled `Add record search`, labelled `feat`:

```text
feat/42-add-record-search
```

Keep the feature on its own branch and preserve unrelated user changes. Do not implement directly on the default branch.

Implement the approved plan while sharing progress so the user can review the code and provide insights during development. Incorporate feedback within the approved scope. If feedback or discoveries materially change the scope or solution, update the issue and obtain approval for the revised portion before implementing it.

Add or extend the feature's E2E suite alongside the implementation. Update the documentation and decision record where needed. Leave the implementation and tests uncommitted until the approved work and required verification are complete. Do not make WIP, checkpoint or intermediate commits, including to enable remote review.

## 3. Run E2E tests and return to implementation on failure

Every feature requires an E2E suite that checks the feature's validity through observable behaviour in the running stack. Cover the approved acceptance criteria, including relevant failure cases. Extend existing suites where appropriate; a generic health check alone does not validate unrelated feature behaviour.

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

## 4. Check the completed work before publication

Once the approved scope is complete and the required E2E suites pass against the current implementation, inspect the final diff and prepare the PR's change summary and test evidence. Proceed to publication without asking for a second approval. Passing tests alone do not complete unfinished acceptance criteria or authorize unrelated work.

If the user requests changes before publication, incorporate them and rerun the affected required verification, including the complete E2E lifecycle for feature changes. Obtain plan approval only for a material change beyond the approved scope. Honor an explicit request to hold publication.

## 5. Commit, push and open the pull request

Once the approved work and required verification are complete:

1. Inspect the final diff and passing E2E results. Include only the approved feature's changes, and ensure the results cover the implementation being published.
2. Create exactly one commit for the feature, including its tests and documentation. Use the matching Conventional Commit type.
3. Describe the implemented solution in the commit body and reference the issue and any relevant subissues.
4. Include a GitHub closing keyword for every issue fully resolved. Use `Closes #<number>`, a supported form of the requested `close` keyword. Use `Refs #<number>` for related issues that remain open.
5. Push only the feature branch and open the PR against the intended base. Describe the final solution, E2E coverage, commands and results, and include the closing references in the PR body too. Honor any narrower authorization, such as a request to commit only. Never push the feature directly to the default branch.
6. Verify the PR and remote branch, then leave the PR open for the owner. Do not merge it, enable auto-merge or bypass branch protections.

Example commit message:

```text
feat(search): add record search

Expose the requested search through the public API and show matching
records. Preserve entered filters after a failed request.

Closes #42
Closes #43
Refs #40
```

Only close issues and subissues actually completed. Do not close an unfinished parent merely because a subissue is resolved. Verify that the branch contains exactly one feature commit relative to its base before publishing the PR.

A request explicitly limited to committing does not authorize opening a PR. A request only to open a PR for existing committed work does not authorize additional implementation. The ordinary approved-task workflow above includes its commit and PR; merging remains the owner's action.

If further changes are requested after the feature commit, keep those edits uncommitted until explicitly authorized to amend the existing commit. Preserve the one-commit rule rather than adding a second feature commit. Obtain authorization before rewriting a published branch.

Closing keywords take effect when the relevant work reaches the repository's default branch. Do not promise automatic closure on opening a PR or merging into a non-default branch. Preserve closing references if the final commit message is edited during merging.

## Handoffs

State the actual issue and branch when available, E2E coverage and results, and the current stage: awaiting plan approval, implementing, testing, fixing failed tests, or PR open awaiting the owner's merge. Link the PR after publication. Identify any narrower instruction holding publication without inventing another approval gate. Never claim unperformed checks or unpublished GitHub actions.

## References

- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
- [GitHub issue linking and closing keywords](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue)
