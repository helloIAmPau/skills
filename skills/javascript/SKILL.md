---
name: javascript
description: Write and review JavaScript source and tests using the project conventions for functions, control flow, promises, formatting, comments and modules. Use the separate React skill for components, JSX, hooks and contexts; this skill does not define project architecture.
---

# JavaScript

Follow the consuming project's applicable instructions and the user's explicit decisions.
Use [react](../react/SKILL.md) for components, JSX, hooks and contexts, and
[react-native-expo](../react-native-expo/SKILL.md) for selected native applications.

This skill defines JavaScript conventions, not architecture. Apply
[npm-workspace-services](../npm-workspace-services/SKILL.md) only when that
architecture is selected. Its libraries export source and its application services
bundle it; language-skill use alone must not scaffold infrastructure or change architecture.
Use [clickhouse](../clickhouse/SKILL.md) only for selected ClickHouse persistence.
SQL examples here illustrate JavaScript structure; derive parameter syntax from
the actual database. CSS, shell, Dockerfiles and SQL migrations have separate conventions.

## The shape of a function

**Use the `function` keyword, including callbacks.** Do not use arrow functions.
A `.map` over a list takes a `function` too:

```js
return pool.query(text, values).then(function({ rows }) {
  return rows;
});
```

- **No space before the parens, ever** — `function(request, response)`, the
  same as a named declaration. Older files carry a spaced form; convert them
  when you touch them.
- Prefer `export function mint()` for named exports. Preserve a selected
  interface that uses `export const mint = function() {}`.
- Do not use classes.

## Arguments

**One or two. A third means an object.**

```js
export function mint({ userId, minutes, limit, window }) { ... }
export function attach({ response, token, days }) { ... }
```

- A third positional is one nobody can read at the call site:
  `attach(response, token, 30)` says nothing about what 30 is.
- **Destructure what you came for.** A callback that wants one field takes one
  field — `function({ rows })`, `function({ id })`. A query returning a single
  row destructures the array, `function([ user ])`, which turns a
  `rows.length === 0` check into `user == null` and reads better for it.
- **A default belongs in the destructure**, not in the body:
  `function mint({ userId, minutes = 30 })`. It is then part of the signature,
  which is where a caller reads what the function takes.
- The exception is a signature you did not choose. Express middleware is
  `function(request, response, next)` and stays that way.
- Express error middleware keeps all four arguments:
  `function(error, request, response, next)`, even when its body only returns
  `response.error(error)`. The shared factory keeps the selected export form
  `export const service = function(name, handler)`; callers configure it with
  `function({ router })`.

## Absence and initialization

Treat `null` and `undefined` as the same absence. Use `value == null` or
`value != null` when checking optional values; do not distinguish the two for
ordinary absence handling.

Prefer initializing local variables with a real, meaningful value at declaration.
Avoid bare declarations such as `let selected;`. Derive the value before declaring
it, or compute it with guard-based control flow. Use `const` for values that do not
change; use an initialized `let` when reassignment is needed. Do not invent an
unrelated placeholder just to supply an initializer.

Do not initialize missing values explicitly to `null` or `undefined`. Omit missing
object fields and avoid parameter defaults that merely spell out absence. Concrete
initial values such as zero for a counter or an empty collection are appropriate
when they represent the actual starting state. React state with no value yet uses
`useState()` rather than an explicit absence initializer.

Pass optional values directly. Use `variable`, never `variable || null` or
`variable ?? null` to normalize absence. An absent value needs no replacement.
React follows the same rule with `useState()`; see its
[state guidance](../react/SKILL.md#state).

## No operator stands in for a check

**No optional chaining. No `??`.** Both hide which absence happened.

```js
// no: a missing body, a missing field and a field of the wrong type
// all arrive here as the same empty string
const email = String(request.body?.email ?? '').trim().toLowerCase();

// yes: guard absence and wrong types before the main path
function readEmail(body) {
  if (body == null) {
    return '';
  }

  if (typeof body.email !== 'string') {
    return '';
  }

  return body.email.trim().toLowerCase();
}
```

The explicit checks distinguish a missing body from an invalid field and make
handling visible. Both `value == null` and `value != null` are allowed.
Prefer guards with early exits to unnecessary nested conditionals.

**No unary `!` or implicit truth tests.** Write boolean comparisons explicitly:

```js
// no
if (!isPicking) { ... }
if (chip) { ... }

// yes
if (isPicking === false) { ... }
if (chip === true) { ... }
```

**`|| {}` is the only fallback-operator allowance.** Use it only for container
defaulting where an absent container and an empty one have the same meaning,
such as `const { email } = location.state || {}`. Explain that equivalence in a
comment. This does not permit value defaulting such as `?? ''`.

## Where a function lives

**A function used in exactly one place is written in that place.**

A named helper with a single caller costs a name to invent, a jump to follow
and a second scope to hold — and buys nothing. The reader has to go and look at
the body anyway, and the name is a summary that will go stale before the code
it summarises does.

```js
// no: a helper nobody else calls
function upsert(email, timezone) {
  return query(`insert into users ...`, [ email, timezone ]);
}

upsert(email, timezone).then(function({ id }) { ... });

// yes: the query, where it is wanted; timezone was already validated
query(`
  insert into users (email, timezone) values ($1, coalesce($2, 'UTC'))
  on conflict (email) do update set timezone = coalesce($2, users.timezone)
  returning id
`, [ email, timezone ]).then(function([ { id } ]) { ... });
```

The example assumes `timezone` has already been validated for the query contract.
Pass it directly; do not normalize an absent value with `timezone || null`.

Extract when a **second** caller appears, not in anticipation of one. For React
components and hooks, follow the [single-responsibility extraction rule](../react/SKILL.md#one-scope-per-component-always),
which also requires extraction with one caller when it separates responsibilities.

The exception is an exported interface. A library's `verify` having one caller
today does not make it private — that is the shape the library offers, not an
accident of who currently uses it.

**Repeating a shape is not a second caller.** Similar error construction can
stay inline when the message and status are the meaningful differences. Extract
shared behavior, rather than hiding those decisions behind a helper name.

A single-caller named helper may remain when its name explains a domain decision
that would otherwise need a comment. A name that only summarizes the body does
not justify extraction.

## Control flow

**Guard, then return.** Check exit or no-op conditions first and return immediately; keep the main path outside conditional blocks. Do not use `else` or ternaries.

```js
if (typeof token !== 'string') {
  return null;
}

const publicKey = Buffer.from(process.env.TOKEN_PUBLIC_KEY, 'base64').toString();
```

- **`if`, never a ternary.** A ternary is an expression pretending to be a
  decision, and it stops reading as one the moment either branch grows.
  Use explicit guards and returns.
- **Exact match — `===` and `!==`.** Permit both `== null` and `!= null`
  to check null and undefined together. This exception does not cover `== 0`
  or `== ''`.
- Use `try`/`catch` to translate an external exception into an intentional normal
  case, such as an invalid credential. Do not hide operational failures; return
  rejected promise chains to their error handler.

Prefer `map`, `Array.from({ length }, function(_, offset) { ... })` and `forEach`
for collection iteration. Use `while` for progression whose end cannot be counted
in advance. Tests may use bounded `for` loops for polling with a clear timeout.

**No `++`, no `+=`, no `-=` in source or tests.**
`attempt = attempt + 1` and `complaint = complaint + chunk` are what is written.
`++` is an assignment disguised as an expression, and it reads differently
depending on which side of the variable it lands.

## Errors

**The selected shared HTTP contract uses the name `error`.** When applying
[npm-workspace-services](../npm-workspace-services/SKILL.md#shared-express-server-contract),
use `response.error(error)` and `response.data(data)` exclusively on `response`.
The generic middleware delegates to `response.error(error)` without a
`headersSent` check. Status and payload formatting belong to that helper.

**Construct a user-facing error as an `Error` with a `status`.** Other JavaScript
error construction and caught rejections use the name `failed`:

```js
const failed = new Error(`Invalid email address ${ parsed }`);
failed.status = 422;

throw failed;
```

- **The message is the sentence the person will read**, written for them rather
  than for a log: *The title is required*, *A title is one line*. It reaches the
  screen unchanged — the envelope carries it out and the component puts it under
  the field — so there is no second copy of the wording in the browser.
- `status` is what turns it into a response. Without one it is a fault here
  rather than a refusal of theirs, and the service answers 500.
- **Echo the value that was refused, unless echoing it is the problem.** An
  address and a day are echoed, because somebody retyping one needs to see what
  arrived; a token is not, because nobody types one and echoing it puts a
  credential-shaped string in every log on the way out; a title is not, because
  it is still sitting in the field that refused it and it can be two hundred
  characters long.
- Caught rejections use `.catch(function(failed) { ... })`; shared HTTP
  handlers use the `error` interface above.

## Promises

**Services chain. Tests await.**

```js
// a service — returned, so a rejection reaches the error handler
router.post('/callback', function(request, response) {
  return consume(token).then(function(userId) {
    response.data({ userId });
  });
});

// a test
const landed = await fetch(link, { redirect: 'manual' });
assert.equal(landed.status, 302);
```

Different jobs. A service composes operations that mostly hand one value to the
next; a test is a sequence of steps a person reads in order, and `await` is how
that reads.

### Return the chain

**A handler returns its promise.** express 5 forwards a rejection from a
returned promise to the service's error handler, which answers in the
envelope.

A chain that is **not** returned is not merely unhandled: it becomes an
unhandled rejection and **node exits**. The process dies, not the request, and a
runtime image carrying no watch and no restart policy does not come back — one
rejected query takes the service down for everyone.

**This needs express 5.** express 4 does not forward a rejection at all, so
returning the chain does not save it there.

**A `.catch` that only logs is not an answer.** In a request handler it leaves
the caller waiting for a timeout while the log quietly records why. Use one only
where nothing is waiting — a background job, a fire-and-forget — and let
everything else reject into the error handler.

## Declarations and spacing

- `const` unless it is reassigned; use `let` for reassignment and never `var`. A `let`
  is almost always one thing: a value decided by a guard that has more than one
  outcome, declared with its plainest answer and reassigned by an `if` — which
  is what this style writes instead of an `else` or a ternary.
- Two spaces. Single quotes. Semicolons.
- **A template literal for anything that spans lines or interpolates** —
  `` `${ base }/auth/callback?token=${ token }` ``, never `+` and never an
  array joined on a newline. Spaces inside the braces, as everywhere else. A
  single-line string with nothing to interpolate stays in single quotes:
  backticks there are noise.
- **Spaces inside array brackets**: `[ userId, hash(token) ]`,
  `const [ first, second ] = values`.
- A blank line before a `return` that follows other work — never inside a
  one-line guard, where it would separate the check from its answer.
- A blank line after a guard block, before the work it was protecting.

## Imports

Write ESM source with extensionless relative JavaScript source imports. This
intentionally requires the application's selected bundler/resolver; unmodified
native Node ESM resolution is not a supported source verification path.
In the selected workspace architecture, libraries expose root `index.js` source
and application services bundle it. Do not add library compilation, a loader,
or a CommonJS migration to accommodate Node's resolver. Keep extensions on actual
filenames, manifest exports targets, entrypoints, output paths and asset imports.
Executable host test files must use imports compatible with their actual runner;
they are distinct from bundled application source.

Groups are separated by a blank line, in this order:

```js
import { randomBytes, createHash } from 'node:crypto';

import { service } from '@scope/service';
import { query } from '@scope/postgres';

import { mint, consume } from './tokens';
```

Node builtins carry the `node:` prefix — `node:crypto`, `node:test`,
`node:assert/strict`.

## Naming and files

- A directory that holds code holds an `index.js`, and that is its entry.
- Workspace entry points are root `index.js` files, without a `src` subfolder.
  Import shared workspace libraries by package name, such as `@scope/service`,
  rather than importing another workspace's internal file path or generated output.
- Directories are kebab-case.
- camelCase for values and functions. Use SCREAMING_SNAKE for test constants only.

## Comments

Explain intent and non-obvious constraints with comments; do not target a
comment count or density.

- Full sentences, capitalised, with full stops.
- They say **why**, never what. A comment that restates the line below it gets
  deleted.
- They frequently name the alternative that was rejected, and why it lost:

  ```js
  // Counting and minting are one statement, and the advisory lock is why. Two
  // requests for the same address arriving together would otherwise both count
  // the rows before either inserted, and both would pass a limit of five.
  ```

- Above the thing, never trailing it.
- A rule with its reason attached survives a case it did not anticipate. That
  is the whole argument for the density.

## SQL

- Lowercase keywords, snake_case identifiers.
- **Values are always parameters.** Interpolating into the text is a review
  blocker, not a style preference.
- **A statement spanning lines is a template literal**, one clause per line,
  so the shape of the query is visible in the shape of the source:

  ```js
  query(`
    update login_tokens set consumed_at = now()
    where token_hash = $1 and consumed_at is null and expires_at > now()
    returning user_id
  `, [ hash(token) ])
  ```

- Migrations are numbered `.sql` files, applied once, never edited after they
  ship. A mistake is a new file.

## Tests

Use E2E tests only: exercise the running system through its public interfaces,
without mocked transports or application-internal imports. Root `npm test` and
all child test drivers run on the host. Follow the shared
[host testing and public interfaces](../npm-workspace-services/SKILL.md#host-testing-and-public-interfaces)
and [E2E run lifecycle](../npm-workspace-services/SKILL.md#e2e-run-lifecycle).
Applying this language skill alone does not select a stack or require a mobile app.

- `node:test` and `node:assert/strict`.
- **A test name is a sentence about behaviour**, not a method name:
  `'a link works once'`, `'the sixth request for one address in an hour is
  accepted and not sent'`.
- The last argument to an assertion is a message saying what broke:
  `assert.equal(found.messages.length, 5, 'the rate limit did not hold at five')`.
- HTTP application assertions use the configured public gateway URL; other
  targets use their documented public protocol or UI.
- **A test takes an `async function`** and uses `await` for ordered steps.
- **Module-level constants are SCREAMING_SNAKE, and only in tests.** For example:

  ```js
  const BASE_URL = process.env.BASE_URL;
  const TODAY = dayjs.utc().format('YYYY-MM-DD');

  // Illustrative document for an already defined public API.
  const READ = `query Resource($id: ID!) {
    resource(id: $id) { id }
  }`;
  ```

  They are the file's fixtures, and the case is what tells them apart from the
  values a test builds as it goes. A document written once at the top is also
  what stops two tests in one file asserting against two subtly different
  queries.
- **A test file writes its own helpers and does not import a sibling test.**
  Share fixture or session administration through non-test modules only where
  needed, using the defined public/admin interface. Keep tests and fixtures flat
  under `tests/e2e/`.
- Bounded polling loops have an explicit timeout and failure result.
- **Use `try`/`finally` to restore acquired resources or temporarily modified
  fixtures.** Failed assertions must still restore state. Do not stub transport
  or global `fetch`; application assertions exercise real public behavior.

---

## What this file does not settle

- **Line length.** There is no fixed column limit. Break long code for readability.
- **Automated enforcement.** Do not add a linter or formatter as an incidental
  change; preserve compatible existing tooling.
- **Every other language.** This file does not prescribe CSS, shell, Dockerfile
  or SQL migration syntax; consult their applicable guidance.
