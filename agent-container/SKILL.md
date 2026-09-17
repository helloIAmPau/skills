---
name: agent-container
description: What an agent needs to know about the container it is running in. The docker CLI is missing and the daemon is the host's; /workdir is a different path to that daemon, which breaks bind mounts silently; sleep is blocked; there are two env files and only one of them works from here; everything created lands owned by root; and gh cannot push workflows or edit an issue body. Load at the start of any session that will run the stack, use docker or compose, touch an .env file, run migrations or the test suite, or edit an issue or pull request body.
---

# The agent container

Every session begins in a fresh one — nothing learned last time survived. How a
change travels from issue to merge is the `repo-workflow` skill; this is only
what the machine underneath is like.


## What is already there

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
curl --retry 60 --retry-delay 2 --retry-all-errors "$BASE_URL/auth/health"
```

## What is missing

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

## The path the daemon sees

**This is the one that bites.** The socket is the host's daemon, so it resolves
build contexts and bind mounts against the *host* filesystem — and `/workdir`
is the repository only inside the container. Make the two agree, then work from
the host path:

```sh
mkdir -p /home/helloiampau/develop
ln -sfn /workdir /home/helloiampau/develop/<repo>
cd /home/helloiampau/develop/<repo>
```

That path is this host's, not a convention. It is wherever the host bind-mounts
the repository from, and `docker inspect` on a running container will show it if
it is ever anything else.

Running compose from `/workdir` is not a harmless alias: a host `/workdir`
exists and belongs to something else. Doing it once pointed `./data/postgres`
at a foreign PostgreSQL data directory and the container refused to start.
Check a mount if anything looks wrong:

```sh
docker inspect <repo>-postgres-1 --format '{{range .Mounts}}{{.Source}}{{end}}'
```

## Which env file

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

## File ownership

The agent is root and the person is not, so anything it creates lands owned by
root and the person cannot edit it. Hand everything back:

```sh
chown -R node:users <paths>
chown -h node:users data data/postgres   # -h: do not follow, they may be links
```

## What `gh` cannot do here

`gh` authenticates as the repository owner. Merging is never yours — that is a
rule, not a limitation, and it lives in the `repo-workflow` skill. These two are
the environment:

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
