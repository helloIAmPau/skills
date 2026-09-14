---
name: code-style
description: How **JavaScript** is written in these repositories — the language rules only; other languages have their own skills. The style is specific and unenforced — no arrow functions, no else, no ternaries, promise chains in services and async/await only in tests, comments that say why — and there is no linter to catch a deviation. Load before writing or reviewing any code in these repositories, and before judging a diff's style in a pull request.
---

# Code style: JavaScript

**This file is about JavaScript and nothing else.** SQL appears in it only as
SQL written *from* JavaScript — how a query string is built and how values
reach it. The `.sql` migration files, the CSS, the shell scripts and the
Dockerfiles have conventions of their own, and none of them are here. A second
language gets a second skill rather than a section in this one.

**Read out of the code, not decided.** Every rule below was counted before it
was written, and the counts are given so a reader can check. The code is the
authority: where it and this file disagree, this file is what is wrong.

There is no linter and no formatter. The style holds because people match what
is already there, which is exactly the knowledge a fresh container lacks.

The counts come from one service repository — 810 lines of source across
eleven files, with `dist/` and `node_modules/` excluded — and were checked
against two others that share the style.

---

## The shape of a function

**The `function` keyword, always. Zero arrow functions in 810 lines**, and none
in the sibling repositories either. A callback is a `function`:

```js
return pool.query(text, values).then(function({ rows }) {
  return rows;
});
```

- **No space before the parens, ever** — `function(request, response)`, the
  same as a named declaration. Older files carry a spaced form; convert them
  when you touch them.
- Exports differ between repositories: `export function useAuth()` in some,
  `export const useAuth = function() {}` in others. Match the file you are
  editing before matching this page.
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
- The exception is a signature you did not choose. Express middleware is
  `function(request, response, next)` and stays that way.

## Control flow

**Guard, then return. There is no `else` in the codebase, and no ternary.**

```js
if (typeof token !== 'string') {
  return null;
}

const publicKey = Buffer.from(process.env.TOKEN_PUBLIC_KEY, 'base64').toString();
```

- `== null` means *absent* — null or undefined and nothing else. Five uses, all
  deliberate. `===` and `!==` are for comparing values.
- `try`/`catch` appears **once**, around `jwt.verify`, because that library
  throws for something ordinary: a caller who is not signed in. A catch is for
  converting someone else's exception into your own normal case, not for
  hiding failures — everywhere else, a rejected promise is left to reject.

## Promises

**Services chain. Tests await.** Zero `await` in the workspaces; 42 in the
tests.

```js
// a service
consume(token).then(function(userId) {
  ...
}).catch(function(failed) {
  console.error('callback failed', failed);
});

// a test
const landed = await fetch(link, { redirect: 'manual' });
assert.equal(landed.status, 302);
```

Different jobs. A service composes operations that mostly hand one value to the
next; a test is a sequence of steps a person reads in order, and `await` is how
that reads.

## Declarations and spacing

- `const` unless it is reassigned: **88 `const`, 1 `let`, 0 `var`.**
- Two spaces. Single quotes. Semicolons.
- **A template literal for anything that spans lines or interpolates** —
  `` `${ base }/auth/callback?token=${ token }` ``, never `+` and never an
  array joined on a newline. Spaces inside the braces, as everywhere else. A
  single-line string with nothing to interpolate stays in single quotes:
  backticks there are noise.
- **Spaces inside array brackets**: `[ userId, hash(token) ]`,
  `const [ state, setState ] = useState('LOADING')`.
- **Spaces inside JSX braces**: `{ children }`, `value={ value }`, and single
  quotes in attributes: `<html lang='en'>`.
- A blank line before a `return` that follows other work — never inside a
  one-line guard, where it would separate the check from its answer.
- A blank line after a guard block, before the work it was protecting.

## Imports

ESM only; the bundler decides what it becomes. Groups separated by a blank
line, in this order:

```js
import { randomBytes, createHash } from 'node:crypto';

import { serve } from '@scope/service';
import { query } from '@scope/postgres';

import { mint, consume } from './tokens';
```

Node builtins carry the `node:` prefix — `node:crypto`, `node:test`,
`node:assert/strict`.

## Naming and files

- A directory that holds code holds an `index.js`, and that is its entry.
- Directories are kebab-case, including component directories. The React
  component inside is PascalCase.
- camelCase for values and functions.

## Comments

**The most distinctive thing in these repositories: roughly one line in eight.**

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

- `node:test` and `node:assert/strict`.
- **A test name is a sentence about behaviour**, not a method name:
  `'a link works once'`, `'the sixth request for one address in an hour is
  accepted and not sent'`.
- The last argument to an assertion is a message saying what broke:
  `assert.equal(found.messages.length, 5, 'the rate limit did not hold at five')`.
- The suite talks to the running stack over HTTP and reads mail out of Mailpit.
  Nothing about the transport is mocked.

---

## What this file does not settle

- **Line length.** 39 of 810 lines exceed 80 columns and the longest is 117,
  so 80 is a habit rather than a limit. Most of the long ones are SQL and
  prose comments.
- **Whether any of this should be enforced.** There is no linter. Adding one
  would catch the arrow function nobody meant to write, and would also have
  opinions of its own about every rule above.
- **Every other language.** The `.sql` files, the CSS, the shell scripts and
  the Dockerfiles are written to conventions nobody has written down. This file
  deliberately does not reach for them.
