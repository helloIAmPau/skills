# Prerelease issue report

Use this as the report structure, adapting length to the project. Keep every
requested assessment visible, even when blocked. Link to immutable commit/file
references where possible. Do not copy raw secrets or private user data.

Title: `chore(release): prerelease review for <candidate short SHA>`.
Use an existing appropriate release/type label when available.

## Candidate and conclusion

- Repository, branch/release line, review date and full candidate SHA.
- Prior release tag and resolved SHA, comparison link and why that baseline
  applies; or a verified initial-release history range.
- Working-tree/submodule state, review environment and report revision.
- Recommendation: ready for owner review / fixes recommended / assessment
  incomplete. Separately state whether required tests passed.
- Blocking risks, unverified areas and suggested actions. The recommendation
  does not constitute owner acceptance.

## New features and changes

Explain delivered behavior in plain language, linked to its source issues.
Then provide a deduplicated Conventional Commit summary covering every material
net change, including fixes, maintenance, security and breaking changes:

| Conventional Commit entry | Issues/PRs | Source commits | User-visible effect |
| --- | --- | --- | --- |
| `feat(scope): describe delivered behavior` | Links | SHA links | Outcome |
| `fix(scope): describe corrected behavior` | Links | SHA links | Outcome |

Use `type(optional-scope)!: description` when the change breaks compatibility,
and explain each `BREAKING CHANGE:` with migration/action requirements. Follow
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) and the
project's established types. Summarize nonconforming commits in this format
without rewriting Git history. Merge commits do not duplicate feature entries.

Attach a compact inventory covering all commits in the range, grouped where
useful. Include unlinked changes, reverts, superseded work and unavailable issues;
do not list reverted features as delivered.

## Requirements and test coherence

| Issue and approved requirement | Implementation | Test and assertion | Result | Gap/finding |
| --- | --- | --- | --- | --- |

Keep intended behavior separate from what the tests currently require. Note
unsettled requirements and approval references where they affect the assessment.

## Verification results

| Suite/check | Command and working directory | Result/exit code | Pass/fail/skip counts | Evidence |
| --- | --- | --- | --- | --- |

Include environment/tool versions, candidate identity and timestamps. Report
setup, readiness, migrations and teardown when applicable. List failed, blocked,
skipped, not-run and flaky checks explicitly, with reasons and diagnostic reruns.
Record all expected suites so absent execution is visible.

## Code review and security assessment

Describe the reviewed areas and omissions. Summarize white-box coverage,
dependency/advisory results and black-box targets/cases separately.

| Finding ID | Area | Severity | Confidence | Summary | Evidence | Proposed action |
| --- | --- | --- | --- | --- | --- | --- |

For each actionable finding, include location or endpoint, requirement violated
where relevant, reproduction/observed behavior, impact, remedy and verification.
Distinguish release-blocking recommendations, other defects and optional style,
maintainability, dead-code or functionality cleanup. Explain severity; counts
alone are not an assessment. Label known pre-existing findings.

## Owner decision and follow-up

Initial state: awaiting owner review. List findings that could become follow-up
issues with proposed acceptance criteria, without creating those issues yet.

When the owner replies, record their actual decision and its reference:
selected findings and authorized follow-up issue links, or accepted report
revision/candidate SHA and any explicitly accepted remaining risks. Do not infer
acceptance from silence, passing tests, issue closure or a recommended bump.

Record the chosen patch/minor/major or exact target version only once provided.
After release, append the version, release commit, tag/target SHA and actual
publication status. Keep prior review evidence and decision history available;
do not close the report as released before the authorized release steps succeed.
