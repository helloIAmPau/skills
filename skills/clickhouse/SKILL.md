---
name: clickhouse
description: Implement Hybrid ClickHouse access, container configuration and numbered SQL migrations. Use when adding persisted features, changing the entries schema or maintaining the migration runner.
---

# ClickHouse

Use ClickHouse for durable application data. Do not add another storage engine implicitly. Ownership, authentication and write semantics remain subject to the relevant approved feature.

## Database access

- Keep the source-only `@hybrid/clickhouse` workspace with `type: module` and root `index.js`. Declare an exact-pinned `@clickhouse/client` in backend `devDependencies`.
- Create one module-level official Node client using `CLICKHOUSE_URL`. Export `query(text, values)`, which binds named, typed placeholders such as `{date:Date}` through `query_params` and returns rows parsed from `JSONEachRow`. Do not interpolate user values into SQL.
- Keep domain projections with their domain code and resolvers thin. Use lowercase SQL keywords and snake_case identifiers; preserve ClickHouse's case-sensitive type, engine and function names.
- Use synchronous inserts or explicitly wait for asynchronous insert completion when a feature promises read-after-write visibility. Do not assume a MergeTree ordering key enforces uniqueness, that retries are automatically idempotent, or that multiple statements form a transaction. Define these guarantees in each writer's approved contract.

## Entries

- Use one `entries` table with Date, Session, Type, Activity, Reps, Weight, Duration and Quantity represented as `date`, `session`, `type`, `activity`, `reps`, `weight`, `duration` and `quantity`, plus stable `id` and creation timestamp.
- Store calendar dates in a native `Date` column; exchange `YYYY-MM-DD` strings at the public boundary. The shared `isDate` validator chains `isString` and checks only that format; do not add calendar validity, database-range checks or Date-object conversion to this API type. Preserve the string unchanged and bind it through typed parameters. Native ClickHouse storage retains its own date behavior and supported range. Preserve selected dates independently of server timezone.
- Types are `BIO`, `ENDURANCE`, `STRENGTH` and `FOOD`; sessions are `MORNING`, `AFTERNOON` and `EVENING`. The app's Weight section reads Bio records. Activity distinguishes Weight, named endurance/strength activities and food measurements such as Calories, Carbs, Fats and Proteins.
- Preserve absent measurements as nullable fields, never default zeros. Duration is seconds; format it for display. Food measurement rows remain separate. Do not infer totals, reconciliation, duplicate handling or multi-row write atomicity from this shape.
- Start with MergeTree ordered by `(date, created_at, id)`. Add engines, partitions or indexes only for an established access pattern. Use explicit UUIDs when a feature needs stable logical write identity; a UUID default alone does not prevent duplicate inserts.

## Migrations

- Keep plain SQL files under root `migrations/`, named with zero-padded sequential prefixes and a purpose. Apply in filename order with `migrations/apply.sh`; add new files after a migration ships rather than editing applied history.
- Use the native `clickhouse-client` inside the running `clickhouse` Compose service. Source the selected environment first and require `CLICKHOUSE_USER` and `CLICKHOUSE_DB`; never print credentials. Do not add a migration framework or migrate in an application listener.
- Keep a MergeTree `migrations` ledger containing `name String` and `applied_at DateTime64(3, 'UTC')`. Check filenames before running and skip completed ones. Validate filenames before placing them in ledger SQL.
- Serialize runners with the server-container migration lock. A lock surviving a killed runner requires inspection before manual removal; do not steal it automatically.
- Run each SQL file with `--multiquery`, stopping on the first error; write the ledger entry only after the whole file succeeds. Do not claim transactional DDL/ledger rollback. Earlier statements may survive a failure, and a process can stop after SQL succeeds but before the ledger is written.
- Make migrations safely rerunnable after partial application, using `if not exists` where appropriate. Data migrations need their own replay strategy. After failure, inspect partial state and retry only after confirming replay is safe; never mark an incomplete file as applied. Do not enable experimental transactions to imitate another database.
- Apply explicitly after database readiness and before serving a release that needs the schema. A Compose dependency is not readiness.

## Containers and configuration

- Pin the `clickhouse/clickhouse-server` image. Name the service `clickhouse`; join only `backend`. Backend consumers join `backend` and their existing proxy network. Publish no database ports, add no `expose`, health check or `service_healthy` condition.
- Bind storage and logs under `./data/clickhouse/`, targeting `/var/lib/clickhouse` and `/var/log/clickhouse-server`. Do not delete prior database data as part of changing engines.
- Configure `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`, `CLICKHOUSE_DB` and `CLICKHOUSE_URL` explicitly with plain Compose interpolation. Keep database credentials out of native bundles. Tooling may receive `HYBRID_E2E_CLICKHOUSE_URL` and join `backend` solely for real E2E fixtures; the app always uses Caddy and GraphQL.

## Verification

Run the full `npm run develop` stack and confirm readiness before all tests via `npm test`. Keep tests flat under `tests/e2e/`. Migration checks run from the Docker-capable host against an isolated database; HTTP/native E2E runs inside mobile tooling using the real database and public gateway.

Cover first application, repeat skipping, partial failure without a success ledger, safe retry and runner exclusion. Use real entries with nullable measurements, preserve unrelated records and clean up fixture IDs. Database faults and delays must be controlled real server conditions, never mocked transports or production test flags. Wait for fixture deletions/mutations to complete before proceeding. Validate the compiled backend outside its workspace package scope after changing the client.
