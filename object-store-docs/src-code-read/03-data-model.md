# 03. Shared Data Model

This module explains the shared semantic types that make the abstraction work.

Primary files:

- `src/path/mod.rs`
- `src/payload.rs`
- `src/util.rs`
- selected types in `src/lib.rs`

## Why This Layer Matters

If `ObjectStore` is the contract, these types are the vocabulary.

Without them, the crate would collapse into:

- raw strings for paths
- raw bytes for payloads
- backend-specific ad hoc option sets

Instead, the crate defines a domain model.

## 1. `Path`

File:

- `src/path/mod.rs`

`Path` is the crate's canonical representation of object keys.

### Why not plain `String`?

Because object-store paths need normalization and validation:

- no accidental `..`
- no empty segments
- no leading/trailing slash ambiguity
- safe handling of URL-encoded inputs
- compatibility with local filesystem translation

### What `Path` is doing architecturally

It is creating a provider-neutral key language.

```text
 user input / URL / fs path
            |
            v
        +--------+
        | Path   |
        | safe   |
        | parsed |
        +--------+
            |
            v
   backend-specific key/URL/path handling
```

That means backends can assume a cleaner key model.

## 2. `ObjectMeta`

File:

- `src/lib.rs`

`ObjectMeta` is the shared object description:

- `location`
- `last_modified`
- `size`
- `e_tag`
- `version`

### Why it matters

This is the minimum metadata set needed for:

- listing
- cache validation
- conditional requests
- optimistic concurrency

The shape is intentionally close to what cloud object stores expose.

## 3. `GetOptions`

File:

- `src/lib.rs`

This is a strong example of treating object access as a semantic request, not a
plain "read bytes" operation.

It carries:

- conditional semantics
- range semantics
- version selection
- head-only mode
- extensions

### Why this is a good module boundary

The crate could have made many separate trait methods:

- `get_if_match`
- `get_if_none_match`
- `head`
- `get_version`
- `get_range`

Instead it centralizes read semantics in one request object.

That keeps the trait surface smaller and more extensible.

## 4. `PutMode`, `UpdateVersion`, and `PutOptions`

File:

- `src/lib.rs`

These types express write semantics directly in the type system.

### Important idea

Not all writes are the same.

The crate distinguishes:

- overwrite
- create-if-absent
- update-if-current-version-matches

This is one of the biggest signs that the crate is designed for real metadata
and data-platform workflows, not just blob upload.

## 5. `PutPayload` and `PutPayloadMut`

File:

- `src/payload.rs`

`PutPayload` is not one `Bytes`.
It is an ordered collection of `Bytes`.

That means:

- cheap clone
- zero-copy chunk reuse
- delayed consolidation into contiguous bytes only if required

`PutPayloadMut` is the builder that grows in blocks.

### Why this exists

A lot of generated content does not know its final size upfront.

Using one giant `Vec<u8>` would:

- reallocate repeatedly
- copy previously written bytes

The block-based builder avoids that.

### This is a Rust-native design

The implementation is guided by:

- ownership
- `Bytes`
- `Arc`
- avoiding unnecessary copies

## 6. `GetRange` and `coalesce_ranges`

File:

- `src/util.rs`

These utilities capture an important real-world workload:

- analytics engines often need many small slices from one object

Instead of forcing users to fetch each slice independently, the crate:

- merges close ranges
- downloads combined data
- slices the result back into the requested pieces

This is a concrete example of shared performance logic that belongs above any
single backend.

## The Big Takeaway

The crate's shared data model is doing three jobs:

1. making semantics explicit
2. improving portability
3. improving performance

This is why these modules are central architecture, not helper code.

## Files To Read Next

Now move to the operational paths:

- `src/client/get.rs`
- `src/list.rs`
- `src/client/list.rs`
- `src/upload.rs`
- `src/multipart.rs`

