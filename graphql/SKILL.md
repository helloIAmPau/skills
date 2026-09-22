---
name: graphql
description: How the data API is built inside — the `graphql` workspace of these repositories. Four files and a folder: a `.graphql` schema loaded as text, a `schema.js` that finishes it by making every custom scalar a validator, an `index.js` that is only the HTTP surface, and `resolvers/` with one module per subject and a hand-written `root`. Covers what is nullable and why, how arguments are refused during coercion rather than by a resolver, what a resolver's two arguments are, how a refusal reaches the screen as a sentence, and the one function that is the browser half. Load before scaffolding a data API, adding a field, a type, a scalar or a mutation, writing a resolver, or deciding whether a check belongs in the schema or in the code.
---

# GraphQL

Read out of one repository's `graphql` workspace — 322 lines of JavaScript and a
155-line schema, of which 323 lines are comments. Where the repository and this
file disagree, the repository wins and this file needs a change.

This is the **inside** of the data API. That it exists, that it is called
`graphql`, that it owns the `/graphql` prefix and that `auth` is deliberately
not GraphQL are the `architecture` skill's. How the JavaScript in here is
written — `function` always, no `else`, promise chains, comments that say why —
is the `javascript` skill's. Neither is repeated here.

---

## What the service is made of

Five files, and the split is the point: each one can be read without the others.

```
workspaces/@<project>/graphql/
  schema.graphql     the type system, in its own language
  schema.js          builds it, and makes the custom scalars mean something
  index.js           the HTTP surface: the session, and one handler
  resolvers/
    index.js         the one module that knows there is more than one
    <subject>.js     what answers a field
  client.js          the browser half, one function
```

**Write them in that order.** The schema is the decision; everything else is a
consequence of it. A resolver written before the field it answers is a guess at
a shape.

`index.js` is 32 lines and knows nothing about what answers a field:

```js
service('graphql', function(router) {
  // Everything this service answers is a user's own data, so everything behind
  // this line requires a session. /graphql/health is registered before it by
  // service() and stays open.
  router.use(session);

  router.all('/', createHandler({
    schema,
    rootValue: root,
    // The express request, so a resolver can see who is asking.
    context: function(incoming) {
      return incoming.raw;
    }
  }));
});
```

- **One endpoint, `router.all('/')`.** No second route, no `/batch`, no GET
  variant.
- **The whole service is behind `session`**, mounted rather than checked. A
  resolver cannot forget to check because there is nothing left for it to
  check — and `/graphql/health`, registered by `service()` before the router,
  stays open so the thing the stack waits on does not need signing in.
- **`context` is the express request and nothing else.** The session middleware
  put the claims on it long before this runs, so `request.user.sub` is what a
  resolver scopes by. No bespoke context object: a second shape to build, keep
  in step, and read.

## The schema is a `.graphql` file, not a template literal

`schema.graphql`, imported and loaded as text:

```json
"build": "esbuild index.js --bundle --platform=node --format=cjs --outfile=dist/index.js --loader:.graphql=text"
```

It is a different language from the file that uses it, it is read by people who
are not reading that file, and a `.graphql` file is what an editor and a linter
expect to find. **The `--loader:.graphql=text` flag goes in every esbuild script
of the workspace** — build and watch both — and is the one thing that is easy to
forget when scaffolding: without it the bundle fails on the import, not on
anything that reads like a schema.

- **Comments are `#`, one to two lines in three**, and they say why a field is
  shaped the way it is — 121 comment lines in a 155-line schema. They are the
  documentation of the API, written for whoever opens the file.
- `"""` descriptions are not used. They would reach introspection, which is the
  argument for them; nothing here consumes introspection, and a `#` comment is
  the same shape as every comment in the project.
- The schema is also what the Dockerfile's manifest-only copy excludes
  (`--exclude=**/*.graphql`), so editing it does not reinvalidate the install
  layer. See `architecture`.

## Custom scalars are the validators

`schema.js` builds the schema and then finishes it. **It exports only the
finished one**, so `index.js` takes delivery of a type system it cannot forget
to wire:

```js
export const schema = buildSchema(definition);

schema.getType('UUID').serialize = isUuid;
schema.getType('Email').serialize = isEmail;
schema.getType('Date').serialize = isDay;
```

`buildSchema` gives a custom scalar identity behaviour — it means nothing until
something is assigned. Assigning the project's own validator is what makes the
type a type, and it means **the schema and the plain-HTTP service refuse the
same value with the same sentence, because it is the same function.**

- **`serialize` is the way out, `parseValue` the way in**, and a scalar only
  gets the direction it is used in. `UUID` and `Email` are out-only because
  nothing takes one as an argument yet — writing the other half in anticipation
  is writing a check nobody has asked for.
- **`parseValue` must be synchronous.** Whatever it returns is taken as the
  parsed value, so a promise from one is accepted *as* the day and the rejection
  nobody awaited exits the process. Measured. This is the single reason one
  validator in the project is synchronous while the rest return promises — see
  the `javascript` skill's Errors section for the shape they share.
- **Set `parseLiteral` too, and not for symmetry.** The default `parseLiteral`
  closes over the identity `parseValue` the type was *constructed* with, not the
  one assigned afterwards — so a schema with only `parseValue` set refuses
  `$from` as a variable and happily accepts `planned(from: "2026-02-30")`
  written out. A type that means one thing as a variable and another as a
  literal is worse than no type at all.

  ```js
  schema.getType('Date').parseLiteral = function(node) {
    return isDay(node.value);
  };
  ```

- **One scalar is a formatter rather than a check**, and it is worth having:
  `DateTime.serialize` turns the `Date` the driver hands back for a
  `timestamptz` into an ISO 8601 string. Doing it in the type system means every
  field that ever returns an instant is already written correctly and no
  resolver can forget. It needs no null branch — coercion returns null for a
  null field before a leaf serializer is reached.

**A scalar checks in both directions, and that is what decides whether a rule
belongs in one.** A title is `String!` and is refused by the resolver, not by a
`Title` scalar: a scalar would also check on the way out, so tightening the rule
later would make rows already stored unreadable rather than merely uncreatable.
**Put a rule in a scalar when it is a fact about the shape of the value
forever** — a uuid, an address, a day the calendar has. Put it in the resolver
when it is this year's opinion about what may be written.

## The resolvers are a folder, one module per subject

**One module per subject, not per field**, and `resolvers/index.js` is the only
thing that knows there is more than one:

```js
import { me } from './me';
import { backlog, createTask, overdue, planned } from './tasks';

export const root = { me, backlog, overdue, planned, createTask };
```

- **The keys are the field names in the schema, written out by hand.** A rename
  in the schema is a rename here, and the two cannot drift apart quietly. No
  globbing, no directory-to-field convention.
- **Queries and mutations share one `root`.** `graphql-http` takes a single
  `rootValue` and looks a field up by name whichever operation it belongs to, so
  a `Query` key and a `Mutation` key would be two nested objects nothing reads —
  and the schema already says which is which.
- **Fields that share a rule share a file.** The three board queries partition a
  person's tasks between them — one task in exactly one list — and clauses that
  have to agree should be readable together: the two that describe the same
  boundary are eleven lines apart in one file. The mutation that writes the
  column those clauses sort on lives there too.
- **A file per subject is what leaves room for the first mutation.** It lands
  next to the queries whose answers it changes rather than in a `mutations/`
  folder that groups by operation type — which is grouping by a word the schema
  already said.
- This is the one workspace shaped as a folder. Every library is flat; a package
  that arrives pre-split by subject is a layout defending against a size it does
  not have.

## A resolver is a function of two arguments

```js
export function planned({ from, to, timezone }, request) {
  const today = dayjs.utc().add(timezone, 'minute').format('YYYY-MM-DD');

  return query(`
    select id, title, planned_on, position, completed_at
    from tasks
    where user_id = $1
      and planned_on between $3 and $4
      and (planned_on >= $2 or completed_at is not null)
    order by planned_on, position, created_at
  `, [ request.user.sub, today, from, to ]);
}
```

- **Arguments destructured, `request` second.** A field taking none is
  `function(_, request)` — `_` for the arguments object nobody wants, which is
  the same convention the error handler uses for the request it does not want.
- **It returns the chain.** No `try`, no `catch`, no `async`. A rejection is
  left to reject and the service's error handler answers it.
- **The statement is the resolver.** There is no repository layer and no model:
  the query is written where the field is answered, one clause per line in a
  template literal, values always as parameters. See the `javascript` skill's
  SQL section.
- **Every field scopes by `request.user.sub`**, and an id never arrives as an
  argument. Tenancy is a `where` clause on the session's own id, and a test
  asserts one account cannot see another's rows.
- **Ordering is the server's.** A field that returns a list returns it in the
  order it is meant to be drawn in, so no client ever sorts. `order by position,
  created_at` — what somebody arranged, with what arrived first underneath.
- **A resolver that returns one row destructures the array**:
  `.then(function([ user ]) { return user; })`.
- **Sibling fields resolve concurrently**, which is what makes a split into
  several narrow fields cost one round trip rather than several. It also means
  they are several snapshots rather than one — worth a comment where it matters,
  and not worth fixing for one person's board.

## What the schema says about absence

**Nullability is the domain speaking, not a habit.** Three rules, all of them
visible in one type:

- **A list is non-null and may be empty** — `[Task!]!`. A new account has
  nothing in any list and that is the ordinary state, not an error and not an
  absence; a caller never has to tell *no tasks* apart from *no answer*.
- **A field behind the session is non-null** — `me: User!`. Everything past the
  middleware has a user by the time a resolver runs, and a nullable `me` would
  invite a check that cannot fire.
- **A nullable field means exactly one thing, and the schema says which.**
  `planned_on: Date` is null for Backlog and null for nothing else;
  `completed_at: DateTime` is null for open and nothing else. A comment above
  each says so, because *null* is the one word in a schema that can quietly come
  to mean two things.

**A mutation returns the thing it wrote, not a boolean.** `createTask(...): Task!`
hands back the row including the `position` the statement worked out, so the
client never computes where its own write went. Non-null, deliberately: a
nullable return is a shape a caller could mistake for a write that succeeded and
returned nothing — where `Task!` failing takes `data` down to null and says so.

**An argument with a sensible zero gets a default in the schema**, not in the
resolver: `timezone: Int! = 0`. A caller with no opinion about where it is is
asking in UTC, and that is a statement the type system can make. An argument
whose absence *means* something does not get one — `planned_on: Date` carries no
default, because null is Backlog and a default here would be the type system
holding an opinion the product holds elsewhere.

## Refusals are the schema's first, and they read as sentences

Two ways a request is refused, and they are told apart by the envelope:

| | answers | because |
|---|---|---|
| a variable that does not coerce | `errors`, **no `data` key at all** | the request never ran |
| a resolver that throws | `data: null` plus `errors` | it ran and failed |

**Prefer the first.** An argument typed as a custom scalar is refused during
coercion, so no resolver ever sees a day the calendar does not have and no
resolver contains a check for one.

The price is paid in the sentence, and it is paid on screen: coercion keeps the
validator's words and puts its own in front —

```
Variable "$from" got invalid value "2026-02-30"; Expected type "Date". Invalid day 2026-02-30
```

- **The message is written for the person who will read it**, because it reaches
  them verbatim through the envelope. Full sentence, capitalised, naming the
  field rather than the variable, echoing the offending value where somebody
  would retype it.
- **The envelope is GraphQL's everywhere**, including in the services that are
  not GraphQL: `{ data }` or `{ errors: [ { message } ] }`, written by the
  `service()` library before any route sees it. A client never parses two
  formats depending on which service replied.
- A resolver refusing an argument does it with the project's ordinary error —
  an `Error` carrying `status`, thrown from a validator. The `javascript`
  skill's Errors section is the shape.

## The browser half is one function

```js
export function client({ query, variables }) {
  return send('/graphql', {
    method: 'POST',
    body: JSON.stringify({ query, variables })
  });
}
```

- **The document is an argument, never something this module knows.** A client
  that knew the queries would be a second place every new field has to be added:
  the component that renders it, and then the library that spells it out. The
  query stays beside the component that asks it.
- `query` and not `document`, because `query` is what it is called on the wire —
  the argument and the field it becomes have one name between them.
- Everything else — the envelope, the credentials, the relative path, the status
  riding out on the rejection — belongs to the shared HTTP client, which is why
  GraphQL's answer shape was the one the whole project adopted.
- **This is the opposite of the `auth` client, and the difference is real.**
  `auth` is four unrelated routes with four shapes, so naming them is the only
  way to call them; `graphql` is one route with one shape, so the single export
  is named for what it is rather than for what it does.
- In React, the binding on top of it is a hook that takes the query and hands
  back `[ handler, isLoading ]` — see the `javascript` skill's React section.

## Tests

The suite talks to the running stack over HTTP. Nothing is mocked, and a
resolver is never called directly: what is under test is the statement and the
schema, and both are on the other side of a request.

- **Each test file writes its own four-line `ask({ cookie, document, variables })`**
  and hoists the documents it sends as SCREAMING_SNAKE constants.
- **Ask for all the fields a screen asks for, in one document.** What is under
  test is one request answering the whole board, not three that happen to agree
  when run apart.
- **Assert against the field a row came back in**, not against a flag inside it:
  the field *is* the answer to which list a task is in, so a test counts
  appearances across every list rather than looking only where it hopes to find
  something.
- The cases worth writing for every new field: no session is refused with the
  envelope's sentence; a forged cookie is refused *identically* to no cookie; one
  account cannot see another's rows; a value the scalar refuses answers with no
  `data` key; a value the resolver refuses answers `data: null` and the
  validator's own sentence.
- `/graphql/health` needs no session, and there is a test that says so — it is
  what the stack waits on.

---

## What this file does not settle

- **Subscriptions, fragments, interfaces, unions and pagination.** None exists
  yet. A list is a list and it is the whole list, because the lists are one
  person's own tasks.
- **Dataloader, or anything about N+1.** Every field here is one statement, and
  nothing resolves a child field with a second query. The first type that needs
  one will need a decision this file cannot make in advance.
- **Introspection and whether it should be turned off in production.** It is on,
  by default, and nobody has argued about it.
- **Where a service that is not the data API keeps its validation.** The
  validators are a shared library precisely so both can call them; which
  refusals belong to which service is the `architecture` skill's boundary
  question.
- **What the product's types mean.** This file says how a type is declared, not
  which types exist. That is the repository's own `AGENTS.md`, and where the two
  disagree `AGENTS.md` is right and this file needs a change.
