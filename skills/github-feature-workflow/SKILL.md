---
name: github-feature-workflow
description: Coordinate development in repositories adopting an issue-first GitHub workflow, with owner approval before implementation, mandatory E2E suites, and owner approval after passing tests before publishing one feature commit and a PR. Use when planning, implementing, or reviewing work under this agreement, or maintaining its README.md and AGENTS.md. Do not impose these approval gates on unrelated projects.
---

# GitHub feature workflow

This is the shared development agreement for developers and coding agents. GitHub issues coordinate the work. The project owner approves plans and decides when implementations are ready to commit and submit for review.

Plan approval and commit approval are separate. Never infer approval from silence, elapsed time, passing tests, or approval of another task. Honor approvals already given without asking for them again unless the relevant plan changes materially.

The feature lifecycle is: issue → owner approval of the issue → implementation → E2E tests. Failed tests return the work to implementation and testing. Passing tests lead to a second owner approval, then publication of the feature commit and pull request.

## Repository documentation

Maintain these files at the repository root:

- `README.md`: a user-targeted description of the app, its purpose, features, and usage. Distinguish planned capabilities from implemented ones.
- `AGENTS.md`: durable architecture, development conventions, and setup/verification commands. Link this skill to preserve the working agreement. Do not keep an issue list, per-issue implementation history, approval/status ledger, or feature progress here; retrieve those through the GitHub API and record task-specific evidence on the issue or PR.

Read existing repository instructions before editing. Preserve relevant existing content. A request to draft initial documentation authorizes that drafting, not application implementation or commits. Keep documentation and repository-local `.agents/skills/` current within approved feature work. Skills record durable definitions and conventions; use GitHub for issue status, approvals and execution evidence rather than duplicating a status ledger in skills.

## 1. Open an issue before implementation

Identify the intended repository from its checkout, remote, or the user's instructions. If the repository is unknown or inaccessible, present the proposal in the conversation and request the missing information or access. Do not invent issue numbers or claim an issue was published.

Keep issue proposals in the conversation or on GitHub. Never create local issue-draft files or directories, including copies of published issues.

For every feature, open a GitHub issue before implementation. Reuse the matching issue when continuing existing work. Include requirements, a proposed solution, meaningful code examples, and an execution plan. Every feature must have an E2E test suite that validates its acceptance criteria. Describe the E2E scenarios and expected observable outcomes in the issue so the owner approves implementation and test scope together.

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
4. Update documentation and technical decisions, then present the passing results for owner approval before publishing the PR.

## Approval
Awaiting explicit project-owner approval of this plan.
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

After plan approval and before implementation edits, preserve any existing uncommitted work, then check out local `main`, update it from `origin/main` with a fast-forward pull, and only then create the feature branch. Do not substitute branching directly from `origin/main` for this owner-required sequence. If the pull cannot fast-forward, resolve the divergence without discarding work before proceeding.

```text
git switch main
git pull --ff-only origin main
git switch -c <label>/<issue-number>-<issue-title>
```

Use the real issue number. Convert the title to a branch-safe slug: lowercase it, replace spaces with hyphens, remove invalid punctuation, and collapse repeated hyphens while retaining its meaning.

Example: issue `#42`, titled `Add weight logging`, labelled `feat`:

```text
feat/42-add-weight-logging
```

Keep the feature on its own branch and preserve unrelated user changes. Do not implement directly on the default branch.

Implement the approved plan while sharing progress so the user can review the code and provide insights during development. Incorporate feedback within the approved scope. If feedback or discoveries materially change the scope or solution, update the issue and obtain approval for the revised portion before implementing it.

Add or extend the feature's E2E suite alongside the implementation. Update the documentation and decision record where needed. Leave the implementation and tests uncommitted through testing and owner review. Do not make WIP, checkpoint, automatic, or intermediate commits, including to enable remote review.

## 3. Run E2E tests and return to implementation on failure

Every feature requires an E2E suite that checks the feature's validity through observable behaviour in the running stack. Cover the approved acceptance criteria, including relevant failure cases. Extend existing suites where appropriate; a generic health check alone does not validate unrelated feature behaviour.

Keep the E2E test layout flat: all test files live directly in `tests/e2e/`, without nested feature or platform directories. Distinguish features through filenames.

This project uses E2E tests only. Exercise the real stack through its public interfaces without mocked transports or in-process application tests. The developer or coding agent manages each test run through separate commands, in this order:

1. Start the entire development stack with `npm run develop` and confirm every service is ready, including application rebuild/restart watchers. Use the base Compose file and development override. If a previous test stack is still running, tear it down before starting this run.
2. Run the project's migrations against that running database and wait for success before testing. If migrations have not been implemented, verify and report that fact; do not invent a migration command.
3. Run the complete E2E suite with `npm test` as a separate command.
4. Tear down the entire development stack after the suite finishes, whether it passes or fails. Preserve persisted data unless its removal was explicitly authorized. If setup or migrations fail, tear down the partially started stack as well.

Keep this lifecycle in the skill instructions, not in test scripts. `npm test`, test hooks and test helpers must not start or stop the stack, run migrations, or wrap this sequence in an orchestration script. Tests may inspect running containers or temporarily inject and restore a failure to validate an acceptance criterion. Do not substitute a partial stack, a directly launched application, or the production runtime stack for the required development verification.

Always run the complete E2E suite with `npm test`, including every feature and existing regression coverage. Do not add or use feature-specific test scripts such as `test:mobile`. If tests fail, return to implementation, correct the cause, and repeat the complete startup, migration, test and teardown sequence. Continue within the approved scope without requesting renewed permission for routine fixes. Do not skip, remove, or weaken valid assertions merely to obtain a passing result.

Tests that could not run are not passing tests. Report any environment or access blocker and resolve it within the available authorization; do not advance to publication while required E2E verification remains incomplete.

## 4. Wait for owner approval after tests pass

Once the required E2E suites pass against the current implementation, present the concrete changes and test results for owner review. Wait for explicit approval to publish the feature. Issue approval, general satisfaction, passing tests, or permission to continue coding is not publication approval.

If the owner requests changes, return to implementation and E2E testing before presenting the revised result for approval. Approval of the tested feature for publication authorizes the single feature commit and PR described below; do not request a redundant confirmation for those steps.

## 5. Commit and open the pull request after approval

Once authorized:

1. Inspect the final diff and passing E2E results. Include only the approved feature's changes, and ensure the results cover the implementation being published.
2. Create exactly one commit for the feature, including its tests and documentation. Use the matching Conventional Commit type.
3. Describe the implemented solution in the commit body and reference the issue and any relevant subissues.
4. Include a GitHub closing keyword for every issue fully resolved. Use `Closes #<number>`, a supported form of the requested `close` keyword. Use `Refs #<number>` for related issues that remain open.
5. Push the feature branch and open the PR against the intended base. Describe the final solution, E2E coverage, commands and results, and include the closing references in the PR body too. Honor any narrower authorization, such as a request to commit only.

Example commit message:

```text
feat(weight): add daily weight logging

Save measurements with the selected date and session, then refresh
that day's summary. Keep form values available after a failed save.

Closes #42
Closes #43
Refs #40
```

Only close issues and subissues actually completed. Do not close an unfinished parent merely because a subissue is resolved. Verify that the branch contains exactly one feature commit relative to its base before publishing the PR.

A request to commit alone does not authorize opening a PR. A request to open a PR for existing committed work does not authorize new commits. Opening a PR does not authorize merging it.

If further changes are requested after the feature commit, keep those edits uncommitted until explicitly authorized to amend the existing commit. Preserve the one-commit rule rather than adding a second feature commit. Obtain authorization before rewriting a published branch.

Closing keywords take effect when the relevant work reaches the repository's default branch. Do not promise automatic closure on opening a PR or merging into a non-default branch. Preserve closing references if the final commit message is edited during merging.

## Handoffs

State the actual issue and branch when available, E2E coverage and results, and the current stage: awaiting issue approval, implementing, testing, fixing failed tests, awaiting publication approval after passing tests, or authorized commit/PR completed. Identify any pending approval without repeating requests already satisfied. Never claim unperformed checks or unpublished GitHub actions.

## References

- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
- [GitHub issue linking and closing keywords](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue)
