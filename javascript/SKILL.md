---
name: javascript
description: How **JavaScript** is written in these repositories, React included — the language rules only; other languages have their own skills. The style is specific and unenforced — no arrow functions, no else, no ternaries, no `!`, no `for`, promise chains in services and async/await only in tests, comments that say why at two lines in five — and in React a component is a function taking one destructured object, a boolean prop is compared with `=== true`, nothing is decided inside JSX, every handler and held child is memoised, styles are a CSS module beside the component, a glyph is an `.svg` file in that same directory used as a mask rather than a path inlined in JSX, and fonts are installed from fontsource rather than fetched from a CDN. There is no linter to catch a deviation. Load before writing or reviewing any code in these repositories, and before judging a diff's style in a pull request.
---

# JavaScript

**This file is about JavaScript and nothing else.** SQL appears in it only as
SQL written *from* JavaScript — how a query string is built and how values
reach it. The `.sql` migration files, the CSS, the shell scripts and the
Dockerfiles have conventions of their own, and none of them are here. A second
language gets a second skill rather than a section in this one. **React is not
a second language** — JSX is an expression and a component is a function — so it
is a section, at the bottom, and everything above it applies inside it.

**Read out of the code, not decided.** Every rule below was counted before it
was written, and the counts are given so a reader can check. The code is the
authority: where it and this file disagree, this file is what is wrong.

There is no linter and no formatter. The style holds because people match what
is already there, which is exactly the knowledge a fresh container lacks.

The counts come from one application repository — 3,716 lines of source across
58 files plus 2,117 lines of tests, with `dist/` and `node_modules/` excluded —
and were checked against the service repositories that share the style. 2,492 of
those source lines are the React web workspace, which is why React has a section
of its own below rather than a skill of its own: it is not a second language,
it is where most of this one is now written.

---

## The shape of a function

**The `function` keyword, always. Zero arrow functions in 3,716 lines of source
and 2,117 of tests**, and none in the sibling repositories either. A callback is
a `function`, and so is a component, an effect body, a `useCallback`, a
`useState` updater and a `.map` over a list:

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
  `function Button({ type = 'button', ... })`, `function DayPicker({ cell = 40 })`,
  `function Form({ defaults = {}, onSubmit, children })`. It is then part of the
  signature, which is where a caller reads what a component takes.
- The exception is a signature you did not choose. Express middleware is
  `function(request, response, next)` and stays that way, and a React component
  takes one object because React hands it one — many fields in that object is
  not many arguments.

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

Extract when a **second** caller appears, not in anticipation of one.

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

**Guard, then return.** There is no `else` in the codebase, and no ternary.

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

**An error is an `Error` with a `status` on it, built in three lines and called
`failed`.** Twenty of them, spelled the same way every time:

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
- **`failed` is also the name of a caught rejection**: `.catch(function(failed) { ... })`.
  Not `error`, not `e` — the variable says what happened rather than what type
  it is.

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
- camelCase for values and functions. SCREAMING_SNAKE appears only in tests.
- **A hook and a context are directories too**, named for the subject rather
  than for the mechanism: `hooks/graphql/index.js` exports `useGraphql`,
  `contexts/auth/index.js` exports `AuthProvider` and `useAuth`. A hook's name
  begins `use`; a context's file exports the provider and the hook and never the
  context object.

## Comments

**The most distinctive thing in these repositories, and denser than the last
count said: 1,571 comment lines in 3,716 — two lines in five.** The React
workspace is 937 in 2,492, near enough the same. The earlier figure of one in eight
was read off a service repository before the application was written; if a file
you are adding is under a third comment, it is probably not explaining itself
the way its neighbours do.

A component's comment sits above the `export function` and is often fifteen
lines: what the thing is, what it deliberately does not hold, what a caller has
to know, and which alternative was rejected. That header is the documentation —
there is no other.

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

## React

2,492 lines: 37 component directories, two contexts and one hook. **Everything
above holds here unchanged** — no arrows, no `else`, no ternary, no `!`, no
loop, promise chains rather than `await`, and the same comment density. What
follows is only what React adds.

### A component is a function that takes one object

```js
export function Button({ secondary, outline, chip, selected, label, value, type = 'button', disabled, onClick, children }) {
```

- `export function`, PascalCase, one component per directory, in that
  directory's `index.js`. The directory is kebab-case: `components/day-picker`
  exports `DayPicker`.
- **Props are destructured in the signature and nowhere else.** The word
  `props` does not appear in the codebase. The signature is the component's
  interface, and reading it is how a caller learns what the thing takes.
- **A flag is a named prop, not a `type` string.** `Button`'s four appearances
  are three props — `secondary`, `outline`, `chip` — and a default for the
  fourth, and which heading `ColumnHeader` draws is decided by whether it was
  given a `title` at all. A prop taking one of a handful of permitted strings is
  a thing somebody can mistype, and nothing here would catch them.

### A boolean prop is compared, never trusted

```js
// no
if (stamped) { ... }
if (!isPicking) { ... }

// yes
if (stamped === true) { ... }
if (isPicking === false) { ... }
```

A prop passed bare — `<ColumnBody stamped tasks={ tasks }/>` — arrives as
`true`, and one nobody passed arrives as `undefined`. `if (stamped)` puts those
two on the same footing as `''` and `0`, which is the objection the `??` section
already makes one section up: the check should say what it accepts.

### Nothing is decided inside JSX

**Zero `&&` and zero ternaries in any return in the codebase.** A conditional
child is a variable decided above it:

```js
let check = null;

if (completed === true) {
  check = (
    <svg width='9' height='9' viewBox='0 0 10 10' fill='none'>
      <path d='M1.4 5.2 3.9 7.7 8.6 2.3' stroke='currentColor' strokeWidth='1.7'/>
    </svg>
  );
}

return (
  <div className={ className }>
    <div className={ styles.checkbox }>{ check }</div>
    <div className={ styles.title }>{ title }</div>
    { stamp }
  </div>
);
```

`{ tasks.length && <List/> }` renders `0` on an empty list, and it reads as a
boolean right up until you notice it is markup. The variable is named for the
thing it is — `check`, `stamp`, `heading`, `body`, `sheet`, `calendar`, `rows` —
so the return reads as a list of what is on screen and the decisions are above
it, in the order they were made.

**Rendering nothing is `null`.** Never an element that hides itself: a sheet
that rendered and went invisible would still be catching every click on the
board behind it.

### A child that can be held is held

```js
const sheet = useMemo(function() {
  if (isWriting === false) {
    return null;
  }

  return <NewTask day={ openedOn } onClose={ close } onWritten={ written }/>;
}, [ isWriting, openedOn, close, written ]);
```

Five of the 23 `useMemo` in the codebase hold JSX like this — a list of rows,
an empty state, a calendar, a sheet, the columns of a span. A render that
changed nothing about the child hands React the same element back and there is
nothing to reconcile. It is the same reason
handlers are `useCallback`: what goes down to a child is a value, and a new one
every render is a re-render every render.

### className is built above the return

There is no `classnames` and no class name written as a literal — every one of
them comes off the `styles` import described below. Two shapes, and which one
you want depends on whether the classes are alternatives or can overlap:

```js
// Alternatives: returned where each is decided, so no variable holds three
// answers in turn and nobody reads on to find out whether the line above was
// the last word.
const className = useMemo(function() {
  if (outline === true) {
    return `${ styles.button } ${ styles.outline }`;
  }

  return `${ styles.button } ${ styles.primary }`;
}, [ outline ]);

// States that can overlap — a day can be today and chosen at once: an array,
// pushed, joined.
const classNames = [ styles.cell ];

if (value === now) {
  classNames.push(styles.today);
}

if (value === selected) {
  classNames.push(styles.chosen);
}
```

### Styles are CSS modules, and esbuild is what scopes them

```js
import styles from './style.module.css';
```

- **One `style.module.css` beside each component**, in that component's own
  directory, imported as `styles` — 32 of the 33 files that import a stylesheet
  spell it that way, the exception being the document component, which imports
  the global sheet as `style` because it is a document rather than a set of
  names. esbuild handles a `*.module.css` file
  natively — the class names are local and hashed per file — and the client
  build emits one `client.css` from everything the bundle imported, which the
  server shell links. No configuration beyond the file's name.
- **No CSS-in-JS, no `styled`, no `classnames`, no global class names.** Two
  screens sharing a shape share a *component*, not a string they both remember.
- **The one global sheet is the document's**, imported as text by the component
  that is the document and inlined into its `<head>` —
  `--loader:.css=text` on the server build is what hands it over verbatim. It
  holds the custom properties and the element defaults, and no class at all.
- **An inline `style` prop is for a computed custom property and nothing else**:
  `style={ { '--cell': ... } }`, `style={ { '--columns': columns.length } }`.
  Anything static belongs in the module.
- What is *inside* those `.css` files is not this file's business.

### A glyph is a file in the component's directory, never a path in the markup

**Never an inline `<svg>` in JSX, and never an `<img>`.** The drawing is its own
file next to the component that uses it — `add-button/icon.svg`,
`nav-user/icon.svg`, `week-paging/previous.svg` and `next.svg` — and it reaches
the screen through the module stylesheet as a mask:

```css
.icon {
  width: 9px;
  height: 9px;
  background-color: currentColor;
  -webkit-mask: url('./icon.svg') no-repeat center / contain;
  mask: url('./icon.svg') no-repeat center / contain;
}
```

```js
<Button outline label='Sign out' onClick={ signOut }>
  <span className={ styles.icon }></span>
</Button>
```

- **The colour has to come from the stylesheet.** A mask uses only the alpha
  channel, so the glyph is painted `currentColor` and follows its button's hover
  and its disabled state. An `<img>`, or a path with a literal `stroke`, freezes
  the colour it was drawn in and the hover stops meaning anything.
- **Path data is the one thing in a component nobody reads.** In JSX it is a
  dozen attributes of noise between the reader and the markup that matters, it
  is rebuilt on every render, and it cannot be reused by the next component that
  wants the same arrow.
- The client build carries `--loader:.svg=dataurl` for them. The server build
  needs nothing, because it renders only the shell.
- **These files are the only ones in the repository with no comment in them.**
  They are inlined as data URLs and `--minify` strips CSS comments where an XML
  comment inside a `url()` is not — measured at 1.5kb of client sheet for one
  explained glyph. The explanation goes in the stylesheet beside it.
- Four files follow this and five components in recent work still carry an
  inline `<svg>`. **The files are right**; the inline ones are the drift this
  rule exists to stop, and they get lifted out when the component is next
  touched.

### Fonts are installed from fontsource, never fetched from a CDN

```js
import '@fontsource/ibm-plex-sans/latin-400.css';
import '@fontsource/ibm-plex-sans/latin-500.css';
import '@fontsource/ibm-plex-mono/latin-400.css';
```

- **`@fontsource/<family>` as an exact-pinned dependency, imported by the client
  entry**, and only the weights and subsets the design actually uses. esbuild
  copies the faces into the assets directory and rewrites the `url()` references
  with `--loader:.woff2=file --loader:.woff=file --public-path=/assets`.
- **Never a `<link>` to a font CDN.** A font fetched from a third party on the
  sign-in screen tells that third party who is signing in, and when. npm is what
  makes self-hosting cheap: the repository stays free of binaries and the faces
  are still served from this origin.
- Both `woff2` and `woff` are emitted because the fontsource stylesheets name
  both, and only `woff2` is ever fetched. **Do not reach for esbuild's `empty`
  loader to drop the legacy files**: it leaves the `url()` behind, which then
  resolves to the document's own URL and invites a browser to fetch the page as
  a font.

### Handlers

**30 `useCallback`, and every function handed to a child or read by an effect is
one.** The dependency array is exhaustive and lists what the body reads.

- The variable is named for what it does — `open`, `close`, `add`, `choose`,
  `pick`, `backlog`, `submit`, `previous`, `next`, `today` — and the prop it
  arrives on is named for the event: `onClick`, `onChange`, `onAdd`, `onClose`,
  `onSelect`, `onWritten`. The child reports what happened; the parent names
  what it does about it.
- **One handler for a row of controls, with the answer on the element.** Forty
  calendar cells and four chips share one `choose`, and the day travels in the
  `value` attribute a button already has —
  `function({ currentTarget }) { onChange(currentTarget.value); }` — rather than
  in a closure built per cell per render. It is the destructure-what-you-came-for
  rule, applied to a DOM event.
- **A handler is given the value, not the event**, wherever the event is the
  control's own business. `Form`'s `register` hands down an `onChange(value)`
  and the `Input` is what reads `event.target.value`, because a checkbox and a
  file picker each read a different property off it.

### State

- **`useState()` with no argument where there is nothing yet**, and `== null` to
  ask. Inventing a value to stand for *not yet* is inventing a second kind of
  nothing — `const [ error, setError ] = useState()`, and `setError()` with no
  argument is how it is cleared.
- **An updater function wherever the next value is read off the current one**,
  so nothing closes over a stale one:

  ```js
  setTouched(function(current) {
    return { ...current, [ column ]: count + 1 };
  });
  ```

- A lazy initial value is a `function` too, and it means *read once*:
  `useState(function() { ... })` opens the picker on the month of the chosen day
  and does not drag a person back to it the moment they page.
- **Held and derived are different questions.** A column holds its tasks and
  derives its class from its day. Nothing holds a value it could work out from a
  prop, and nothing holds a value that is already in the URL — the span is read
  from `useSearchParams` by all three components that care about it rather than
  passed down from the screen, so they cannot disagree.

### Effects

- The body is a `function`, the work inside it is a promise chain, and the
  cleanup is a returned `function`.
- **Depend on strings, not on objects.** `parameters.get('from')` is pulled out
  above the effect so the array holds a string: `useSearchParams` hands back a
  new object whenever any part of the search string changes, and
  `location.state` is a fresh object after every navigation — an effect
  depending on an object it also causes runs forever, measured once at 417
  requests in two seconds.
- **A number that only goes up is a dependency, not a condition.** A column
  re-reads because `touched[ day ]` changed. There is nothing to guard, nothing
  to compare against and nothing to get wrong; a list of what changed would have
  to be compared against, and the comparison is where the bug lives.
- **Clear before re-reading** where the old answer would be wrong for the new
  question: `setTasks([])` first, so paging does not leave last week's Thursday
  sitting under this week's.

### Asking for data

- **A component that draws a thing asks for it.** `NavUser` fetches the address
  it shows rather than being handed one; every column on the board sends its own
  query. A component that fetches what it draws can be re-read on its own.
- The document is a template literal written in the component, beside the thing
  it fills. There is no queries module: a client that knew the queries would be
  a second place every new field has to be added.
- **The handler is named for the operation, suffixed with what it does** —
  `query Plan` becomes `planQuery`, `mutation CreateTask` becomes
  `createTaskMutation` — so a screen sending two reads as two named calls rather
  than two handlers told apart by the line they were destructured on.
- A refusal comes back as a rejected promise and the component decides what to
  show: `.catch(function(failed) { setError(failed.message); })` puts the
  server's own sentence under the field. Swallowing one happens once in the app,
  and the comment there says why it is allowed to.

### Contexts

- `createContext()` with **no default value**. It is only ever reached without a
  provider by mistake, and a value invented for that case is a value somebody
  eventually relies on.
- One file exports the `Provider` component and, at the bottom, a `useXxx()`
  that is nothing but a `useContext`. Consumers never import the context object.
- **The value is `useMemo`'d** over the callbacks it carries. A new object every
  render re-renders every consumer, and the consumers here are whole screens.
- A context holds the one question it is about and nothing adjacent to it —
  `AuthProvider` holds three states and no user, because who is signed in
  belongs to whatever puts them on screen.

### JSX spacing

- **Spaces inside braces**, including the braces of an object passed as one:

  ```js
  <div className={ styles.picker } style={ { '--cell': `${ cell }px` } }>
  ```

- **Single quotes in attributes**: `<html lang='en'>`, `type='button'`.
- **Self-closing with no space before the slash**: `<Splash/>`, `<Weekbar/>`,
  `<Day key={ day } day={ day }/>`.
- A blank line between sibling elements that are more than a word each; none
  between the three short rows of a `Task`.
- **One attribute per line once the element stops fitting one line.** A calendar
  cell puts its `key`, `type`, `value`, `className`, `aria-label`,
  `aria-pressed` and `onClick` each on their own, with the closing `>` on a line
  of its own.
- A comment inside JSX is `{/* ... */}` on its own lines, above what it explains,
  and it says why the markup is shaped that way rather than what it renders.
- **`key` is the row's own id.** `key={ index }` appears once, on the weekday
  initials of the calendar, because two Thursdays in a week of seven make the
  letter a bad key — and the comment above it says exactly that.

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
- **When a JSX element breaks across lines.** It happens when the attributes
  stop fitting, and where that is has never been agreed: `Button` renders its
  own seven attributes on one 154-column line, and a calendar cell renders
  seven on seven lines.
- **Whether any of this should be enforced.** There is no linter. Adding one
  would catch the arrow function nobody meant to write, and would also have
  opinions of its own about every rule above.
- **Every other language.** The `.sql` files, the CSS, the shell scripts and
  the Dockerfiles are written to conventions nobody has written down. This file
  deliberately does not reach for them. CSS modules are named here only where
  JavaScript imports one.
- **What each component is for.** This file says how a component is written,
  not which components exist or why the board is split the way it is. That is
  the repository's own `AGENTS.md`, and where the two disagree `AGENTS.md` is
  right and this file needs a change.
