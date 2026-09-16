---
name: repo-workflow
description: The contribution workflow for these repositories. An issue exists before work starts; the plan is agreed with the owner and written into the issue as a task list before any code, then resolved task by task; one branch per issue, named after it; one commit per branch. Implement then stop for local review — commit only on the owner's word, and push or open a pull request only on the owner's word too; the owner squash-merges. Also covers the commit types that double as the only labels, what a commit body and a PR body are for, and keeping the README true when a decision contradicts it. Use before creating a branch, committing, pushing, force-pushing, or opening a pull request. What the container the work happens in is like is the agent-container skill.
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
1.  plan together     read the issue; ask the owner for any implementation
                      detail still open; when it is all clear, write the plan
                      into the issue body as a task list where each item spells
                      out in detail what it will do — then wait for the owner to
                      approve it before any code
2.  refresh main      git switch main && git pull --ff-only
3.  branch per task   git switch -c <type>/<issue>-<short-slug>
4.  implement         work the issue's task list in order, one item at a time,
                      checking each off; then stop — the owner reviews locally
5.  commit on command only when the owner asks; one commit per branch,
                      carrying the Closes #<issue> footer
6.  push on command   git push -u origin <branch> — only when the owner asks
7.  open a PR         only when the owner asks; title in conventional-commit
                      form, body says why
8.  merge             the owner squash-merges; you never merge
```

Steps 5, 6 and 7 each wait for an explicit word from the owner. Implement the
change and stop; the owner reviews it locally before anything is committed, and
again before it is pushed or a PR is opened. Do not run ahead of these gates.

Step 1 is a gate of its own, at the other end: **nothing is branched or written
until the plan lives in the issue as a task list and the owner has approved it.**
Read the issue, ask the owner about anything the implementation leaves open, and
only once it is all clear turn the plan into a checklist on the issue body.
**Each item spells out in detail what you will do** — which files and functions,
what the change is, how you will verify it — so the owner is reviewing the
intended work, not a bare title, before any of it happens. Then wait for the
owner's approval of that plan — a fourth explicit word, before the
commit, push and PR ones. Only on it do you branch and start, implementing the
list one item at a time and checking each off as it lands. This is the one place
questions belong — see §6.

Keep the branch at **one commit**. When told to commit, amend the existing
commit rather than stacking a new one — flatten with:

```sh
git reset --soft main && git commit     # one commit, one message
```

The branch reaches main as a single commit two ways at once: it already *is*
one commit, and the owner **squash-merges** the PR. The branch's message is the
one that survives into main.

## 2. Rules that are not negotiable

- **Never commit to main.** Not a typo fix, not a README line. main moves only
  through a merged pull request.
- **Never force-push main.** Force-pushing your own open branch is expected.
- **Always cut from a freshly pulled main.** A branch cut from a stale one is
  how a conflict becomes someone else's problem later.
- **No issue, no work.** The issue is what the commit closes and what the
  branch is named after — `feat/7-task-ordering`, `fix/23-overdue-cutoff`. If
  there is no issue, the task is not described well enough to start; write one.
- **No approved plan, no code.** Before implementing, read the issue and ask the
  owner about anything the implementation leaves open. When it is all clear write
  the plan into the issue as a task list whose every item spells out in detail
  what it will do — enough for the owner to review the intended work before it
  happens — then **wait for the owner to approve it** — branching and the first
  line of code both wait on that word. Once the
  plan is approved, work the list without re-asking permission at every later
  step.
- **One label per issue, and it is a Conventional Commit type.** The same word
  appears on the issue, in the branch prefix and in the merged subject. There
  are no other labels — a taxonomy that does not survive into the history is
  one more thing to keep in sync.
- **One branch, one task, one commit.** A second concern appearing mid-branch
  gets its own issue and its own branch.
- **Commit only on the owner's word.** Implement the change and stop. The owner
  reviews it locally and tells you when to commit; until then the branch stays
  uncommitted.
- **Push and open the PR only on the owner's word.** Even with the commit made,
  do not push or open a PR until the owner asks. Two separate gates — one before
  committing, one before pushing — and the owner reviews at each.
- **`Closes #<issue>` goes in the commit**, not only the PR body. A squash-merge
  carries the branch's commit message into main, so that footer is what closes
  the issue. Repeating it in the PR body is harmless and useful during review.
- **Rebase onto refreshed main, never merge main in.** History stays linear and
  the PR shows only its own work.
- **Never merge, never approve.** Pull requests are approved and merged by the
  repository owner and by nobody else — the owner squash-merges. Open the PR and
  stop there. `gh` authenticates as the owner and would let you — that it can is
  not that you may.

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
  moved. Step 2 of §1 exists to catch exactly this.
- **Do not wait on CI after pushing.** Push and move on; run the e2e suite
  locally instead of watching the Action.
- **Get the plan approved first, then write the code without asking again.**
  Questions belong in the plan gate (§1, step 1): read the issue, clarify with
  the owner, write the task list into the issue, and wait for the owner to
  approve it. That approval is the first of four waiting gates. Once it comes,
  work through the list without pausing for approval to start each item — do not
  re-ask your way down the checklist. The remaining gates come later, at commit,
  push and PR: the owner reviews locally and says when. Describe what changed
  while they review.
- **Assert before you patch.** When rewriting a README or an issue body with a
  script, assert the old string is present before replacing it. A silent
  no-match leaves the document claiming something the code no longer does, and
  that drift is the failure §5 is trying to prevent.
