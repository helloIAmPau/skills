---
name: react
description: Write and review React web services with server-rendered documents, separate browser entries and minimal components with one responsibility. Apply component, JSX and hook conventions to React Native; document rendering, CSS, DOM and fontsource rules apply to web apps.
---

# React

Follow [javascript](../javascript/SKILL.md) for language conventions, including
functions, control flow, promises, formatting and comments. This skill adds
React-specific rules. Follow [react-native-expo](../react-native-expo/SKILL.md)
for native application structure, styling and device E2E tests; CSS modules,
DOM event examples, SVG masks and fontsource instructions below apply to web
applications only.

Follow the consuming project's applicable instructions and explicit user decisions.
The web service structure applies when that architecture is selected; React
component conventions alone must not migrate an existing application's architecture.

## One scope per component, always

**Every React and React Native component must perform exactly one scope of
responsibility and be as minimal as possible.** Enforce this even when existing
code combines responsibilities.

- Keep only the props, state, handlers, effects, markup and styles needed for
  that responsibility. Remove unused code, speculative variants and unnecessary
  wrappers; preserve required behavior and accessibility.
- Split independently meaningful UI concerns into focused child components.
  Screens compose those components; move separate data/lifecycle orchestration
  into focused hooks rather than accumulating it in a screen component.
- Extract a component or hook when it separates responsibilities, even with
  only one caller. This takes precedence over the single-caller helper rule
  in [javascript](../javascript/SKILL.md#where-a-function-lives). Create only
  units that are actually used.
- During implementation and review, identify each component's one responsibility.
  A component that mixes independent concerns must be split before the work is
  considered complete. Judge minimality by responsibility and necessary code,
  rather than an arbitrary line count or compressed formatting.

## The web service renders the document on the server

**For the selected web service architecture, render the document on the server
with a separate browser entry.** React Native keeps its native entry and rendering model.

- Workspace-root `index.js` is the server entry. Render `<Page/>` per document
  request with `renderToPipeableStream` from `react-dom/server`. Follow the
  consuming project's shared HTTP service contract and route definitions.
- `components/page/index.js` is server-only and owns the `<html>` document:
  metadata, document-wide styles, the browser mount root and noscript guidance.
  Render an empty `<div id='root'></div>`; keep browser state and effects in the
  interactive application. Do not serve a copied static HTML template.
- Workspace-root `client.js` is the separate browser entry. Import self-hosted
  fonts there and use `createRoot` to mount the interactive application into
  `#root`. This mounts into an empty root; do not describe it as hydration of
  server-rendered application content.
- Keep every web UI component under the workspace's `components` folder,
  including page-level composition such as `components/workspace`. Do not create
  a separate `screens` folder. Hooks and contexts remain separate non-component
  modules. The Page owns only the document; page-level components compose focused
  browser components.
- Supply `/assets/client.js` through `bootstrapScripts`; React injects the
  bootstrap tag. Set the HTML response type and pipe the stream in
  `onShellReady`. Forward failures before the shell is ready through the
  project's HTTP error handler using `onShellError`.
- The Page imports its global `style.css` as `style` and inlines that static
  text in `<head>`. The server build uses JSX and `.css=text`; the browser build
  uses JSX, CSS modules and IIFE output for the classic bootstrap script.
  Link `/assets/client.css` from the document. Emit browser assets under
  `dist/assets`, resolve them beside the compiled server entry and keep the
  production artifact independent of checkout source.
- Configure esbuild through CLI arguments in npm scripts. Use separate server
  and browser build commands; reuse them with `--watch=forever` for development.
  Do not introduce `build.js`, `build/index.js` or other JavaScript build helpers.
- Watch server document/style imports and browser component/style imports with
  their respective builds. Verify the actual document, browser startup, assets,
  error handling and source rebuilds through the project's required tests.

The server handler's rendering boundary is:

```js
const stream = renderToPipeableStream(<Page/>, {
  bootstrapScripts: [ '/assets/client.js' ],
  onShellReady: function() {
    response.type('html');
    stream.pipe(response);
  },
  onShellError: function(error) {
    next(error);
  }
});
```

## A component is a function that takes one object

```js
export function Button({ secondary, outline, chip, selected, label, value, type = 'button', disabled, onClick, children }) {
```

- `export function`, PascalCase, one component per directory, in that
  directory's `index.js`. The directory is kebab-case: `components/day-picker`
  exports `DayPicker`.
- **Destructure props in the signature.** The signature is the component's interface.
- **A flag is a named prop**, such as `secondary`, `outline` or `chip`, rather
  than a variant string. Keep actual multi-valued domain props distinct from flags.

## A boolean prop is compared, never trusted

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
in [javascript](../javascript/SKILL.md#no-operator-stands-in-for-a-check)
already makes: the check should say what it accepts.

## Nothing is decided inside JSX

**No `&&` or ternaries in JSX.** A conditional child is decided above it:

```js
const check = useMemo(function() {
  if (completed !== true) {
    return null;
  }

  return <span className={ styles.check }></span>;
}, [ completed ]);

return (
  <div className={ className }>
    <div className={ styles.checkbox }>{ check }</div>
    <div className={ styles.title }>{ title }</div>
    { stamp }
  </div>
);
```

For this web example, `styles.check` masks a component-local `check.svg` through
the stylesheet convention below. Native conditional children use native components.

`{ tasks.length && <List/> }` renders `0` on an empty list, and it reads as a
boolean right up until you notice it is markup. The variable is named for the
thing it is — `check`, `stamp`, `heading`, `body`, `sheet`, `calendar`, `rows` —
so the return reads as a list of what is on screen and the decisions are above
it, in the order they were made.

**Return `null` when rendering nothing.** Initialize a conditional child from
its computed render result, as above, instead of leaving a local variable
uninitialized. Never render an element that hides itself: it can still intercept
clicks behind it.

## A child that can be held is held

```js
const sheet = useMemo(function() {
  if (isWriting === false) {
    return null;
  }

  return <NewTask day={ openedOn } onClose={ close } onWritten={ written }/>;
}, [ isWriting, openedOn, close, written ]);
```

Use `useMemo` for held JSX such as rows, empty states and conditional panels.
A render that changed nothing about the child hands React the same element back and there is
nothing to reconcile. It is the same reason
handlers are `useCallback`: what goes down to a child is a value, and a new one
every render is a re-render every render.

## className is built above the return

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

## Styles are CSS modules, and esbuild is what scopes them

This is the web's rule. A native app has no CSS: its styles are a
`StyleSheet` above the component in its own file, as defined in
[react-native-expo](../react-native-expo/SKILL.md).

```js
import styles from './style.module.css';
```

- **One `style.module.css` beside each component**, imported as `styles`.
  The server document imports its global text stylesheet as `style` instead.
  esbuild handles CSS modules, hashes local class names and emits `client.css`
  from browser imports; the server document links it.

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

## A glyph is a file in the component's directory, never a path in the markup

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
- Keep SVG files free of comments: comments can survive data URL embedding.
  Put the glyph's explanation in the stylesheet. Move existing inline glyphs
  into component-local SVG files when touching their component.

## Fonts are installed from fontsource, never fetched from a CDN

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

## Handlers

**Every function handed to a child or read by an effect uses `useCallback`.** The dependency array is exhaustive and lists what the body reads.

- The variable is named for what it does — `open`, `close`, `add`, `choose`,
  `pick`, `backlog`, `submit`, `previous`, `next`, `today` — and the prop it
  arrives on is named for the event: `onClick`, `onChange`, `onAdd`, `onClose`,
  `onSelect`, `onWritten`. The child reports what happened; the parent names
  what it does about it.
- **One handler for a row of controls, with the answer on the element.**
  Repeated controls can share one `choose`, reading their value from the
  button's `value` attribute —
  `function({ currentTarget }) { onChange(currentTarget.value); }` — rather than
  in a closure built per cell per render. It is the destructure-what-you-came-for
  rule, applied to a DOM event.
- **A handler is given the value, not the event**, wherever the event is the
  control's own business. `Form`'s `register` hands down an `onChange(value)`
  and the `Input` is what reads `event.target.value`, because a checkbox and a
  file picker each read a different property off it.

## State

- **Prefer a meaningful initial state when one exists.** Use `useState()` with
  no argument when there is no value yet. Treat `null` and `undefined` as
  equivalent absence, following JavaScript's
  [initialization rule](../javascript/SKILL.md#absence-and-initialization).
  Do not use `useState(null)` or `useState(undefined)`. For example,
  `const [ error, setError ] = useState()`; clear it with `setError()`.
  Check absence with `== null` or `!= null`, and pass optional values directly
  rather than using `value || null`. Initialize concrete required state, such as
  a counter map, with its actual initial value.
- **An updater function wherever the next value is read off the current one**,
  so nothing closes over a stale one:

  Initialize the counter map with `useState({})`. Inside an event or completion
  handler, derive every previous-state-dependent value from the updater parameter:

  ```js
  setTouched(function(current) {
    const count = current[ column ];

    if (count == null) {
      return { ...current, [ column ]: 1 };
    }

    return { ...current, [ column ]: count + 1 };
  });
  ```

  Never set state during render. Successive events and asynchronous completions
  from the same render each advance the counter; do not use an outer captured
  count, including for invalidation counters read by effects.

- A lazy initial value is a `function` too, and it means *read once*:
  `useState(function() { ... })` opens the picker on the month of the chosen day
  and does not drag a person back to it the moment they page.
- **Held and derived are different questions.** A column holds its tasks and
  derives its class from its day. Nothing holds a value it could work out from a
  prop. Read URL-backed state from the selected router rather than keeping a
  second state value that could disagree.

## Effects

- The body is a `function`, the work inside it is a promise chain, and the
  cleanup is a returned `function`.
- **Depend on strings, not on objects.** `parameters.get('from')` is pulled out
  above the effect so the array holds a string: `useSearchParams` hands back a
  new object whenever any part of the search string changes, and
  `location.state` is a fresh object after every navigation — an effect
  depending on an object it also causes can loop indefinitely.
- **A number that only goes up is a dependency, not a condition.** A column
  re-reads because `touched[ day ]` changed. There is nothing to guard, nothing
  to compare against and nothing to get wrong; a list of what changed would have
  to be compared against, and the comparison is where the bug lives.
- **Clear before re-reading** where the old answer would be wrong for the new
  question: `setTasks([])` first, so paging does not leave last week's Thursday
  sitting under this week's.

## Asking for data

- **A component that draws a thing asks for it.** `NavUser` fetches the address
  it shows rather than being handed one; every column on the board sends its own
  query. A component that fetches what it draws can be re-read on its own.
- When GraphQL is used, the query document is a template literal written in the
  component, beside the thing it fills. There is no queries module: a client that
  knew the queries would be a second place every new field has to be added.
- **The handler is named for the operation, suffixed with what it does** —
  `query Plan` becomes `planQuery`, `mutation CreateTask` becomes
  `createTaskMutation` — so a screen sending two reads as two named calls rather
  than two handlers told apart by the line they were destructured on.
- A refusal comes back as a rejected promise and the component decides what to
  show: `.catch(function(failed) { setError(failed.message); })` puts the
  server's own sentence under the field. Intentionally ignored failures need
  an explicit reason; ordinary errors must remain visible.

## Contexts

- **A hook and a context are directories too**, named for the subject rather
  than for the mechanism: `hooks/graphql/index.js` exports `useGraphql`,
  `contexts/auth/index.js` exports `AuthProvider` and `useAuth`. A hook's name
  begins `use`; a context's file exports the provider and the hook and never the
  context object.

- `createContext()` with **no default value**. It is only ever reached without a
  provider by mistake, and a value invented for that case is a value somebody
  eventually relies on.
- One file exports the `Provider` component and, at the bottom, a `useXxx()`
  that is nothing but a `useContext`. Consumers never import the context object.
- **The value is `useMemo`'d** over the callbacks it carries. A new object every
  render re-renders every consumer, including consumers that render whole screens.
- A context holds one concern. Derive its state and public interface from the
  application's actual needs; do not prescribe an authentication model.

## JSX spacing

- **Spaces inside braces**, including the braces of an object passed as one:

  ```js
  <div className={ styles.picker } style={ { '--cell': `${ cell }px` } }>
  ```

- **Single quotes in attributes**: `<html lang='en'>`, `type='button'`.
- **Self-closing with no space before the slash**: `<Splash/>`, `<Weekbar/>`,
  `<Day key={ day } day={ day }/>`.
- Use a blank line between substantial sibling elements; closely related short
  rows may stay together.
- **One attribute per line once the element stops fitting one line.** A calendar
  cell puts its `key`, `type`, `value`, `className`, `aria-label`,
  `aria-pressed` and `onClick` each on their own, with the closing `>` on a line
  of its own.
- A comment inside JSX is `{/* ... */}` on its own lines, above what it explains,
  and it says why the markup is shaped that way rather than what it renders.
- **`key` is the row's stable id.** An index is appropriate only for a fixed,
  non-reordered list without an unambiguous natural key; explain that exception.

## Component comments

A component's header comment explains its responsibility, necessary caller
knowledge and non-obvious choices. Scale the comment to those needs.

For native asynchronous font startup, follow the
[font startup contract](../react-native-expo/SKILL.md#font-startup): pending keeps
the splash, success renders the application, failure releases the splash and
renders an error view using system fonts. Keep that view independent of success-only layouts.

## What this skill does not settle

- **Line wrapping.** Break attributes across lines when readability requires it;
  no fixed column threshold is prescribed.
- **Automated enforcement.** Component scope and minimality remain review checks
  regardless of lint tooling; do not add tooling incidentally.
- **What each component is for.** The project's `AGENTS.md` defines the product
  architecture and component boundaries.
