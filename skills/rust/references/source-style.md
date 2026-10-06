# Source evidence and variations

The Rust skill is an interpretation of the owner's code, not an upstream style
guide. Its instructions target the repeated patterns below. These snapshots
were inspected on 2026-10-06:

- pul.se: commit
  [`03be032c33edf774407448bd2f330c4f6e928dab`](https://github.com/helloIAmPau/pul.se/tree/03be032c33edf774407448bd2f330c4f6e928dab),
  five Rust files under `remuxer/src/`.
- Olivia: commit
  [`3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0`](https://github.com/helloIAmPau/olivia/tree/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0),
  23 Rust files across `harness/src/` and `tools/*/src/`.

The corpus contains 4,585 lines including comments, prompts and whitespace.
Formatting strings containing `{:?}` and URL strings containing `?` are not
error-propagation operators. Prompt prose mentioning `else` is not Rust control
flow. Check source context before drawing a rule from text counts.

## Representative modules

| Source | What to inspect |
| --- | --- |
| [pul.se main](https://github.com/helloIAmPau/pul.se/blob/03be032c33edf774407448bd2f330c4f6e928dab/remuxer/src/main.rs) | Module declarations, separate imports, Tokio entry point, stepwise resource construction and log-and-return startup failures. |
| [pul.se PostgreSQL wrapper](https://github.com/helloIAmPau/pul.se/blob/03be032c33edf774407448bd2f330c4f6e928dab/remuxer/src/postgres.rs) | Error kind/message family, handwritten Display, explicit field initialization, match-based pool/query handling and parameter values separate from SQL. |
| [pul.se protocol](https://github.com/helloIAmPau/pul.se/blob/03be032c33edf774407448bd2f330c4f6e928dab/remuxer/src/protocol.rs) | Trait callbacks, intermediate bindings, early returns, state dispatch and readable sequential operations. |
| [pul.se stream](https://github.com/helloIAmPau/pul.se/blob/03be032c33edf774407448bd2f330c4f6e928dab/remuxer/src/stream.rs) | Structs/enums, owned settings, explicit Option matches, intentional unwrap_or defaults and an actual if/else selection. |
| [Olivia configuration](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/harness/src/config.rs) | Typed error variants, result/error import aliases, return match in Display and explicit async read/parse handling. |
| [Olivia model client](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/harness/src/agent/llm_client.rs) | Typed request/response structs, long signatures/builders, field shorthand, explicit request/parsing errors and negative boolean comparison. |
| [Olivia agent](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/harness/src/agent/mod.rs) | Domain error enum, named Serde defaults, raw prompt text, bounded explicit loop, result/tool state dispatch and deliberate if/else payload updates. |
| [Olivia tool registry](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/harness/src/agent/tool_registry.rs) | Matching Ok(Some), Ok(None) and Err, continue guards, explicit resource ownership and domain failures. |
| [Olivia services](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/harness/src/services/mod.rs) | Separate imports, typed config variants, explicit for loops, Arc sharing, JoinSet and while let. These implement Olivia's services, not Hybrid's concurrency contract. |
| [Olivia HTTP tool](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/tools/http/src/lib.rs) | Named parameter struct, documented fields, enum-to-method selection, optional-value matches and sequential builder changes. |
| [Olivia S3 tool](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/tools/s3/src/lib.rs) | A focused local helper, constant in application source, enum dispatch and compact single-line error arms. |
| [Olivia common tool definition](https://github.com/helloIAmPau/olivia/blob/3e08db92b19c1f0c7887f4f13b8ba3a0e92766c0/tools/common/src/lib.rs) | A macro justified by repeated tool exports. This does not establish a universal preference for macros. |

## Strong common patterns

Both projects consistently use two-space blocks, separate import declarations,
named data/domain types, explicit matching of fallible operations and explicit
returns. Neither inspected source uses the error-propagation `?` operator,
`.map_err`, `.unwrap()`, `.expect()` or `if let` for these operations. The source
does not use `thiserror`/`anyhow` to replace its domain errors.

This supports enforcing those conventions in new code rather than refactoring
them into a different Rust style. It does not establish that every valid Rust
construct missing from the corpus must be banned.

## Variations that must remain visible

- **Returns:** explicit returns dominate. A few implementations end with a
  bare `match`, and the hello tool has a tail struct expression. Use explicit
  returns for new code without claiming the snapshots contain no exceptions.
- **Initializers:** pul.se repeats `field: field`; Olivia uses shorthand
  extensively. Both forms belong to the owner's code.
- **Errors:** pul.se uses a kind/message struct; Olivia preserves underlying
  error values in enums. Preserve local error families, and prefer the latter
  for a new typed integration module.
- **Booleans and branches:** negative tests use `== false`; positive predicate
  calls are also used. `else` occurs in real source. There is no Rust ban on it.
- **Defaults and panics:** pul.se has non-panicking `unwrap_or` configuration
  fallbacks. Olivia has startup/CLI `panic!` calls. Neither establishes a reason
  to unwrap recoverable operations or silently default required settings.
- **Iteration:** Olivia uses explicit loops and small iterator closures. It
  also uses `while let` to collect service tasks; absence of `if let` does not
  justify banning that different construct.
- **Punctuation and wrapping:** omitted final commas and long single-line
  signatures/chains are common, but semicolons and compact error arms vary.
  Preserve coherent nearby formatting rather than accidental whitespace.
- **Tooling/tests:** neither inspected tree supplies a rustfmt configuration
  or Rust test functions. This supports no particular test architecture or
  formatter settings beyond the observed source style.

Do not copy isolated mixed tabs, stray indentation, typos, secret-bearing log
content, application-specific prompts or operational mistakes as conventions.
Reuse the owner's code structure while satisfying the new application's
correctness and privacy requirements.
