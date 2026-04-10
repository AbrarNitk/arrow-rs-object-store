# 01. Crate Entry And Shape

This module explains how the crate is laid out from the entry point outward.

Primary file:

- `src/lib.rs`

Important supporting files:

- `Cargo.toml`
- `README.md`

## What To Notice First

The crate is shaped around a single public abstraction:

- `ObjectStore`

But the repository is not organized as "one trait and many random backends".

It has a deliberate layering:

```text
src/lib.rs
  |
  +-- public trait and public semantic types
  |
  +-- shared modules
  |    - path
  |    - payload
  |    - upload
  |    - multipart
  |    - list
  |    - util
  |
  +-- backend modules
  |    - memory
  |    - local
  |    - aws
  |    - gcp
  |    - azure
  |    - http
  |
  +-- composition modules
       - prefix
       - limit
       - throttle
       - registry
       - parse
```

## Why `src/lib.rs` Matters So Much

`src/lib.rs` is not a thin re-export file. It is the semantic center of the
crate.

It defines:

- what an object store is
- what operations are guaranteed
- what metadata means
- what error categories exist
- what convenience APIs users should call

When reading `src/lib.rs`, read it in this order:

1. crate-level docs
2. module declarations and feature gates
3. `ObjectStore`
4. `ObjectStoreExt`
5. request/response types
6. error model

## Feature-Gated Shape

`Cargo.toml` shows an important design decision:

- in-memory support is always available
- filesystem support is optional through `fs`
- cloud support is layered through `cloud`
- providers are separately gated

This means the repository is intentionally modular both:

- at source layout level
- at compile-time dependency level

## Why This Shape Is Good Rust Design

The crate avoids two common mistakes:

1. putting too much logic directly into backend implementations
2. forcing every user to compile every provider

Instead it centralizes semantic logic and pushes protocol-specific code
downward.

## Files To Read Now

Read these next:

- `src/lib.rs`
- `Cargo.toml`
- `README.md`

## Questions To Keep In Mind

- Why does the crate define both `ObjectStore` and `ObjectStoreExt`?
- Why are `path`, `payload`, `upload`, and `list` separate modules instead of
  nested helper types?
- Why are local and memory backends separate from cloud shared client code?

