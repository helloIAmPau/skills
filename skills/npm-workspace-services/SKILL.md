---
name: npm-workspace-services
description: Plan or maintain npm workspace web applications with source-only libraries, compiled Express and GraphQL application services, a shared server contract, one Dockerfile, and Docker Compose with inline Caddy configuration. Use for repositories adopting this structure; preserve other architectures unless migration is requested.
---

# npm workspace services

Use this convention for repositories adopting an npm monorepo with containerized Express services. Derive the package scope from the project; Hybrid uses `@hybrid`. Examples describe structure, not authorization to scaffold applications or choose service boundaries.

## Repository layout

```text
repository/
├── package.json
├── package-lock.json
├── Dockerfile
├── docker-compose.yml
├── docker-compose.develop.yml
├── .env                  # local deployment settings, ignored by Git
├── .env.example          # tracked deployment template
├── .env.develop          # tracked, non-secret development settings
└── workspaces/
    └── @scope/
        └── <module>/
            ├── package.json
            └── index.js
```

- Use a root `package.json`, singular, with `"private": true` and `"workspaces": ["workspaces/@scope/*"]`. Replace `scope` with the actual project namespace.
- Put each application, service, or shared package under `workspaces/@scope/<module>`, with its own manifest named `@scope/<module>`.
- Use npm and one root `package-lock.json`. Install from the root; use `npm ci` for reproducible CI and container builds and workspace-targeted scripts for individual modules.
- Put the source entry `index.js` directly at each JavaScript workspace root; do not introduce a `src` subfolder.
- For backend workspaces, declare every dependency in the using package's `devDependencies`, including workspace imports, Express, and build tools. Mobile workspaces are the owner-approved exception and may use runtime `dependencies` under the React Native/Expo skill. This convention belongs to the agreed bundled-application architecture; keep development dependencies available during installation and builds. Do not move Express into `dependencies` merely because the running application uses it.
- Keep client implementation conventions in separate skills. This skill defines repository layout and backend infrastructure.

## Source libraries and compiled applications

- A library is imported by an application; it is not itself a running service. Libraries declare `"type": "module"` and `"exports": "./index.js"`, exporting their source directly. Importing a library alone must not open a listener.
- Libraries have no build script, compiler dependency, generated `dist/`, or Compose service. Do not add a prerequisite library compilation step to a consuming application's build or Dockerfile.
- Compile only exposed application service workspaces. Their esbuild command bundles the root `index.js`, including imported workspace library source and dependencies, into self-contained CommonJS `dist/index.js` with source maps. For the current toolchain, use `--bundle --platform=node --target=node24 --format=cjs --outfile=dist/index.js --sourcemap`.
- Application workspaces own esbuild and their development process tools. Omit `type: module` in application packages producing CommonJS `dist/index.js`, so that output runs both inside its workspace and in the runtime image. Their source can still use ESM because esbuild reads it.
- Root `npm run build` may use `npm run build --workspaces --if-present`: only application services have build scripts. A repository containing only libraries has nothing to compile; that is a valid state.
- A library such as `@scope/service` owns its direct imports, including Express, in its own `devDependencies`. Consumers declare the library package in their own `devDependencies` and import it by package name.
- Do not create a demonstration application or container just to exercise the library. When no application has been selected, keep a Caddy-only stack and a development override containing `services: {}`. Hybrid's example service was explicitly removed; do not recreate it as part of ordinary maintenance.

## Shared Express server contract

Use `@scope/service` for the shared server library; Hybrid uses `@hybrid/service`. Its exported declaration is `export const service = function(name, handler)`. A consuming application's root entry calls it:

```js
import { service } from '@scope/service';

service('service_name', function({ router }) {
  router.get('/', function(request, response) {
    return response.data({ key: 'value' });
  });
});
```

- Each invocation creates a fresh Express application and router, mounts the router at `/${ name }`, and invokes `handler({ router })` once, synchronously, before listening. Application route paths are relative to that prefix. There is no automatic `/api` prefix.
- Attach `response.data(data)` and `response.error(error)` to the response object before routing. Never attach these helpers to `request`.
- `response.data` sends HTTP 200 and `{ "data": { "key": "value" } }`. `response.error` sends `{ "errors": [{ "message": "error message" }] }`. These are the common success and error envelopes.
- Only `response.error` selects the error status. Preserve an integer `error.status` from 400 through 599; otherwise use 500. Serialize `error.message` in the error payload.
- Register the library's `GET /health` route before the application callback. It returns `{ "data": { "status": "ok" } }` at `/<service_name>/health`. Applications do not duplicate this route. It indicates HTTP availability, not dependency readiness.
- After application routes, register a fallback that forwards a 404 `Error`, followed by the four-argument generic middleware `function(error, request, response, next) { return response.error(error); }`. Keep all four parameters because Express uses the signature to recognise error middleware. Use the name `error` and omit a `headersSent` check, as explicitly selected for this contract.
- Application handlers may call `response.error(error)` directly or forward with `next(error)`. Express 5 forwards synchronous exceptions and rejected promises returned by handlers; application promise chains must be returned.
- Return `app.listen(80, '0.0.0.0')` directly. Do not add a `server.once` hook, shutdown procedure, or process signal handlers to the shared library under this agreement.
- Additional middleware, request-body parsing, authentication, persistence, and product API contracts are separate decisions. The server abstraction does not establish a domain service boundary.

## GraphQL schema and resolvers

For Hybrid's GraphQL service, follow the owner-selected conventions recorded in [issue #3](https://github.com/helloIAmPau/hybrid/issues/3):

- Create the GraphQL application with `@hybrid/service`, using `service('graphql', function({ router }) { ... })`. Register the GraphQL handler on the router's `/` path; the shared library supplies `/graphql/health` and the listener.
- Load SDL from `workspaces/@hybrid/graphql/schema.graphql`, beside the application's `index.js`. Keep schema definitions out of JavaScript. Use GraphQL Tools' `makeExecutableSchema` from `@graphql-tools/schema` to construct the executable schema and `createHandler` from `graphql-http/lib/use/express` to expose it.
- Put resolver implementations under `workspaces/@hybrid/graphql/resolvers/`. Group them by subject: all user-related resolvers belong in `user.js`, with equivalent modules for other subjects. Do not split the same subject across separate query and mutation modules.
- Assemble the subject modules in `resolvers/index.js` and export the combined root object as `rootValue`. Import it into the application's root `index.js` and pass it with `createHandler({ schema, rootValue })`. Keep this name consistent; do not rename the exported object to `root`.
- Subject modules export root resolver functions; keep their exported field names unique when combining them. The index composes source modules; resolver code is bundled with the application rather than compiled as a separate library.
- Subject names illustrate organization, not authorization to invent API fields or storage behavior. Add subject modules when their fields are defined.

Illustrative aggregation once user-related fields exist:

```js
// workspaces/@hybrid/graphql/resolvers/index.js
import * as user from './user.js';

export const rootValue = {
  ...user
};
```

The owner approved issue #3's implementation plan. Load the workspace's SDL with esbuild's `.graphql=text` loader, bundling its text into the application. Construct the executable schema before opening the listener; the owner requested removal of the explicit `assertValidSchema` call, so do not add a separate schema-validation step. The Docker builder's workspace copy and the development workspace bind mount already include the schema; do not add a separate schema copy or mount. Schema edits rebuild/restart the development application; production updates require rebuilding the image. Verify the bundle outside the checkout so schema loading cannot depend on its source path.

The owner confirmed that standard GraphQL responses are compatible with the project's format. Follow issue #3's `createHandler({ schema, rootValue })` example and pass its responses through unchanged. Health, unmatched routes, and generic Express errors retain the shared response contract.

## One Dockerfile for backend services

- Keep one reusable Dockerfile at the repository root. Every Express service build uses that Dockerfile and the repository root as its build context.
- Hardcode the exact latest LTS Node image version in every Node `FROM` instruction. Verify the current LTS release when creating or updating the pin; do not use floating tags or an image build argument/environment variable.
- Do not declare `EXPOSE`, `VOLUME`, or Dockerfile health checks for application services. Every Express service must hardcode its listener to port `80` on `0.0.0.0`. Do not configure service ports through `PORT`, `SERVICE_PORT`, Dockerfile environment variables, or Compose. Backend listeners are reached through Caddy on the container network.
- Select an exposed application service with a `SERVICE` build argument containing its module name. Build only the matching `@scope/${SERVICE}` workspace; its esbuild bundle includes imported source libraries. Never select a library as a container entry point.
- Use a multi-stage build. Install from the root lockfile in the builder, then copy only runtime requirements into the final image.
- Standardize application service build scripts and output locations. Emit `dist/index.js` and run `node index.js` from the copied output directory. Source libraries retain their uncompiled root `index.js`.
- Copying only `dist/` works only when it is self-contained. Otherwise include production dependencies, required workspace packages, native modules, and runtime assets. TypeScript compilation alone does not bundle dependencies.
- Exclude local dependencies, build outputs, Git metadata, and local environment files from the build context as appropriate. Do not bake `.env` values into images.
- Keep a runnable `builder` stage with build tools, persist the `SERVICE` argument as `ENV SERVICE=${SERVICE}`, and run the selected workspace's `develop` script when that stage is started. The final stage runs the built entry point directly, independently of development tooling.

When a real application service is added, select its module in Compose. This illustrates a placeholder name, not a request to scaffold it:

```yaml
services:
  service_name:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        SERVICE: service_name
```

## Environment and lifecycle commands

- Environment files are shell-sourceable assignments using `export NAME=value`. Quote values according to shell syntax when needed. Keep local deployment values in ignored `.env` and a safe template in `.env.example`; keep non-secret local development settings in tracked `.env.develop`.
- Root `npm start` sources `.env` and runs the base `docker-compose.yml` stack with `up --build --detach`.
- Root `npm run develop` sources `.env.develop` and runs `docker compose -f docker-compose.yml -f docker-compose.develop.yml up --build --abort-on-container-exit`. The development override includes Android tooling and Metro under [react-native-expo](../react-native-expo/SKILL.md). Its separately maintained `ghcr.io/helloiampau/mobile-tools` image owns the generic `mobile-develop` command and uses Debian for Android glibc compatibility. Consume a published digest and select `MOBILE_WORKSPACE=workspaces/@hybrid/mobile`; keep both its Dockerfile and lifecycle script in the tooling repository. Follow the native skill for local candidate verification, mounts and publication. It is not an Express service. Mobile tooling also mounts root manifests and tests as explicit development source exceptions. Use POSIX `. ./file` in npm scripts so sourcing works with npm's default `/bin/sh`.
- The development override selects `build.target: builder` for application services and bind-mounts `./workspaces:/source/workspaces` for live editing. This source mount is an explicit exception to the persistent-data path rule; keep all runtime data under `./data/<service-name>/`. Do not introduce named or anonymous dependency volumes.
- Each application service's `develop` script first builds, then runs its esbuild watcher and `node --watch dist/index.js` concurrently. The esbuild watcher follows imports into source libraries; do not add separate library builds or watchers. Stop the companion watcher when either process exits. An application `start` script runs its built entry point.
- Ensure the output module format works both in the workspace during development and in `/app` at runtime. For CommonJS `dist/index.js`, avoid inheriting a `type: module` package scope.
- Both environments retain the same host-variable convention, hardcoded service port 80, frontend/backend networks, fixed Caddy TCP ports 80/443, and prohibition on exposed backend ports and health checks.

## Compose and Caddy

- Always name the root Compose file `docker-compose.yml`. Use that filename in scripts, documentation, and validation commands.
- Run each backend application service as a separate Express server and Compose service. Infrastructure containers such as Caddy and databases use their own suitable images.
- Shared libraries have no Compose entries, `depends_on` entries, or proxy targets. When removing an application, also remove its workspace dependency, lockfile entries, Compose service, development override, and Caddy routes; remove now-unused build tools with that workspace.
- Hardcode Caddy's image with an exact release version in `docker-compose.yml`. Caddy does not maintain a separate LTS line; verify and pin the latest supported stable release. Do not use floating tags or an image environment variable.
- Caddy is the only service allowed to publish ports. Publish only TCP ports `80:80` and `443:443`. Do not add a UDP port mapping. Store the full Caddy site address, including `http://` or `https://`, in `.env` as `<PROJECT>_HOST`, replacing `<PROJECT>` with the actual project name in uppercase (for example, `HYBRID_HOST` for Hybrid). Use that concrete variable name consistently in environment files, Compose, and documentation, and interpolate it directly into the Caddyfile. Never prepend a scheme or append a port in Compose. Keep the published ports fixed at 80 and 443; the address scheme controls which protocol is served. An explicit `http://` address serves HTTP without automatic HTTPS; `https://` enables HTTPS with HTTP redirects. For local development use `http://localhost`, or `https://localhost` with Caddy's local CA.
- Backend services must have neither `ports` nor `expose` entries. External clients reach them only through Caddy. Name the networks `backend` and `frontend`. Connect Caddy and HTTP services it proxies to `frontend`. Connect dependencies such as databases to `backend`, and attach application services to `backend` only when they need those dependencies. Caddy does not need `backend`. Do not set `internal: true` on either network. Address HTTP upstreams as `http://<service-name>`, using the hardcoded port 80 without a port suffix.
- Never use Docker-managed named or anonymous volumes or a top-level `volumes` declaration. Use host bind mounts for every service's persistent storage, with all persistent-data sources under `./data/<service-name>/`. The development source bind mount described above is separate from persistent storage. For example, bind `./data/caddy/data` to `/data` and `./data/caddy/config` to `/config`. Keep `data/` ignored by Git and excluded from image build contexts. Stateless services need no storage mount.
- Define the Caddyfile inline under top-level `configs.caddyfile.content` in `docker-compose.yml`. Grant the Caddy service the `caddyfile` config with target `/etc/caddy/Caddyfile`. The Compose key is `configs`, plural; inline content requires Compose 2.23.1 or newer.
- Use `.env` for deployment-specific settings and Compose interpolation in the inline config. Use only plain `${VARIABLE}` references in Compose. Do not add defaults, fallbacks, required-value checks, error messages, or any other interpolation operators. Document explicit deployment values in `.env.example`; ignore `.env` and private environment variants. Track only non-secret `.env.develop` settings.
- Compose `.env` interpolation does not automatically populate container environments. Pass runtime variables explicitly through `environment` or an appropriate `env_file`. If using Caddy's own environment substitution, account for Compose dollar escaping and supply the variables to Caddy.
- Omit health checks for now, including Compose `healthcheck` blocks and `depends_on` conditions requiring `service_healthy`. A simple startup dependency is allowed but does not guarantee readiness. Existing diagnostic endpoints may remain.
- Structure the Caddyfile with one site block from the full environment address, direct path-matched `reverse_proxy` directives, and explicit HTTP upstreams identified by Compose service name. All Express upstreams use port 80, so omit the port suffix.
- Preserve incoming path prefixes; Express services own their full public routes. Do not use `handle_path` or strip prefixes unless specifically requested. A `/prefix/*` matcher excludes the exact `/prefix` path; add a separate rule when both are required.
- When a frontend web service exists, use an unqualified `reverse_proxy http://web` as the catch-all. Otherwise keep unknown paths at 404 with an explicit unmatched-path matcher; update it when adding routes. Do not add a web service solely to fill this slot. Avoid a bare `respond 404` beside proxy rules, since Caddy's directive ordering would make it intercept matched requests.
- Caddy's own unmatched-path and proxy-generated errors use the shared `{ "errors": [{ "message": "..." }] }` envelope and `Content-Type: application/json`. Trigger unmatched-path errors with `error @unmatched "Not found" 404`, then use `handle_errors` with `respond` and `{err.status_code}`. Standard `{err.status_text}` can be placed in the JSON message; raw error text needs JSON escaping and must not be interpolated unescaped. Upstream application error responses already use the shared envelope and pass through unchanged.
- In a Caddy-only stack, `@unmatched path *` covers every request. There is no public application health endpoint until an application is present and routed. Narrow this matcher when adding proxy paths, including both the exact service prefix and its descendants where needed.
- When replicas are requested, configure upstream discovery or explicit upstreams and balancing deliberately. A single-backend route does not establish verified replica load balancing.

## Apply and verify

Read existing manifests and repository instructions before edits. Preserve established service boundaries and public contracts. Leave unchosen database, authentication, API, and language decisions open. Keep planning distinct from implementation and honor the user's commit instructions.

For actual changes, check workspace discovery, direct library source imports, affected application build scripts, Compose configuration, and relevant startup and routing as available. Verify application bundles outside their package scope when an application exists; do not build a source-only library merely to perform that check. A native Caddy check can supplement validation when Docker is absent, but does not verify container builds or Docker network routing. Avoid printing resolved configuration containing secrets. Report checks that could not run. Apply the workspace ownership convention at the end of each filesystem-changing operation, excluding generated outputs where allowed.

Respect project decisions recorded in `AGENTS.md`. Hybrid uses E2E tests only, as authorized by the owner in issue #3. Before every test run, start the entire development stack with `npm run develop`, or confirm that every service is already ready and current changes have rebuilt. Run the complete suite with root `npm test`, which executes Node's built-in test runner inside the running mobile-tools container. HTTP E2E requests go through Caddy; native E2E uses Maestro under the mobile skill. Tests do not manage stack startup or shutdown. Keep all tests and fixtures directly in `tests/e2e/`; do not add feature-specific test commands. Do not add unit tests, in-process integration tests, mocked transports, a fixture workspace, or a replacement example service.
