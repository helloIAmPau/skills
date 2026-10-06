---
name: javascript
description: Write and review JavaScript source and tests using the project conventions for functions, control flow, promises, formatting, comments and modules. Use the separate React skill for components, JSX, hooks and contexts; this skill does not define project architecture.
---

# JavaScript

Adapted from [helloIAmPau/skills JavaScript skill](https://github.com/helloIAmPau/skills/blob/master/javascript/SKILL.md), with local architecture notes for Hybrid. Counts and application-specific examples describe the upstream repositories, not this project. Follow this project's `AGENTS.md` when instructions conflict; examples do not select databases, authentication, API contracts, or service boundaries. Follow [react](../react/SKILL.md) for components, JSX, hooks and contexts, and [react-native-expo](../react-native-expo/SKILL.md) for native application structure, styling and device E2E tests.

For package layout, dependency placement, application builds, containers, and the shared HTTP contract, use [npm-workspace-services](../npm-workspace-services/SKILL.md). In that architecture, libraries export their root `index.js` source directly and only exposed application service workspaces compile. The JavaScript rules below apply to both kinds of source.

For Hybrid database code, follow [clickhouse](../clickhouse/SKILL.md), including typed named parameters and ClickHouse SQL semantics. Upstream SQL examples below illustrate JavaScript structure, not a choice of database or parameter syntax for Hybrid.

**This file is about JavaScript and nothing else.** SQL appears in it only as
SQL written *from* JavaScript — how a query string is built and how values
reach it. The `.sql` migration files, the CSS, the shell scripts and the
Dockerfiles have conventions of their own, and none of them are here. A second
language gets a second skill rather than a section in this one. React-specific
conventions live in the separate [react](../react/SKILL.md) skill; these language
rules also apply to the JavaScript used in React applications.

**Existing conventions were read out of the code.** The counts are given so a
reader can check. Explicit owner requirements take precedence over existing
code; update code that violates them.

There is no linter and no formatter. The style holds because people match what
is already there, which is exactly the knowledge a fresh container lacks.

The counts come from one application repository — 3,716 lines of source across
58 files plus 2,117 lines of tests, with `dist/` and `node_modules/` excluded —
and were checked against the service repositories that share the style. 2,492 of
those source lines are the React web workspace. React-specific instructions
are maintained in the separate React skill.

---

## The shape of a function

**The `function` keyword, always. Zero arrow functions in 3,716 lines of source
and 2,117 of tests**, and none in the sibling repositories either. A callback is
a `function`, and so is a `.map` over a list:

```js
return pool.query(text, values).then(function({ rows }) {
  return rows;
});
```

- **No space before the parens, ever** — `function(request, response)`, the
  same as a named declaration. Older files carry a spaced form; convert them
  when you touch them.
- Exports differ between repositories: `export function mint()` in some,
  `export const mint = function() {}` in others. Match the file you are
  editing before matching this page. Here it is `export function`, 64 times and
  without exception.
- No classes anywhere.

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
  `response.error(error)`. The shared factory keeps the approved export form
  `export const service = function(name, handler)`; callers configure it with
  `function({ router })`.

## No operator stands in for a check

**No optional chaining. No `??`.** Both hide which absence happened.

```js
// no: a missing body, a missing field and a field of the wrong type
// all arrive here as the same empty string
const email = String(request.body?.email ?? '').trim().toLowerCase();

// yes: the check says what it accepts
const body = request.body;
let email = '';

if (body != null && typeof body.email === 'string') {
  email = body.email.trim().toLowerCase();
}
```

The long form is longer, and it says three things the short one throws away:
that a body might be absent, that the field might not be a string, and what
happens when either is true. An operator that quietly handles absence is one
that stops you asking which absence it was — and the answer is usually the
thing worth knowing.

The same objection applies one step later to a default: `?? ''` spends the last
chance to say what was missing.

**No `!` either. Zero unary `!` in 3,716 lines**, against 11 `!==` and 20
`== null`. A truth test asks a question nobody wrote down — `!stamped` is true
of `undefined`, of `''` and of `0` alike — and the character itself inverts a
line while being the easiest one on the keyboard to miss. Both directions are
written out:

```js
// no
if (!isPicking) { ... }
if (chip) { ... }

// yes: 14 `=== true` and 10 `=== false` in the codebase
if (isPicking === false) { ... }
if (chip === true) { ... }
```

This is the same rule as `===`, one step further: a comparison says what it
accepts, and a coercion says only that something was there.

**`|| {}` is the one allowance, four times, and each one says so in a comment
above it.** `request.body = request.body || {}` in `@scope/service`, and
`const { email } = location.state || {}` in the three screens that are only ever
navigated to. It is allowed exactly where the two absences are genuinely the
same absence and nothing downstream could act on the difference: a navigation
state that is missing and one carrying no address both mean *nothing navigated
here*. A bare destructure would throw on the case the line exists to handle.
That is the whole of the allowance — it is not a licence for `?? ''`, which
defaults a value rather than a container.

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

// yes: the query, where it is wanted
query(`
  insert into users (email, timezone) values ($1, coalesce($2, 'UTC'))
  on conflict (email) do update set timezone = coalesce($2, users.timezone)
  returning id
`, [ email, timezone || null ]).then(function([ { id } ]) { ... });
```

Extract when a **second** caller appears, not in anticipation of one. For React
components and hooks, follow the [single-responsibility extraction rule](../react/SKILL.md#one-scope-per-component-always),
which also requires extraction with one caller when it separates responsibilities.

The exception is an exported interface. A library's `verify` having one caller
today does not make it private — that is the shape the library offers, not an
accident of who currently uses it.

**Repeating a shape is not a second caller.** `@scope/validations` writes the
same three lines twenty times — `const failed = new Error(...)`,
`failed.status = 422`, `throw failed` — and does not extract them. What differs
between them is the sentence, which is the whole of what the extracted function
would contain; what is left is a constructor call and one assignment, and a
`refuse('The title is required')` would hide the status that is half the reason
the lines exist. Extract a second *caller*, not a second *typing*.

**One module-level helper with one caller survives, and it is worth knowing
why.** `components/plan`'s `spanAround` is called once, on the line below it.
It stays a named function because the name is the decision: it is the only
`startOf('isoWeek')` in the project, the one moment a week is chosen rather
than read, and a comment above it says which other `startOf('isoWeek')` is not
this decision said twice. A name that is the explanation earns its jump. A name
that is a summary of the body does not, which is every other case.

## Control flow

**Guard, then return.** Check exit or no-op conditions first and return immediately; keep the main path outside conditional blocks. There is no `else` in the codebase, and no ternary.

```js
if (typeof token !== 'string') {
  return null;
}

const publicKey = Buffer.from(process.env.TOKEN_PUBLIC_KEY, 'base64').toString();
```

- **`if`, never a ternary.** A ternary is an expression pretending to be a
  decision, and it stops reading as one the moment either branch grows. There
  are none; keep it that way.
- **Exact match always — `===` and `!==`.** The one exception is `== null`,
  which is not a comparison: it asks *is this absent*, and the loose form is
  the only one that answers null and undefined together. That is the whole of
  the exception; `== 0` and `== ''` are not covered by it.
- `try`/`catch` appears **once**, around `jwt.verify`, because that library
  throws for something ordinary: a caller who is not signed in. A catch is for
  converting someone else's exception into your own normal case, not for
  hiding failures — everywhere else, a rejected promise is left to reject **into
  the error handler**, which is what returning the chain is for. Left to reject
  with nothing to receive it is not restraint, it is a dead process; see
  Promises below.

**No loop in the source. Zero `for` in 3,716 lines**, and one `while` — the
calendar grid walking day by day to an end it cannot count to in advance.
Iteration is `map`, `Array.from({ length }, function(_, offset) { ... })` and
`forEach`, because each of those says what the loop is *for* in its name, where
a `for` says only that something is repeated. The four `for` loops in the
repository are all in the tests, and all of them are the same thing: polling
Mailpit for a message that has not arrived yet, which is a loop with a bail-out
count rather than an iteration over anything.

**No `++`, no `+=`, no `-=`.** Zero of each, in source and tests alike.
`attempt = attempt + 1` and `complaint = complaint + chunk` are what is written.
`++` is an assignment disguised as an expression, and it reads differently
depending on which side of the variable it lands.

## Errors

**Hybrid's shared HTTP contract uses the name `error`.** This is the owner's
explicit exception to the upstream `failed` naming below. Use
`response.error(error)` for an explicit error response and `response.data(data)`
for success; both helpers belong exclusively to `response`. The generic error
middleware delegates to `response.error(error)` without a `headersSent` check.
Status selection and payload formatting belong to that helper, as defined in
[npm-workspace-services](../npm-workspace-services/SKILL.md#shared-express-server-contract).

**An error is an `Error` with a `status` on it, built in three lines and called
`failed` in the upstream examples.** Twenty of them, spelled the same way every time:

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
- **`failed` is also the name of a caught rejection in the upstream style**:
  `.catch(function(failed) { ... })`. Hybrid's shared HTTP handlers use the
  explicit `error` exception above.

## Promises

**Services chain. Tests await.** Zero `await` in 3,716 lines of source; 372 in
the tests.

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

Measured, not inferred. A forgotten chain answered `502`, because the service
was gone:

```
Error: a promise rejected
Node.js v24.13.1
Failed running 'dist/index.js'. Waiting for file changes before restarting...
```

**This needs express 5.** express 4 does not forward a rejection at all, so
returning the chain does not save it there.

**A `.catch` that only logs is not an answer.** In a request handler it leaves
the caller waiting for a timeout while the log quietly records why. Use one only
where nothing is waiting — a background job, a fire-and-forget — and let
everything else reject into the error handler.

## Declarations and spacing

- `const` unless it is reassigned: **178 `const`, 18 `let`, 0 `var`.** A `let`
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

Write ESM source. Shared libraries remain uncompiled ES modules and expose their
root `index.js`; exposed application services bundle their own entry and imported
library source. Do not convert library source to CommonJS to match an application's
CommonJS output. Groups are separated by a blank line, in this order:

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
  Import shared workspace libraries by package name, such as `@hybrid/service`,
  rather than importing another workspace's internal file path or generated output.
- Directories are kebab-case.
- camelCase for values and functions. SCREAMING_SNAKE appears only in tests.

## Comments

**The most distinctive thing in these repositories, and denser than the last
count said: 1,571 comment lines in 3,716 — two lines in five.** The React
workspace is 937 in 2,492, near enough the same. The earlier figure of one in eight
was read off a service repository before the application was written; if a file
you are adding is under a third comment, it is probably not explaining itself
the way its neighbours do.

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

Hybrid uses E2E tests only, as requested by the owner in issue #3. Tests exercise
the running stack through its public HTTP gateway and native application UI
without mocking transport or importing application internals. Do not add unit or in-process integration tests.

- `node:test` and `node:assert/strict`.
- **A test name is a sentence about behaviour**, not a method name:
  `'a link works once'`, `'the sixth request for one address in an hour is
  accepted and not sent'`.
- The last argument to an assertion is a message saying what broke:
  `assert.equal(found.messages.length, 5, 'the rate limit did not hold at five')`.
- The suite talks to the running stack over HTTP. Nothing about the transport
  is mocked; infrastructure such as mail remains undecided for Hybrid.
- **A test takes an `async function`**, and `await` is the whole point of a test
  file: 372 of them against zero in the source. A test is a sequence of steps a
  person reads in order.
- **Module-level constants are SCREAMING_SNAKE, and only in tests.** The source
  has none at all in 178 `const`; a test file opens with a handful:

  ```js
  const BASE_URL = process.env.BASE_URL;
  const TODAY = dayjs.utc().format('YYYY-MM-DD');

  const CREATE = `mutation CreateTask($title: String!, $planned_on: Date) {
    createTask(title: $title, planned_on: $planned_on) { id title planned_on position }
  }`;
  ```

  They are the file's fixtures, and the case is what tells them apart from the
  values a test builds as it goes. A document written once at the top is also
  what stops two tests in one file asserting against two subtly different
  queries.
- **A test file writes its own helpers and does not import them from a sibling
  test.** `board.test.mjs` and `create-task.test.mjs` each define the same
  four-line `ask({ cookie, document, variables })`, deliberately: a test that
  can be read without leaving the file is worth more than the duplication is.
  What is shared lives in a module that is not a test — `session.mjs` signs
  somebody in, `fixture.mjs` puts rows in with psql.
- The `for` loops in this repository are all here, and all the same thing:
  polling Mailpit up to fifty times for a message that has not arrived yet.
  A bail-out count is not an iteration over anything, which is why `map` has
  nothing to offer it.
- **`try`/`finally` is how a test hands back what it took** — a browser page,
  a stubbed global `fetch` — and every `try` outside `jwt.verify` is one of
  these. The assertions go in the `try` and the giving back in the `finally`, so
  a failed assertion still restores what the next test needs. Nothing is caught:
  a failure is the test failing.

---

## What this file does not settle

- **Line length.** 121 of 3,716 lines exceed 80 columns and the longest is 208,
  so 80 is a habit rather than a limit. The long ones are SQL, prose comments,
  a component signature with ten props on it, and the inline HTML of the mail
  template.
- **Automated enforcement.** There is no linter. Adding one would also give it
  opinions about the rules above.
- **Every other language.** The `.sql` files, the CSS, the shell scripts and
  the Dockerfiles are written to conventions nobody has written down. This file
  deliberately does not reach for them.
