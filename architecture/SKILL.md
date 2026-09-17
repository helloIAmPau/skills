---
name: architecture
description: How a project is shaped in these repositories — every workspace under `workspaces/@<project>/*`, a service per path prefix, an ingress that routes by prefix, and one image built for each service from a single root Dockerfile. Covers what a service is called, and in particular that the data API is GraphQL and is named for it, where the health check lives, why there is no `.dockerignore`, and how the web app is built — a `web` workspace that renders a server shell and hydrates a client from one bundle each. Load before adding a service or a workspace, naming either, deciding what belongs in a library, changing the ingress, touching the Dockerfile, or building or scaffolding the web app.
---

# Architecture

Read out of three repositories that share the shape. Where a repository and
this file disagree, the repository wins and this file needs a change.

## Every workspace lives under `workspaces/@<project>/`

There is exactly one place a package can be:
`workspaces/@<project>/<name>`. The npm scope is the project's own name —
`@pul.se` in that repo, and the same shape in the other two — and every
service and every library is a directory directly under it. The root
`package.json` declares `"workspaces": ["workspaces/@<project>/*"]` and carries
nothing else of its own; the lockfile beside it is the only one, because npm
hoists a single `node_modules` to the root.

- **A workspace's directory name is the service or library name**, and its
  package name is `@<project>/<name>`. That one name is what the Dockerfile
  builds (`--workspace=@<project>/${SERVICE}`), what the ingress routes to, and
  what a cross-workspace dependency names (`"@<project>/postgres": "*"`).
- **Nothing lives at the repository root but** the root manifest, the lockfile,
  the compose file and the one Dockerfile. Sources are always a workspace down.

## The data API is GraphQL, and the service is called `graphql`

Not a naming preference. A service called `api` says nothing except that it is
not the other ones. A service called `graphql` tells you which protocol you are
about to speak before you open a file — and the name is only honest if the
service really does speak it, which is the other half of the rule.

So the two travel together: **if it is the data API it is GraphQL, and if it is
GraphQL it is called `graphql`.** A service called `graphql` that does not speak
it is worse than either, because the name is now actively lying.

It owns the `/graphql` prefix, and everything the product does with data goes
through it: one endpoint, a schema, resolvers.

## `auth` is a service, and it is not GraphQL

Sign-in is redirects, cookies and status codes. A magic link arrives as a
top-level `GET` from a mail client and has to answer with a `302` and a
`Set-Cookie`; a browser has to be handed a `401` it can recognise. GraphQL has
no good answer for any of that — it would wrap the whole flow in `200 OK` and
put the real outcome in a field.

So `auth` keeps its own prefix and its own plain HTTP, and the data API stays
out of it. Two protocols, each doing what it is good at, is not an
inconsistency to resolve.

## One service per path prefix

The ingress routes on the prefix and nothing else, and that is the whole
service boundary:

```
/auth/*      auth
/graphql*    graphql
everything   the client
```

- **A prefix is owned by exactly one service.** If two want it, they are one
  service.
- **The last rule has no path.** Whatever is left goes to the client, which
  answers every one of them with the same shell — that is what lets a
  client-side route survive a reload.
- The Caddyfile lives inline in the compose file, so the routing table and the
  services it points at are one file and cannot drift apart.

## Services and libraries

- A **service** has a `build` script, gets an image, and appears in the compose
  file.
- A **library** has a `main` and no build. It never ships on its own; it is
  bundled into whatever depends on it. Cross-workspace dependencies are
  declared as `"@<project>/postgres": "*"`.
- A library exists when a second service needs the same thing, not in
  anticipation of one. An `index.js` arrives on the branch that first imports
  it — an empty package is a file nobody wrote.
- **The health check is an express handler living in a service's workspace** —
  the plain-HTTP service (`auth`), never the GraphQL one and never a shared
  library. It is waited on before anything else is up and must answer without a
  query document, so it stays plain HTTP; and it is that service's own route,
  not something factored out, because there is only ever one of it and a
  library exists only once a second service needs the same thing.

## One image, many services

One Dockerfile at the repository root, and which service it builds is the
`SERVICE` build argument. It is the same file in every repo:

```dockerfile
from node:24.13-alpine as builder

arg SERVICE
env SERVICE=${SERVICE}

copy package.json /source/package.json
copy package-lock.json /source/package-lock.json

copy --exclude=**/*.js --exclude=**/*.graphql workspaces /source/workspaces
run cd /source && npm install

copy workspaces /source/workspaces
run cd /source && npm run build --workspace=@<project>/${SERVICE}

expose 80
cmd cd /source && npm run develop --workspace=@<project>/${SERVICE}

from node:24.13-alpine

arg SERVICE
env SERVICE=${SERVICE}

copy --from=builder /source/workspaces/@<project>/${SERVICE}/dist /app

expose 80
cmd cd /app && node index.js
```

- The **`builder` stage installs once and builds the named workspace.** The
  first `copy` brings in only the manifests — `--exclude` drops every `.js` and
  `.graphql` source — so the `npm install` layer is keyed on the `package.json`
  files alone and a source edit does not reinstall. The full `copy workspaces`
  then brings the sources in, and `npm run build` builds the one workspace.
- The **development override stops at `builder`**, bind-mounts the sources, and
  runs that workspace's `develop` — the stage's own `cmd` — so a saved file
  rebuilds and restarts in place.
- The **second stage copies only `dist/`** from the built workspace — no
  `node_modules`, no sources, no lockfile — which is why every dependency is a
  `devDependency`. It runs `node index.js`, so a service's built entry is
  always `dist/index.js`.
- A service that needs none of the above is not a service. The reverse is also
  true: a folder of files served by the ingress does not need a container, but
  a shell that has to be rendered does.

### There is no `.dockerignore`, and that is on purpose

`.dockerignore` does **not** read `.gitignore` — Docker has never honoured it,
the two files are independent, and one has never covered for the other. So the
reason these repos carry no `.dockerignore` is not that `.gitignore` stands in
for it. It is that the Dockerfile never copies anything that should not ship: it
names the manifests and the `workspaces` tree explicitly, `node_modules` is
hoisted to the repository root and is never a `copy` target, and the second
stage takes only `dist/`. Nothing to ignore means no file to keep in sync.

## The web app is the client, and it renders a shell

The path-less ingress rule sends everything left to the client, and the client
is a `web` workspace. It is a service and not a folder of static files for one
reason: it renders a shell per request rather than serving a fixed one, and
that is what lets any client-side route survive a reload. A page that only ever
shipped the same bytes would not need a container — this one runs `express`.

Build it as **two entry points from one workspace**, because the server and the
browser are two runtimes and each needs its own bundle:

- `index.js` is the **server**. It is an `express` app that renders the React
  shell with `renderToPipeableStream`, serves `/assets` statically, names
  `/assets/client.js` as the bootstrap script, and listens on the fixed port
  the ingress points at. `build:server` bundles it with esbuild for
  `--platform=node`; the runtime stage copies only its `dist/`.
- `client.js` is the **browser**. It does one thing — `createRoot` on `#root`
  and render — because everything else is a component. `build:client` bundles
  it with esbuild for the browser into `dist/assets`, so it lands where the
  server serves it.
- `build` runs both, minified. `develop` runs both under `concurrently` in
  `--watch`, which is the `develop` the Dockerfile's development override runs
  in place. Everything is a `devDependency` — the same rule as every service,
  for the same reason: the runtime keeps only `dist/`.

The shell and the app are **two different React trees**, and keeping them apart
is the point:

- `<Page/>` is server-only. It is the `<html>` document — meta, the critical
  CSS inlined into a `<style>`, the `#root` div, the bootstrap script. It never
  hydrates, so nothing in it may depend on browser state.
- `<App/>` is what hydrates. It wraps the providers around the router
  (`<AuthProvider><Router/>`), and the router is a table keyed on auth state —
  a splash while it loads, sign-in when signed out, the protected routes once
  in — so an unauthenticated reload of a deep link resolves without the server
  knowing the route.

The workspace is laid out so a file's directory says what kind of thing it is:

```
workspaces/@<project>/web/
  index.js            server entry — renders <Page/>
  client.js           browser entry — hydrates <App/>
  components/         one directory per component, each a self-named index.js
  contexts/           providers — auth, stream
  hooks/              reusable browser hooks
  package.json        every dependency a devDependency
```

- **One component per directory**, named for itself, holding an `index.js`. A
  component is a directory and not a loose file for the same reason a skill is:
  anything it needs — its styles, a child used nowhere else — sits beside it,
  and a sibling at the root would be associated with nothing.
- A **context** is the shared state a provider owns; a **hook** is browser
  behaviour reused across components. Data still comes from `graphql` and
  sign-in still goes through `auth` — the web app talks to both across the
  ingress, it does not reach around them.

## Configuration

- A service is handed **only the variables it reads**. The `environment:` block
  is the list, and a service missing a variable is how you know what it cannot
  do — one with no mail URL cannot send mail, whatever its code says.
- Secrets split by capability, not by convenience: whoever signs holds the
  private key and nothing else does; whoever verifies holds the public one.
- Services listen on a fixed port inside the network and are never published.
  Only the ingress is.
