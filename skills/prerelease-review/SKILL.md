---
name: prerelease-review
description: Review a release candidate against changes since the last release tag, linked issues, tests, code quality and white-box/black-box security checks; publish a GitHub prerelease report and handle the owner's subsequent release acceptance, version bump and tag. Use for prerelease analysis and its release follow-up, not ordinary feature implementation.
---

# Prerelease review

Deliver an evidence-based GitHub report before the owner decides whether to fix
findings or release. Read the consuming project's instructions and applicable
style, architecture, testing and release conventions. Resolve repository,
remotes, branches, commands and version files from that project; do not assume
a language, package manager, hosting provider or monorepo version policy.

## Authorization and stages

A request to run this workflow authorizes the analysis, local test/security
checks and creation or update of its prerelease issue. A request merely to
create or edit this skill does not authorize running it.

Keep these stages distinct:

1. **Review:** inspect changes, check requirements against tests, rerun all suites,
   review the whole first-party codebase, assess vulnerabilities and publish the
   report. Findings are proposals; leave application code, dependencies and
   existing tests unchanged.
2. **Remediation:** the owner selects findings to address or asks for new issues.
   Follow the consuming project's implementation and publication approvals.
   Do not infer implementation permission from the review request.
3. **Release:** only after explicit acceptance of the report, follow
   [release.md](references/release.md) to update the version and create the tag.
   If neither the bump type nor an exact target version was supplied, ask
   **"Should this be a patch, minor, or major release?"** before changing versions.
   Never infer that choice from commit types or a recommendation in the report.

Keep report revisions, task status and approval references on GitHub. Reuse
authorization already given. Resolve any conflict with the project's workflow
before a dependent action; explain the exact instruction if it requires a pause.

## 1. Establish the release range

- Inspect working-tree changes, nested repositories/submodules, the current
  branch, upstream and intended release target. Pin the candidate's full commit
  SHA before analysis. Preserve unrelated edits; use an isolated checkout when
  necessary. Identify uncommitted work separately, never as part of that SHA.
- Check the selected remote and fetch relevant history and tags without force
  or overwriting conflicting tags. Record unavailable access or shallow history.
  Do not mistake missing local tags for proof this is the first release.
- Select the latest applicable release tag preceding the candidate on its
  release line, using the project's tag/package convention and reachability.
  Do not choose by tag date or highest version across unrelated branches.
  Include lightweight tags. Exclude preview or unrelated package tags unless
  the project's release policy selects them. Explain the chosen baseline.
- If complete history and remote inspection establish there is no applicable
  prior release tag, label this an initial release and review all history through
  the candidate. If the baseline remains ambiguous, ask while continuing
  independent code review.
- Collect **all commits** in the range, including merged branch commits and full
  messages/bodies/footers, plus the net diff, renames, deletions, migrations,
  dependency/lockfile changes, configuration and submodule changes. A first-parent
  traversal may help select a release tag; it must not omit merged commits from
  the inventory.
- Extract issue references from commit subjects, bodies and footers, including
  closing/reference keywords, qualified repository references and issue URLs.
  Fetch each issue's description, acceptance criteria and relevant approval or
  scope-changing comments. PR references are not automatically issue numbers;
  follow associated PRs when needed. Keep unlinked and inaccessible requirements
  visible rather than inventing them.
- Account for reverted and superseded work. Include it in the historical
  inventory without describing absent functionality as shipped.

## 2. Compare intended behavior with tests

Build a traceability table:
**issue/approved requirement → implementation → test and actual assertions →
observed result → gap**.

Read test bodies and the helpers/fixtures that affect their assertions. Check
happy paths and applicable refusal cases, permissions, edge cases and regression
behavior. Flag missing tests, assertions that allow incorrect behavior, skipped
or exclusively selected tests, and tests that encode a different requirement.
A passing test or a matching test name alone does not establish coherence.

Use approved requirements as the expected behavior. Keep proposals, unresolved
decisions and contradictions separate. Do not turn optional ideas into defects,
or close an issue merely because its commit claims to implement it.

## 3. Rerun all tests and required checks

Discover suites and checks from project instructions, manifests, CI and test
configuration. Run **every applicable suite**, including the full regression/E2E
suite; focused checks supplement it. Run required build, lint and type checks
where defined. Follow the project's installation, startup, readiness, migration,
host/container and teardown contracts.

Use the pinned candidate with reproducible dependencies and fictional test data.
Record commands, working directory, relevant nonsecret environment, tool/runtime
versions, timestamp, candidate SHA, exit codes and pass/fail/skip counts. Preserve
useful redacted failure output and artifact links. Stop the resources started
for the review on success or failure, preserving existing data and user services.

Classify failed, blocked, skipped and not-run checks separately. Missing tools
or services do not count as a pass. Investigate failures without weakening tests
or fixing product code. Any diagnostic rerun retains the initial failure and
explains the changed conditions; a flaky pass does not erase evidence.

## 4. Review the full codebase

Inventory current first-party source, tests, migrations, build/deployment
configuration and public entrypoints. Review the whole current codebase,
including unchanged shared code; use the release diff to prioritize regressions.
Inspect changed first-party submodules under their own conventions. Dependencies,
generated outputs and vendored code require suitable artifact/dependency checks
rather than claims of exhaustive manual source review.

Cover correctness and integration; established code style and architecture;
maintainability, duplication and complexity; error handling and data integrity;
dead code, unused dependencies and unused functionality; and concrete improvements.
Before claiming something is unused, check imports, dynamic registration, public
exports, runtime entrypoints, scripts, configuration and external consumers.
A text-search miss alone is insufficient.

Tie findings to candidate file/line permalinks, observable impact, supporting
evidence and a proportionate remedy. Distinguish defects from optional cleanup.
State which areas were reviewed and any omissions; do not call a partial review
complete.

## 5. Assess vulnerabilities

Read [security.md](references/security.md) and perform both source-informed
white-box assessment and black-box checks against the running candidate's public
interfaces. Dependency scanning alone satisfies neither assessment.

Report applicable threats, tested boundaries, tools and exact versions, current
advisory sources, reproducible evidence, confidence and limitations. Keep confirmed
vulnerabilities distinct from suspicious patterns and unverified scanner alerts.
A clean scan means no findings in that check, never proof of no vulnerabilities.

## 6. Publish the prerelease report

Read [report.md](references/report.md). Search for an existing prerelease issue
for the same release line/candidate and update it when resuming; otherwise create
one in the intended GitHub repository. An ordinary issue is the deliverable;
do not create a GitHub Release or tag during analysis.

Include the complete range, linked issues, feature summary and Conventional
Commit change entries, traceability matrix, test results, code/security findings,
limitations and recommended next actions. Give findings stable IDs so the owner
can select them. Respect repository visibility when publishing security evidence.

Use structured issue-tool arguments or a temporary body file outside the checkout
for CLI submission. Do not create an issue-draft/status directory in the project.
Verify the resulting issue content and URL. If GitHub publication is blocked,
provide the report in the conversation and identify the blocker without claiming
an issue exists.

Present the issue URL, the strongest findings and actual test outcome, then leave
the decision with the owner. Do not automatically create fix issues, apply fixes,
bump a version or tag. After authorized fixes land, retain finding IDs, refresh
the range and evidence, rerun the complete required suites and affected security
checks, and present the revised report for acceptance.
