# 02. The `ObjectStore` Contract

This module is about the core trait design.

Primary file:

- `src/lib.rs`

Main items to read:

- `ObjectStore`
- `ObjectStoreExt`
- `ListResult`
- `ObjectMeta`
- `GetOptions`
- `GetResult`
- `PutMode`
- `PutOptions`
- `CopyOptions`
- `RenameOptions`
- `Error`

## The Most Important Design Choice

The crate separates:

- the backend implementor contract
- the user convenience API

In code, that means:

- `ObjectStore` is the required backend trait
- `ObjectStoreExt` is the extension trait for ergonomic calls

## Why This Split Exists

The crate used to rely more heavily on default trait methods.

That is convenient, but it creates a subtle problem:

- wrappers can accidentally inherit default behavior instead of delegating to a
  backend's specialized implementation

The repository documents this concern directly in `src/lib.rs`.

That makes the trait design not just an API detail, but an architecture
decision.

## Mental Model

```text
            implementors write this
                    |
                    v
           +-------------------+
           |   ObjectStore     |
           | minimal contract  |
           +-------------------+
                    |
                    v
           +-------------------+
           |  ObjectStoreExt   |
           | convenience layer |
           +-------------------+
                    |
                    v
              application code
```

## What The Trait Promises

At a high level the trait says:

- writes are atomic at the object level
- reads are asynchronous and can be streamed
- listing is prefix-based
- multipart upload is a first-class operation
- copy and rename are object-store operations, not filesystem handle operations

The abstraction is intentionally stateless.

That is why you do not see open file handles or seek cursors in the core trait.

## Important Methods And Their Meaning

### `put_opts`

This is the authoritative write API.

Why it exists:

- allows overwrite/create/update semantics
- allows tags, attributes, and extensions
- keeps the actual backend contract explicit

### `put_multipart_opts`

This is how large writes are represented.

Why it exists:

- object stores frequently support large-object upload through multipart/block
  APIs
- this does not fit cleanly into a single "write all bytes now" method

### `get_opts`

This is the authoritative read API.

Why it exists:

- range reads
- conditional reads
- version selection
- head-only behavior

### `delete_stream`

This is more important than it looks.

Why it is stream-based:

- some backends support bulk delete
- some backends only delete one object at a time
- stream input allows large or dynamically produced delete sets

### `list` and `list_with_delimiter`

These represent the object-store view of listing:

- recursive prefix listing
- delimiter-based grouping

This is not directory traversal in a filesystem sense.

## Why The Option Structs Matter

The API uses explicit request structs:

- `GetOptions`
- `PutOptions`
- `PutMultipartOptions`
- `CopyOptions`
- `RenameOptions`

This is good architecture for three reasons:

1. avoids trait method explosion
2. allows backwards-compatible growth
3. makes provider-specific extensions possible without changing the core shape

## Error Model As Part Of The Contract

The public `Error` enum is part of the trait design, not just support code.

It tells implementors which semantic failures should be normalized:

- missing object
- object already exists
- failed precondition
- not modified
- not implemented
- permission/authentication failures

That means applications can reason in storage terms instead of raw HTTP or OS
error terms.

## Files To Read Next

After understanding the trait surface, move to:

- `src/path/mod.rs`
- `src/payload.rs`
- `src/util.rs`

The trait tells you what the crate promises.
Those files tell you how the shared semantic model is represented.

