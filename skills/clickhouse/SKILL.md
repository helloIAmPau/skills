---
name: clickhouse
description: Maintain selected ClickHouse persistence, typed Node client access, container settings and numbered SQL migrations with replay, locking and write guarantees. Does not choose a product schema or storage engine for unrelated projects.
---

# ClickHouse

Apply when ClickHouse has been selected. Derive schema, authentication, application
permissions and write guarantees from the consuming project's approved contracts;
this skill does not select persistence for another application.

## Database access and schema

- In the selected npm architecture, use a source-only library such as
  `@scope/database` with `type: module` and root `index.js`. Declare exact-pinned
  `@clickhouse/client` in backend `devDependencies` under that architecture.
- Create one module-level official Node client using the configured endpoint,
  illustrated as `CLICKHOUSE_URL`. Export `query(text, values)`, binding named,
  typed placeholders such as `{date:Date}` through `query_params` and parsing
  `JSONEachRow`. Do not interpolate user values into SQL.
- Keep domain projections with domain code and resolvers thin. Use lowercase SQL
  keywords and snake_case identifiers; preserve case-sensitive ClickHouse types,
  engines and function names.
- Derive tables, columns, enum values, units and validation from actual requirements.
  Do not install a product-specific schema. Preserve optional measurements as
  nullable fields when absence is meaningful rather than inventing default zeros.
- Use native `Date` storage for calendar dates when that is the selected domain
  representation. Define the public format/validation contract explicitly and use
  typed parameters; preserve date meaning independently of server timezone.
- Choose ordering keys, engines, partitions and indexes for established access
  patterns. A MergeTree ordering key does not enforce uniqueness. Explicit UUIDs
  may identify logical writes but do not by themselves prevent duplicate inserts.
- Use synchronous inserts or explicitly wait for async insert completion when
  read-after-write visibility is promised. Retries are not automatically idempotent
  and multiple statements are not implicitly transactional. Define retry, duplicate
  and multi-row write guarantees for each writer.

## Migrations

- Keep plain SQL under root `migrations/`, with zero-padded sequential filenames
  and a purpose. Apply in filename order using the configured runner, illustrated
  as `migrations/apply.sh`. Never edit shipped migration history; add a new file.
- Source selected settings first, require the configured database/user/endpoint,
  and never print credentials. Use the native `clickhouse-client` and the documented
  database protocol; host verification requires it installed on the host and a
  published endpoint. Do not migrate inside an application listener or add a framework.
- Keep a MergeTree ledger with `name String` and `applied_at DateTime64(3, 'UTC')`.
  Check filenames, skip completed files, and validate names before using ledger SQL.
- Serialize every runner targeting the database with a shared server-associated
  migration lock. Preserve the configured server-container lock when that is the
  selected mechanism; a host driver may coordinate its acquisition through Docker
  control operations. A host-local lock alone cannot exclude independent runners.
  Inspect a lock surviving a killed runner before manual removal; never steal it.
- Apply each file with `--multiquery`, stopping on the first error. Write its ledger
  entry only after the complete file succeeds. DDL and ledger writes are not a
  transaction: prior statements can survive failure, or execution can stop after
  SQL success but before recording the ledger.
- Make replay safe after partial application with `if not exists` where appropriate;
  data migrations require an explicit replay strategy. Inspect partial state before
  retrying and never mark incomplete files applied. Do not enable experimental
  transactions to imitate another database.
- Apply after actual database readiness and before serving a release requiring its
  schema. Compose dependency ordering is not readiness.

## Containers and configuration

- Pin the server image, resolve the service name and connect it to the backend
  network. Backend applications join it when needed. Under the selected Compose
  convention, omit `expose`, health checks and `service_healthy` conditions.
- Bind storage/logs under `./data/<database-service>/` to the image's documented
  paths, such as `/var/lib/clickhouse` and `/var/log/clickhouse-server`.
  Preserve persisted data; engine changes do not authorize deletion.
- Supply configured database user/password/name/URL explicitly with plain Compose
  interpolation. Keep credentials out of native bundles.
- Development-only host publication is allowed for database/migration tests or
  defined fixture administration; publish only needed protocol endpoints, with
  host-loopback bindings for local access. Do not add production publication merely
  to satisfy tests or give native tooling credentials just to run host test drivers.

## Verification

Follow the shared [host-test/public-interface contract](../npm-workspace-services/SKILL.md#host-testing-and-public-interfaces)
and [E2E lifecycle](../npm-workspace-services/SKILL.md#e2e-run-lifecycle).
Root `npm test` and migration verification drivers run on the host; services remain
in the selected stack. Keep tests/fixtures flat under `tests/e2e/`. Database tests
use the documented protocol against an isolated database; application assertions
use the public gateway/UI. Direct database access is limited to database/migration
tests or defined fixture administration.

Cover first application, repeat skipping, partial failure without a success
ledger, safe retry and runner exclusion. Preserve unrelated records and clean up
fixture identifiers; wait for deletions/mutations to complete. Faults/delays use
controlled real server conditions, never mocked transports or production test
flags. Verify compiled backends outside the checkout/package scope after client
changes. Report unavailable verification accurately.
