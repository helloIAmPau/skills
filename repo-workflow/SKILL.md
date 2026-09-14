---
name: repo-workflow
description: The contribution workflow for these repositories. An issue exists before work starts; one branch per issue, named after it; squash to a single commit before pushing; open a pull request and stop, because review and merge belong to the repository owner. Also covers the commit types that double as the only labels, what a commit body and a PR body are for, and keeping the README true when a decision contradicts it. Use before creating a branch, committing, pushing, force-pushing, or opening a pull request. What the container the work happens in is like is the agent-container skill.
---

# Working in these repositories

The path a change takes from issue to merge. The repository's own README is the
contract and wins wherever the two differ — read it first, every session,
because nothing you learned last time survived.

Working inside the agent container — the missing docker CLI, the host path the
daemon resolves bind mounts against, which env file to source, file ownership,
and what `gh` cannot do there — is the `agent-container` skill.

---

## 1. The path every change takes

```
0.  an issue exists   the task is described and numbered before work starts
1.  refresh main      git switch main && git pull --ff-only
2.  branch per task   git switch -c <type>/<issue>-<short-slug>
3.  implement         commit freely while the branch is open
4.  squash the branch one commit, carrying the Closes #<issue> footer
5.  push              git push -u origin <type>/<issue>-<short-slug>
6.  open a PR         title in conventional-commit form; body says why
7.  stop              review and merge are the owner's
```

Flatten the branch before pushing it:

```sh
git reset --soft main && git commit     # one commit, one message
```

Squashing happens **on the branch, not at the merge button**. The PR is merged
with a merge commit, so main carries the merge alongside the single commit the
work actually was, and the branch's message is the one that survives.

## 2. Rules that are not negotiable

- **Never commit to main.** Not a typo fix, not a README line. main moves only
  through a merged pull request.
- **Never force-push main.** Force-pushing your own open branch is expected.
- **Always cut from a freshly pulled main.** A branch cut from a stale one is
  how a conflict becomes someone else's problem later.
- **No issue, no work.** The issue is what the commit closes and what the
  branch is named after — `feat/7-task-ordering`, `fix/23-overdue-cutoff`. If
  there is no issue, the task is not described well enough to start; write one.
- **One label per issue, and it is a Conventional Commit type.** The same word
  appears on the issue, in the branch prefix and in the merged subject. There
  are no other labels — a taxonomy that does not survive into the history is
  one more thing to keep in sync.
- **One branch, one task, one commit.** A second concern appearing mid-branch
  gets its own issue and its own branch.
- **`Closes #<issue>` goes in the commit**, not only the PR body. A merge
  commit preserves the branch's message verbatim, so that footer is what closes
  the issue. Repeating it in the PR body is harmless and useful during review.
- **Rebase onto refreshed main, never merge main in.** History stays linear and
  the PR shows only its own work.
- **Never merge, never approve.** Pull requests are approved and merged by the
  repository owner and by nobody else. Open the PR and stop there. `gh`
  authenticates as the owner and would let you — that it can is not that you
  may.

## 3. Commit types

`feat` · `fix` · `refactor` · `perf` · `docs` · `build` · `ci` · `chore`

These are the repository's labels, and there are no others.

## 4. Writing the commit and the PR

- Subject in conventional-commit form, matching the issue's label.
- The body says **why**, not what — the diff already says what.
- A new dependency needs its reason in the commit body. These stacks are
  deliberately small.
- The PR body says why the change exists and what a reviewer should look at
  first. **If the README changed, say so** — that is the part worth reading
  carefully.

## 5. Keeping the README true

- A decision that contradicts the README changes the README **in the same
  commit**. A README that has drifted is worse than none.
- Something genuinely undecided goes in the README's Open questions — not in a
  code comment, and not in someone's memory.
- Prefer deleting code to guarding it with a flag.
- When an issue's scope changes, edit the issue too. The README's build order
  links the issues; the two drifting apart is the same failure as a drifted
  README.

---

## 6. Failure modes seen in practice

Not in any README — these are mistakes that actually happened here.

- **Check the PR is still open before force-pushing.** A branch was amended and
  force-pushed five times after its PR had already been merged. Every push
  succeeded and none of the work reached main, because the PR was closed. Run
  `gh pr view <n> --json state` before amending, and
  `git fetch && git log --oneline origin/main` before assuming main has not
  moved. Step 1 of §1 exists to catch exactly this.
- **Do not wait on CI after pushing.** Push and move on; run the e2e suite
  locally instead of watching the Action.
- **Do not ask for approval before writing code.** Write it, push it to the
  open branch, and describe what changed.
- **Assert before you patch.** When rewriting a README or an issue body with a
  script, assert the old string is present before replacing it. A silent
  no-match leaves the document claiming something the code no longer does, and
  that drift is the failure §5 is trying to prevent.
