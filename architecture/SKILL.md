---
name: architecture
description: How a project is shaped in these repositories — a workspace per package, a service per path prefix, an ingress that routes by prefix, and one image built for each service. Covers what a service is called, and in particular that the data API is GraphQL and is named for it. Load before adding a service or a workspace, naming either, deciding what belongs in a library, changing the ingress, or touching the Dockerfile.
---

# Architecture

Read out of three repositories that share the shape. Where a repository and
this file disagree, the repository wins and this file needs a change.

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
  declared as `"@scope/postgres": "*"`.
- A library exists when a second service needs the same thing, not in
  anticipation of one. An `index.js` arrives on the branch that first imports
  it — an empty package is a file nobody wrote.
- **Health belongs on a plain HTTP service**, not the GraphQL one: it is waited
  on before anything else is up, and it should not require a query document to
  answer.

## One image, many services

One Dockerfile, and which service it builds is a build argument.

- A `builder` stage installs once and builds the named workspace. The
  development override stops there, bind-mounts the sources, and runs that
  workspace's `develop`, so a saved file rebuilds and restarts in place.
- A `runtime` stage copies **only** `dist/`. No `node_modules`, no sources, no
  lockfile — which is why every dependency is a `devDependency`.
- A service that needs none of the above is not a service. The reverse is also
  true: a folder of files served by the ingress does not need a container, but
  a shell that has to be rendered does.

## Configuration

- A service is handed **only the variables it reads**. The `environment:` block
  is the list, and a service missing a variable is how you know what it cannot
  do — one with no mail URL cannot send mail, whatever its code says.
- Secrets split by capability, not by convenience: whoever signs holds the
  private key and nothing else does; whoever verifies holds the public one.
- Services listen on a fixed port inside the network and are never published.
  Only the ingress is.
