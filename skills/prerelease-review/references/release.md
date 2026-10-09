# Accepted release: version and tag

Read this only for the release follow-up. The owner's explicit acceptance of a
specific prerelease report authorizes the version update, a release commit and
creation of the release tag in that workflow. Honor a narrower instruction.
Do not require the same acceptance again when it is already recorded.

## Resolve the version choice

If the owner asks to update the version or accepts the release but has supplied
neither a bump category nor an exact target version, ask:

> Should this be a patch, minor, or major release?

Wait for that answer before editing version files, committing or tagging.
An unambiguous exact target such as `1.4.0` is already a choice. A suggested bump
in the report, commit classification or feature size is not the owner's choice.

Derive the current version and tag convention from the accepted release line
and authoritative metadata. For standard X.Y.Z versions, patch increments Z,
minor increments Y and resets Z, and major increments X and resets Y and Z.
Respect [Semantic Versioning](https://semver.org/) and any documented package,
initial-development or preview-version policy. Ask about genuine version-policy
ambiguity rather than silently inventing a starting version or preview promotion.
If metadata and the last tag disagree, resolve that before choosing a new value.

## Check the accepted candidate

Retrieve the report, owner acceptance, selected version, tests and open findings.
Confirm the release checkout matches the accepted SHA and has no unrelated work
in the release diff. Check local and remote branch/tag state.

If product code, dependencies or configuration changed after the accepted review,
refresh the analysis, rerun the complete required suites and affected security
checks, publish the revised report and obtain acceptance of that new candidate.
Preserve existing approvals for portions unchanged; do not treat them as approval
of a different release candidate.

Do not turn failed, skipped or unavailable tests into passes based on acceptance.
Honor the consuming project's release prerequisites; describe any remaining
blocker precisely. If no usable prerelease report exists, run the review stage
first and present its results rather than manufacturing prior acceptance.

## Update and verify

Identify authoritative version files and their dependents: root/package/workspace
manifests, lockfile metadata, runtime version constants and established changelog
files, as applicable. Follow the project's synchronized or independent package
version policy. Inspect release scripts and lifecycle hooks before using them:
a version command may also commit, tag, publish packages or deploy.

Update the selected version consistently without unrelated dependency resolution,
automatic publication or duplicate tags. Follow required branch/PR protections
and repository instructions; never bypass them for a release. Do not rewrite
shipped migrations or previous release artifacts.

Check the resulting diff contains only expected release metadata/documentation.
Run the project's required release checks and the complete test suite against
the final versioned tree, with its mandated lifecycle. If this reveals defects
requiring product changes, return to remediation and candidate review. A pure,
verified version bump already authorized by acceptance needs no repeated approval.

## Commit, tag and verify

1. Commit only the authorized version/release changes using the established
   convention, for example `chore(release): prepare <version>`, with a reference
   to the prerelease issue. Stage explicit paths, not unrelated work.
2. Resolve the exact release commit after any required approved merge workflow.
   The tag must identify that final tested release commit, not the pre-bump SHA
   or an unmerged feature candidate. If merging changes the tested content,
   refresh verification and any affected acceptance.
3. Check whether the tag exists locally or remotely. Reuse an already completed
   matching release on retries; if the tag targets different content, stop and
   report the conflict. Never force-move, delete or overwrite a release tag.
4. Create an annotated tag (or follow the project's signing convention), naming
   the version and report issue. Verify the tag's resolved commit and complete
   tree against the tested release tree, including any changes from commit hooks.
   Unexpected tree changes require verification before release publication.
5. Follow existing authorization for remote publication. Acceptance alone
   authorizes the local commit/tag; pushing refs, opening/merging a PR, creating
   a GitHub Release, package publication and deployment require their own
   existing authorization or an explicitly adopted release policy. Reuse any
   such authorization rather than asking again. Push only the intended release
   branch/ref and tag, never every local tag, and never force-push.
6. For authorized publication, verify remote refs and report partial failures
   honestly. Resume from actual state without duplicating commits/tags. Keep a
   local-only release clearly labelled as such; do not imply it was published.
7. Update the prerelease issue with the decision reference, final test evidence,
   version, commit/tag and publication state. Close it only when its agreed
   release scope is complete. Report what succeeded and any pending action.

[Git tag documentation](https://git-scm.com/docs/git-tag) defines annotated and
signed tags. Preserve the project's existing identity/signing configuration.
