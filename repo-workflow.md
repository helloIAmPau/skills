---
name: repo-workflow
description: Contribution workflow and agent-container setup for helloIAmPau's repositories. Use before creating a branch, committing, pushing, force-pushing, or opening a pull request in these repos, and at the start of any session running inside the agent container. Covers issue-first branching, squash-before-push, one branch one commit, what gh can and cannot do, installing the docker CLI, the host path the daemon resolves bind mounts against, which env file to source, and handing files back to the person who owns them.
---

# Working in these repositories

Two things this skill is for: the path a change takes from issue to merge, and
what an agent needs to know about the container it is running in. The
repository's own README is the contract and wins wherever the two differ — read
it first, every session, because nothing you learned last time survived.

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
  repository owner and by nobody else. Open the PR and stop there.

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

## 6. The agent container

### What is already there

| | |
|---|---|
| `/workdir` | the repository, bind-mounted |
| `/var/run/docker.sock` | the **host's** docker daemon |
| user | `root` |
| network | isolated from the Compose network; internet and host both reachable |
| host address | `172.17.0.1` — the default gateway |
| node | 20, so `node --test` needs `--experimental-websocket` |

`sleep` is blocked in the agent shell. Wait on the thing itself:

```sh
curl --retry 60 --retry-delay 2 --retry-all-errors "$BASE_URL/api/health"
```

### What is missing

The docker CLI. The daemon is the host's through the socket, so install the CLI
and the compose plugin only:

```sh
apt-get update -qq
apt-get install -y -qq ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" > /etc/apt/sources.list.d/docker.list
apt-get update -qq
apt-get install -y -qq docker-ce-cli docker-compose-plugin
```

`psql` is not installed and is not needed — `npm run db` runs it in the
container.

### The path the daemon sees

**This is the one that bites.** The socket is the host's daemon, so it resolves
build contexts and bind mounts against the *host* filesystem — and `/workdir`
is the repository only inside the container. Make the two agree, then work from
the host path:

```sh
mkdir -p /home/helloiampau/develop
ln -sfn /workdir /home/helloiampau/develop/<repo>
cd /home/helloiampau/develop/<repo>
```

Running compose from `/workdir` is not a harmless alias: a host `/workdir`
exists and belongs to something else. Doing it once pointed `./data/postgres`
at a foreign PostgreSQL data directory and the container refused to start.
Check a mount if anything looks wrong:

```sh
docker inspect <repo>-postgres-1 --format '{{range .Mounts}}{{.Source}}{{end}}'
```

### Which env file

Source `.env.claude`, **never** `.env.develop`. `.env.claude` points the base
URL at `172.17.0.1`, the only address this container can reach; `.env.develop`
points at localhost. Starting the stack from the wrong one fails in a way that
looks unrelated — Caddy matches on Host and 404s everything, or a magic-link
test fails because the emailed link is unreachable.

```sh
cd /home/helloiampau/develop/<repo>
source ./.env.claude
export COMPOSE_FILE=docker-compose.yml:docker-compose.develop.yml

docker compose up -d --build
npm run migrate
npm test
docker compose down          # leave the host as it was found
```

### File ownership

The agent is root and the person is not, so anything it creates lands owned by
root and the person cannot edit it. Hand everything back:

```sh
chown -R node:users <paths>
chown -h node:users data data/postgres   # -h: do not follow, they may be links
```

### What `gh` cannot do

`gh` authenticates as the repository owner. Three limits:

- **Merging.** Never. See §2.
- **Pushing under `.github/workflows/`.** The token lacks the `workflow` scope
  and the REST contents API refuses it too. The refresh is a device flow, so a
  person must run it in their own shell:
  `gh auth refresh --hostname github.com -s workflow`
- **`gh pr edit` and `gh issue edit`** fail here with a Projects-classic
  deprecation error. Edit bodies and titles through the REST API instead:
  ```sh
  gh api -X PATCH repos/<owner>/<repo>/pulls/<n> --input body.json
  gh api -X PATCH repos/<owner>/<repo>/issues/<n> -f title='...'
  ```

---

## 7. Failure modes seen in practice

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
