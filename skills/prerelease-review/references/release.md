# Accepted release: version commit, tag and PR

Read this only for the release follow-up. The owner's explicit acceptance of a
specific prerelease report authorizes the version update, one release commit,
local tag creation, pushing the release branch and opening a PR after verification.
Honor narrower instructions and reuse acceptance already given. The handoff is
an open PR for the owner to merge. Never push directly to the default branch,
merge the PR or enable auto-merge as part of release acceptance.

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
in the release diff. Check local and remote branch/tag state, the intended PR
base, and existing release branches/PRs so resumed work does not create duplicates.

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

Create or resume a separate release branch for the prerelease issue, following
the consuming project's branch conventions. Start from the accepted candidate;
preserve unrelated local work, using an isolated checkout when necessary. Never
make the version commit on the default branch.

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

## Commit, prepare the tag and open the PR

1. Stage only the authorized version/release files and create exactly one
   Conventional Commit, for example `chore(release): prepare <version>`. Its
   body **must include `Closes #<prerelease-issue>`**, not only `Refs`. Preserve
   the project's conventions for other issue/subissue references; do not close
   unfinished remediation issues.
2. Verify that the release commit's complete tree matches the final tested
   versioned tree, including any changes from commit hooks. Unexpected changes
   require verification before publication.
3. Check whether the tag exists locally or remotely. Reuse an already completed
   matching release on retries; if the tag targets different content, stop and
   report the conflict. Never force-move, delete or overwrite a release tag.
4. Create an annotated local tag (or follow the project's signing convention)
   on the tested **version commit**, naming the version and report issue. Verify
   its resolved commit and tree. Keep the tag local until the owner merges the
   PR unless the owner explicitly chooses earlier tag publication; pushing a
   tag may deploy even while its PR is open.
5. Push only the release branch and open a PR against the intended base. Put
   `Closes #<prerelease-issue>` in the PR body as well, alongside the accepted
   report, selected version, exact test evidence and tag target/publication
   state. Do not push the default branch or use a blanket tag push.
6. Verify the remote branch and open PR. Update the prerelease issue with the
   decision reference, final tests, version, release commit, tag status and PR
   URL. Hand the PR to the owner and stop; do not merge or enable auto-merge.

Example release commit:

```text
chore(release): prepare <version>

Synchronize the version metadata for the accepted prerelease candidate.

Closes #<prerelease-issue>
```

The closing reference completes the prerelease issue when the commit reaches
the repository's default branch. Creating a local commit/tag or opening the PR
does not close it. Do not manually close the report or substitute deployment
completion for the commit's closing reference. If a project targets another
base branch, explain that GitHub closure waits until the default branch is
reached. Preserve the closing reference in any owner-edited merge message.

## After the owner merges

Confirm the actual merge and release target before publishing the prepared tag.
If review changes the tagged commit or squash/rebase replaces it, resolve the
target with the owner before publication; never publish a stale tag or silently
move an existing one. Changes to tested content require refreshed verification
and any affected acceptance.

Follow existing owner authorization and the project's release policy for tag
publication and its deployment effects. Reuse authorization already given;
creating a GitHub Release or publishing packages remains separate scope. Push
only the intended tag, verify its remote target and record actual publication
and deployment results. Issue closure from the merged version commit is not
proof of successful deployment. Report failures or pending jobs accurately
without treating the report as still awaiting its version commit.

[Git tag documentation](https://git-scm.com/docs/git-tag) defines annotated and
signed tags. Preserve the project's existing identity/signing configuration.
[GitHub closing keywords](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue)
define when the prerelease issue closes.
