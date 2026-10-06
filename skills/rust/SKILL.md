---
name: rust
description: Write and review Rust code in helloIAmPau's style, derived from pul.se and Olivia. Apply to Rust modules, error handling, async execution, typed configuration and Cargo manifests; project architecture and test policy remain in AGENTS.md.
---

# Rust

Match the owner's Rust code rather than substituting generic idiomatic Rust.
These conventions are inferred from the Rust source in
[pul.se](https://github.com/helloIAmPau/pul.se) and
[Olivia](https://github.com/helloIAmPau/olivia). The
[source reference](references/source-style.md) records the inspected commits,
representative files and variations. Read it when deciding whether an unusual
pattern is a convention or an exception.

Follow the consuming project's `AGENTS.md` for architecture, dependencies,
concurrency, persistence and verification. This skill defines the Rust code's
shape. Its examples do not authorize importing either project's infrastructure.

## Layout and formatting

- Indent with **two spaces**, including `impl`, `match` arms and closures.
  Use double-quoted strings and snake_case module files, functions and variables.
  Types and enum variants use PascalCase; Rust constants use SCREAMING_SNAKE_CASE.
- Declare modules near the top of the crate/module entry, then imports. Keep
  related types and their implementations in the module that owns their subject.
  Use ordinary Rust `src/main.rs`, `src/lib.rs` and nested `mod.rs` where needed.
  JavaScript's workspace-root entry convention does not apply to Rust.
- Write one imported path per `use` declaration. Group related imports with
  blank lines: standard library, dependencies and local modules. Match the
  surrounding grouping; do not collapse imports into `use crate::{ A, B };`
  or impose alphabetical ordering on unrelated code.
- Import the functions and types actually used. Qualify local cross-module
  imports with `crate::`; use aliases to distinguish error/result names, such
  as `Error as IoError` and `Result as FormatterResult`.
- Separate construction, validation, side effects and the returned result with
  blank lines when they are distinct steps. Name intermediate values so the
  sequence can be read directly.
- In multiline structs, enum definitions and initializers, separate entries
  with commas and omit the final trailing comma. Multiline block `match` arms
  commonly end in `},` except the final arm. Terminate statements with `;`.
- Keep short signatures, calls and builder chains on one line as in the source.
  There is no demonstrated hard 80-column limit. Break genuinely unwieldy code
  for readability while preserving two-space indentation.
- Do not run stock formatter defaults over the repository: preserve the
  observed indentation and comma layout. Use an existing compatible
  formatter configuration if present, and review its output against this skill.
  Do not introduce a formatter or linter as an incidental style change.

## Control flow and returns

Use explicit `return` for function results and early exits, including the final
successful result. Keep direct value expressions inside `match` arms used to
initialize a variable. A function that only dispatches on an enum can use
`return match ...;`.

Handle fallible operations with `match`, at the place they occur:

```rust
let client = match Client::builder().build() {
  Ok(client) => client,
  Err(error) => {
    return Err(AgentError::Request(error));
  }
};

return Ok(client);
```

- Do not replace these matches with `?`, `.map_err(...)`, `try!`, `if let`
  or `let ... else` when writing code in this style. Explicit handling is
  consistent across both inspected projects, rather than a one-file preference.
- Match `Option` explicitly when extracting a value, choosing a fallback or
  returning an error. Match compound states directly, such as `Ok(Some(entry))`,
  `Ok(None)` and `Err(error)`.
- When only failure matters, use `Err(error) => { ... }, _ => {}`. When the
  success value matters, name it in `Ok(value) => value`. Do not silently
  discard failures; the surrounding operation determines whether to propagate,
  report, retry or intentionally ignore them.
- Prefer guards that return or continue before the main work. For a negative
  boolean test, write `condition == false`; positive tests such as
  `if values.is_empty()` are also used. Do not introduce `if !condition`.
- `if`/`else` is allowed for genuine two-way selection. Both repositories use
  it; the JavaScript skill's ban on `else` does not apply to Rust.
- Use ordinary `for`, `loop`, `break` and `continue` for sequential operations
  and stateful execution. Small iterator closures such as `.any(...)`,
  `.map(...)` and `.collect(...)` are also present; do not ban them or convert
  a readable operation into a long combinator chain.
- Do not use `.unwrap()` or `.expect()` for operational errors. Non-panicking
  `.unwrap_or(...)` for an intentional fallback is present in pul.se; it is
  not the same pattern. Defaults must still follow the project's config rules.
- Olivia uses `panic!` for fatal startup/CLI failures, while pul.se logs and
  returns. Preserve the chosen entry-point policy. Do not carry startup panics
  into recoverable task, model, database or tool execution paths.

## Types, constructors and ownership

- Model configuration, requests, responses, parameters and state with named
  structs and enums. Use `Option<T>` for genuinely optional values and
  `Result<T, DomainError>` for fallible operations.
- Put construction/loading behavior on the owning type: `new`, `load` and
  small domain methods. Build resources step by step, then return the instance.
  An async constructor is appropriate when construction actually needs I/O.
- Both `field: field` and field shorthand occur. pul.se favors explicit field
  names; Olivia commonly uses shorthand. Follow the nearby initializer style;
  in a new module, shorthand is supported by the more recent Olivia code.
- Pass one named parameter/configuration struct when the domain already has
  one. Borrow inputs such as `&str`, `&self` and `&mut self` where appropriate,
  and move owned configuration into its owner. Do not import JavaScript's
  two-positional-argument limit: Rust source has larger signatures.
- Use `.to_string()` when an owned string is needed from borrowed text. Clone
  when ownership requires it, such as shared `Arc` handles or retained messages;
  do not add clones to avoid considering ownership.
- Derive the traits the value needs, such as `Deserialize`, `Serialize`,
  `Debug`, `Clone` or `PartialEq`. Do not add a blanket set to every type.

## Errors

Write named domain errors and their `Display` implementation by hand. Olivia
uses enums that retain underlying typed errors; pul.se also uses a
`<Domain>ErrorKind` enum plus a `<Domain>Error` struct containing kind and message.
Preserve the error family already used by the module. For a new integration
module, Olivia's typed enum is the closer model:

```rust
use std::io::Error as IoError;
use std::fmt::Display;
use std::fmt::Formatter;
use std::fmt::Result as FormatterResult;

#[derive(Debug)]
pub enum WorkspaceError {
  Io(IoError),
  MissingInput
}

impl Display for WorkspaceError {
  fn fmt(&self, formatter: &mut Formatter) -> FormatterResult {
    return match self {
      WorkspaceError::Io(error) => write!(formatter, "IO Error - {}", error),
      WorkspaceError::MissingInput => write!(formatter, "Missing input")
    };
  }
}
```

Map errors explicitly at the boundary, and propagate an existing domain error
unchanged when it already expresses the failure. Do not introduce `anyhow`,
`thiserror`, automatic `From` conversions or a global boxed-error abstraction
merely to shorten the code. Implement additional standard traits when an actual
API contract requires them; their absence in the samples is not a prohibition.

## Async work and external data

- Use `async fn` and direct `.await` for asynchronous work, with the same explicit
  matches as synchronous code. Tokio-based programs use `#[tokio::main]`.
- Make shared ownership and locking visible with `Arc` and the mutex type
  appropriate to the operation. Release guards when their protected work ends;
  do not add shared state or spawned tasks just because Olivia has services.
- Follow the project's concurrency contract. Hybrid requires one sequential
  model/tool loop per agent with concurrency across separate engine instances.
  Olivia's `JoinSet` service startup is not permission to parallelize that loop.
- Use typed Serde request/response/config structs and explicit parsing. Use
  `#[serde(rename_all = ...)]`, tagged enums and named `default_*` functions
  when those shapes or defaults are actually needed. Derive `JsonSchema` only
  when a schema-consuming interface needs it.
- Compose readable strings with `format!`; use raw multiline strings for
  substantial prompts or text templates. Keep wire format, schemas and prompt
  policy specific to the consuming application.
- Use parameterized database calls with values supplied separately, as in
  pul.se's PostgreSQL wrapper. Do not transfer Olivia's model-facing arbitrary
  SQL tool into Hybrid's internal persistence layer.

## Comments, Cargo and verification

Use comments for intent, protocol constraints and non-obvious behavior. Use
`///` descriptions for schema-exposed parameters when callers or the model need
their meaning. Avoid adding comments that merely narrate each Rust statement.

Both inspected projects use Rust edition `2024`, regular Cargo dependencies and
feature lists. Follow the consuming crate's manifest; use edition `2024` for new
crates in this project unless its toolchain contract says otherwise. Verify
dependency versions when selecting them. The examples do not require Wasmtime,
GStreamer, LiteLLM, Axum, S3 or any particular database driver.

Verify compilation and behavior through the consuming project's required
commands. The inspected source does not establish a Rust unit-test convention.
Hybrid's E2E-only policy remains in its `AGENTS.md`; do not add a competing
`cargo test` policy through this skill. During review, check the two-space
layout, separate imports, explicit matches/returns, domain errors and the
project's actual execution invariants.
