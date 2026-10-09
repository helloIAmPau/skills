# Rust examples and permitted variations

Read this reference when selecting an error family, initializer, startup policy
or control-flow variation. It supplements the [Rust conventions](../SKILL.md),
without choosing infrastructure or forbidding every unmentioned Rust construct.

## Error families

Preserve a module's existing domain error family. New integration modules may
use a typed enum retaining underlying errors, as shown in the skill. A kind/message
family is also valid:

```rust
#[derive(Debug)]
pub enum StorageErrorKind {
  Connection,
  Query
}

#[derive(Debug)]
pub struct StorageError {
  pub kind: StorageErrorKind,
  pub message: String
}
```

Implement `Display` by hand and any standard traits required by actual callers.
Translate external errors explicitly at boundaries; propagate an existing domain
error unchanged when it already expresses the failure.

## Initialization

Both repeated field names and shorthand are permitted. Match surrounding code;
prefer shorthand in a new module when no local instruction differs:

```rust
let explicit = Settings {
  address: address
};

let shorthand = Settings {
  address
};
```

These are alternative examples: initialize the selected form, rather than moving
the same non-Copy value twice. Construct resources step by step and use explicit
matches for fallible initialization before returning the completed instance.

## Control flow

Use explicit function returns, including successful final returns. Direct values
inside `match` arms initializing a variable remain appropriate. A dispatch-only
function can `return match ...;`. Existing tail expressions may be preserved when
required by a selected interface; they do not change the preference for new code.

- Negative boolean tests use `== false`; positive predicate calls are permitted.
- `if`/`else` is permitted for actual two-way selection; JavaScript's rule differs.
- Explicit loops and small iterator closures are permitted. `while let` may collect
  completed service tasks; the rule against `if let` for fallible extraction does
  not imply a ban on this distinct iteration construct.
- Non-panicking `.unwrap_or(...)` is allowed for intentional configuration defaults,
  subject to the configuration contract. Do not default required values silently.
- Fatal startup/CLI failures may panic or log and return under the chosen entry-point
  policy. Recoverable operations return domain errors; do not use `.unwrap()` or
  `.expect()` to handle them.

## Formatting and tooling

Keep two-space indentation, separate imports and the selected comma layout.
Compact match error arms and long signatures may remain when readable. Preserve
coherent local formatting, rather than accidental whitespace. Use compatible
existing formatter settings; do not introduce tooling or a test architecture
because an example happens to omit it.
