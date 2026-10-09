---
name: npm-workspace-services
description: Plan or maintain npm workspace web applications with source-only libraries, compiled Express and GraphQL application services, a shared server contract, one Dockerfile, and Docker Compose with inline Caddy configuration. Use for repositories adopting this structure; preserve other architectures unless migration is requested.
---

# npm workspace services

Use this convention for repositories adopting an npm monorepo with containerized Express services. Derive the package scope, paths and selected services from the consuming configuration. Examples describe structure, not authorization to scaffold applications or choose service boundaries.

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
- For backend workspaces, declare every dependency in the using package's `devDependencies`, including workspace imports, Express, and build tools. When present, mobile workspaces are the exception and may use runtime `dependencies` under the React Native/Expo skill. This convention belongs to the agreed bundled-application architecture; keep development dependencies available during installation and builds. Do not move Express into `dependencies` merely because the running application uses it.
- Keep client implementation conventions in separate skills. This skill defines repository layout and backend infrastructure.

## Source libraries and compiled applications

- A library is imported by an application; it is not itself a running service. Libraries declare `"type": "module"` and `"exports": "./index.js"`, exporting their source directly. Importing a library alone must not open a listener.
- Libraries have no build script, compiler dependency, generated `dist/`, or Compose service. Do not add a prerequisite library compilation step to a consuming application's build or Dockerfile.
- Compile only exposed application service workspaces. Their esbuild command bundles the root `index.js`, including imported workspace library source and dependencies, into self-contained CommonJS `dist/index.js` with source maps. For a selected Node 24 toolchain, use `--bundle --platform=node --target=node24 --format=cjs --outfile=dist/index.js --sourcemap`.
- Application workspaces own esbuild and their development process tools. Omit `type: module` in application packages producing CommonJS `dist/index.js`, so that output runs both inside its workspace and in the runtime image. Their source can still use ESM because esbuild reads it.
- Root `npm run build` may use `npm run build --workspaces --if-present`: only application services have build scripts. A repository containing only libraries has nothing to compile; that is a valid state.
- A library such as `@scope/service` owns its direct imports, including Express, in its own `devDependencies`. Consumers declare the library package in their own `devDependencies` and import it by package name.
- Do not create a demonstration application or container just to exercise the library. When no application has been selected, keep a Caddy-only stack and a development override containing `services: {}`.

## Shared Express server contract

Use the configured shared server library, illustrated as `@scope/service`. Its exported declaration is `export const service = function(name, handler)`. A consuming application's root entry calls it:

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

For selected GraphQL data services, derive service boundaries and route names from the consuming project's instructions. HTTP health remains a separate route. Use these schema and resolver conventions:

- Create the GraphQL application with `@scope/service` and the project's selected service name. Register `createHandler({ schema, rootValue })` with `router.all('/')`; the shared library supplies the service's `/health` and listener.
- Load SDL from `schema.graphql`, beside the application's `index.js`. Keep schema definitions out of JavaScript. Use GraphQL Tools' `makeExecutableSchema` from `@graphql-tools/schema` to construct the executable schema and `createHandler` from `graphql-http/lib/use/express` to expose it.
- Put resolver implementations under the application workspace's `resolvers/`. Group them by subject: all user-related resolvers belong in `user.js`, with equivalent modules for other subjects. Do not split the same subject across separate query and mutation modules.
- Assemble the subject modules in `resolvers/index.js` and export the combined root object as `rootValue`. Import it into the application's root `index.js` and pass it with `createHandler({ schema, rootValue })`. Keep this name consistent; do not rename the exported object to `root`.
- Subject modules export root resolver functions; keep their exported field names unique when combining them. The index composes source modules; resolver code is bundled with the application rather than compiled as a separate library.
- Subject names illustrate organization, not authorization to invent API fields or storage behavior. Add subject modules when their fields are defined.

Illustrative aggregation once user-related fields exist:

```js
// workspaces/@scope/data-service/resolvers/index.js
import * as user from './user';

export const rootValue = {
  ...user
};
```

Load the workspace's SDL with esbuild's `.graphql=text` loader, bundling its text into the application. Construct the executable schema before opening the listener; do not add a separate `assertValidSchema` step. The Docker builder's workspace copy and the development workspace bind mount already include the schema; do not add a separate schema copy or mount. Schema edits rebuild/restart the development application; production updates require rebuilding the image. Verify the bundle outside the checkout so schema loading cannot depend on its source path.

Pass `createHandler({ schema, rootValue })` responses through unchanged. Browser clients check GraphQL `errors` even when HTTP status is 200. Health, unmatched routes, and generic Express errors retain the shared response contract.

## One Dockerfile for backend services

- Keep one reusable Dockerfile at the repository root. Every Express service build uses that Dockerfile and the repository root as its build context.
- Hardcode the exact latest LTS Node image version in every Node `FROM` instruction. Verify the current LTS release when creating or updating the pin; do not use floating tags or an image build argument/environment variable.
- Do not declare `EXPOSE`, `VOLUME`, or Dockerfile health checks for application services. Every Express service must hardcode its listener to port `80` on `0.0.0.0`. Do not configure service ports through `PORT`, `SERVICE_PORT`, Dockerfile environment variables, or Compose. Backend listeners are reached through Caddy on the container network.
- Select an exposed application service with a `SERVICE` build argument containing its module name. Build only the matching `@scope/${SERVICE}` workspace; its esbuild bundle includes imported source libraries. Never select a library as a container entry point.
- Use a multi-stage build. Install from the root lockfile in the builder, then copy only runtime requirements into the final image.
- Configure esbuild entirely through CLI arguments in npm scripts; do not add `build.js`, `build/index.js` or other JavaScript build helpers. Watch scripts reuse build commands with watch flags.
- Standardize application service build scripts and output locations. Emit `dist/index.js` and run `node index.js` from the copied output directory. Source libraries retain their uncompiled root `index.js`.
- Copying only `dist/` works only when it is self-contained. Otherwise include production dependencies, required workspace packages, native modules, and runtime assets. TypeScript compilation alone does not bundle dependencies.
- Exclude local dependencies, build outputs, Git metadata, and local environment files from the build context as appropriate. Do not bake `.env` values into images.
- Keep a runnable `builder` stage with build tools, persist the `SERVICE` argument as `ENV SERVICE=${SERVICE}`, and use container initialization to install after mounts before invoking the selected workspace's `develop` script. The final stage runs the built entry point directly, independently of development tooling.

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

- Environment files use shell-sourceable assignments, `export NAME=value`, with
  shell quoting where needed. Keep deployment values in ignored `.env`, a safe
  `.env.example`, and non-secret development settings in tracked `.env.develop`.
  Use POSIX `. ./file` in npm scripts for npm's default `/bin/sh`.
- Root `npm start` sources deployment settings and starts the base
  `docker-compose.yml` with `up --build --detach`.
- Root `npm run develop` sources development settings and runs
  `docker compose -f docker-compose.yml -f docker-compose.develop.yml up --build --abort-on-container-exit`.
  Run this long-lived command in a managed terminal/process while checking
  readiness and running tests separately. Foreground Compose returns when the
  stack stops: do not put installation after it with `docker compose up && npm ci`.
- The development override selects `build.target: builder` for backend services
  and retains `./workspaces:/source/workspaces` for live editing. Source mounts
  are separate from persistent data under `./data/<service-name>/`. Do not add
  named or anonymous dependency volumes.
- Include emulator/Metro tooling only when a native workspace has been selected;
  follow [react-native-expo](../react-native-expo/SKILL.md) for its distinct lifecycle.
- Backend `develop` scripts build initially, then run esbuild watch and
  `node --watch dist/index.js` concurrently. Start them only after the install
  barrier below. Stop the companion process if either exits. Source libraries
  need no separate builds/watchers. Application `start` runs compiled output.
- CommonJS output must run both inside its development workspace and outside
  the checkout's package scope in the runtime image.

## Post-mount installation barrier

This is the shared development initialization contract for service containers
and selected native tooling:

1. Start containers so the development bind mounts are active.
2. In each environment that builds/runs an application, install from its effective
   workspace root with `npm ci` using the existing valid lockfile.
3. Wait for successful installation before advancing. Failure stops startup:
   no initial build, native preparation, application watchers, Metro or readiness.
4. Perform the initial build or native preparation requiring dependencies.
5. Start application watchers, Metro and other long-running development processes.
6. Declare readiness only after the relevant application/service/device is ready.

The effective root must see the current root manifests/lockfile and all selected
workspace manifests after mounts. Refresh image-copied root inputs when they change;
never install against stale image manifests while consuming updated mounted sources.

Implement the barrier in container/startup initialization. For a private backend
root, a sequential entry command can be `npm ci && npm run develop --workspace=@scope/service_name`
from `/source`; the selected `develop` script must build before launching watchers.
Replace the example package with the configured service. When roots share writable
dependency paths, use a serialized installation coordinator and explicit successful
completion barrier before starting any consumer; do not run competing `npm ci`
commands on those paths. An image-build install may be needed for the image but
does not replace this post-mount installation.

An install in one container's private root does not provision another container.
Account for dependencies under both root and workspace-local `node_modules`:
workspace directories mounted writable can put installed artifacts in the host
checkout. Do not claim all container-installed dependencies are host-isolated.
Verify that workspaces requiring incompatible versions each resolve their locked
version through the selected application resolver, including nested installations.

Before sharing writable dependency trees, check platform and native module ABI
compatibility. Runtimes with incompatible artifacts need compatible, separately
provisioned dependency paths while retaining the source bind mounts. Never let a
host tool or another runtime consume an incompatible tree. Keep generated
dependency directories ignored by Git.

On root/workspace manifest or lockfile changes, stop affected application processes,
serialize installation, wait for success, then rebuild/restart. Watchers must not
observe a partially rewritten tree. Use `npm install` only for intentional
dependency/lockfile changes, never as a silent fallback when locked installation fails.

Host test tools require a separate host installation from the effective checkout
root with `npm ci`, plus any host-native prerequisites. Stop affected processes
and coordinate with container installs if paths overlap. Container tooling does
not satisfy host executable prerequisites.

## Compose and Caddy

- Always name the root Compose file `docker-compose.yml`. Use that filename in scripts, documentation, and validation commands.
- Run each backend application service as a separate Express server and Compose service. Infrastructure containers such as Caddy and databases use their own suitable images.
- When a dependency is hosted externally in production, put its local development
  container and local startup dependencies only in `docker-compose.develop.yml`.
  Keep production connection settings and server-side credentials explicit;
  do not require a development-only container in the base Compose file.
- Shared libraries have no Compose entries, `depends_on` entries, or proxy targets. When removing an application, also remove its workspace dependency, lockfile entries, Compose service, development override, and Caddy routes; remove now-unused build tools with that workspace.
- Hardcode Caddy's image with an exact release version in `docker-compose.yml`. Caddy does not maintain a separate LTS line; verify and pin the latest supported stable release. Do not use floating tags or an image environment variable.
- Caddy is the sole entry point for HTTP application traffic. Publish its TCP
  ports 80/443; do not add UDP publication. Store its full site address including
  scheme in the configured environment variable (for example `PUBLIC_URL`).
  Interpolate that address directly in the Caddyfile; do not prepend a scheme or
  append a port there. Explicit `http://` disables automatic HTTPS; `https://`
  enables it. Local access may use localhost with an appropriate trust setup.
- Development configurations may publish ADB and other documented service
  protocols required by host tests, including database infrastructure tests.
  Publish Metro where a host driver needs it. Expose only required endpoints,
  bind to host loopback for local-only access, and keep development-only mappings
  out of production configuration. See the host-test contract below.
- Backend HTTP application services must have neither `ports` nor `expose` entries. External clients reach them only through Caddy. Name the networks `backend` and `frontend`. Connect Caddy and HTTP services it proxies to `frontend`. Connect dependencies such as databases to `backend`, and attach application services to `backend` only when they need those dependencies. Caddy does not need `backend`. Do not set `internal: true` on either network. Address HTTP upstreams as `http://<service-name>`, using the hardcoded port 80 without a port suffix.
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

## Host testing and public interfaces

Root `npm test` and every test-driver process run on the host: Node tests, Maestro,
recording tools and migration verification tools. Containers run the services,
emulator, app and development servers under test. Host Docker/Compose commands
may control them, but must not execute the test suite inside a service container.

| Target | Host access |
| --- | --- |
| HTTP application | Configured host-reachable public URL through Caddy |
| Native app/emulator | Published ADB server, used by host-native tools |
| Metro/development tooling | Documented host endpoint where needed, with a working device-side connection |
| Other services, including database infrastructure | Documented public protocol published to the host when required |

HTTP application tests must use the gateway; never publish individual backend
HTTP ports for a bypass. Let the HTTP client derive ordinary Host/authority from
the request URL; do not supply a manual Host header or use an internal gateway
name as the host test destination. Configure routing for the actual host/device
public URLs. Do not use container IP discovery or Docker-only DNS destinations
for host tests. Host and device `localhost` are different addressing contexts.

Application behavior assertions use the public API/UI, without in-process
application imports, internal implementation access or mocked transports. Direct
database access is appropriate for database/migration interface tests or explicitly
defined fixture administration, never as a replacement for application assertions.
No test-suite bind mount or test-only database credentials in native tooling are
needed merely to run the host suite. Missing host tools, devices, ports or services
are failures/blockers, never skipped passing tests.

## E2E run lifecycle

For repositories using this selected E2E-only workflow, the developer/agent runs
these separate steps for every required suite run:

1. Tear down any previous test stack, then start a fresh complete applicable
   development stack with `npm run develop` in a managed process.
2. Wait for post-mount installation, initial builds, watchers and actual
   service/device readiness.
3. Run the selected migration process where applicable and wait for success.
   Report missing required migration tooling rather than inventing a command.
4. Run the entire suite with root `npm test` on the host.
5. Tear down the stack after success or failure, including setup/migration failure;
   preserve persisted data unless removal is separately authorized.

Keep startup, migration application and teardown outside `npm test`, test hooks
and helpers; do not wrap the lifecycle in test scripts. Host migration verification
may exercise replay/locking against an isolated database as test behavior, but
must not apply the application's required startup migrations from test hooks.
Keep all tests/fixtures directly in `tests/e2e/`, without feature/platform folders
or feature-specific test commands. Do not add unit/in-process integration tests,
mocked transports or demonstration applications under this workflow. Correct
failures and rerun the full lifecycle; report unavailable verification honestly.
Publication authorization is separate where
[github-feature-workflow](../github-feature-workflow/SKILL.md) applies.

## Apply and verify

Read current manifests and instructions. Preserve selected boundaries and public
contracts; language-skill use alone does not choose database, authentication or
architecture. Honor existing user authorization and commit instructions.

Check workspace discovery, library source imports through the selected bundler/
resolver, application builds, Compose configuration and routing as available.
Do not require source libraries to resolve through unmodified native Node ESM or
add library compilation/loaders as a workaround. Run compiled application output
outside the checkout and its package scope when an application exists. A local
Caddy configuration check alone does not verify Docker builds or network routing.
Avoid printing resolved secrets. Distinguish documentation validation from a real
stack/E2E run, and report checks that could not run.

Apply [workspace-ownership](../workspace-ownership/SKILL.md) to authored filesystem
changes before handoff, including touched metadata, using owner UID `1000`.
