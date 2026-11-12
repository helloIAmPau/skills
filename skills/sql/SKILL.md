---
name: sql
description: Write and review SQL queries and migrations with readable clauses, minimal nesting, named WITH stages, bound parameters and explicit transactions. Apply to standalone SQL and SQL embedded in any host language; database capabilities and schema come from the consuming project.
---

# SQL

Follow the consuming project's requirements and selected database dialect. Use
this skill for query structure and SQL migrations alongside the host-language
skill. Apply [clickhouse](../clickhouse/SKILL.md) only for selected ClickHouse
persistence. This skill does not choose a database, schema or client library.

## Readable queries

**Optimize for a person reading the query.** Prefer simple, explicit steps over
the shortest SQL or the fewest statements.

- Use lowercase keywords and snake_case identifiers. Preserve case-sensitive
  names required by the dialect, including types, engines and functions.
- Put each clause on its own line in a multiline statement. Indent CTE bodies
  and continued expressions with two spaces. Break long column lists and
  conditions where it helps the reader follow the logic.
- Name CTEs and aliases for the rows or result they represent. Avoid anonymous
  stages such as `q1`, `q2` and `tmp`.
- Keep a simple select, join or update direct. Do not wrap it in a CTE that only
  repeats the same query.

**Avoid nesting wherever practical. Use `with` to name the stages of a complex
query.** Flatten nested derived tables into sibling CTEs in one top-level `with`
block, ordered so each stage follows the stages it references. Give each CTE a
clear purpose; do not split every clause into a separate CTE.

Keep a nested subquery only when it is necessary or makes the intent clearer,
such as a short correlated `exists` check. Preserve row counts, null handling,
ordering and authorization conditions when rewriting; replacing an existence
check with a join must not duplicate rows.

For example, with PostgreSQL placeholders bound by the caller:

```sql
with customer_totals as (
  select customer_id, sum(amount) as total_amount
  from orders
  where created_at >= $1
  group by customer_id
)
select customers.id, customers.name, customer_totals.total_amount
from customers
join customer_totals on customer_totals.customer_id = customers.id
where customer_totals.total_amount >= $2
order by customer_totals.total_amount desc, customers.id;
```

CTEs describe query structure, not a procedural execution sequence. Do not rely
on their written order for side effects or assume they improve performance.
Check the selected database's execution and snapshot rules when they matter;
for PostgreSQL, see its [WITH documentation](https://www.postgresql.org/docs/current/queries-with.html).

## Parameters and embedded SQL

- **Bind runtime values separately from SQL text.** Interpolating values into
  the statement is a review blocker. Use the actual driver's placeholder syntax.
- Dynamic identifiers and sort directions use a controlled mapping and the
  driver's supported identifier quoting where needed; value placeholders do not
  stand in for SQL syntax.
- Keep multiline SQL in a multiline literal supported by the host language,
  such as a JavaScript template literal. Keep its clauses visible in the source
  instead of assembling them with string concatenation.

## Transactions

**Use transactions when related statements must succeed or fail together.**
Prefer a readable sequence of simple statements in an explicit transaction when
that expresses the operation more clearly than one complex statement. Transaction
helpers are allowed; use the project's interface or introduce a small helper
when it makes connection ownership and cleanup clear.

- Use transaction support actually provided by the selected database and engine.
  Multiple statements or CTEs do not by themselves provide a transaction across
  separate calls.
- Keep the transaction on the same connection or driver-managed transaction
  context. Commit on success, roll back on failure and release the connection
  on every path. With node-postgres, follow its
  [transaction contract](https://node-postgres.com/features/transactions).
- Choose isolation and locking for the operation's concurrency requirements.
  Adding a transaction alone does not fix every read-then-write race.
- Keep transactions focused on the database work that belongs together; perform
  unrelated network calls outside the transaction.

## Migrations

Keep migrations in numbered `.sql` files, applied in order and recorded after
success by the project's migration runner. Skip successfully applied files.
Never edit or rename a shipped migration; a correction is a new file. Follow the
selected database's DDL, transaction and replay guarantees, including recovery
after partial application when DDL cannot be rolled back.
