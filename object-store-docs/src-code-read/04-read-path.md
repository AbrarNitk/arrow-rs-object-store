# 04. Read Path

This module explains how reads move through the crate.

Primary files:

- `src/lib.rs`
- `src/util.rs`
- `src/client/get.rs`
- `src/list.rs`
- `src/client/list.rs`
- `src/local.rs`

## The Read Path In One Diagram

```text
application
   |
   v
ObjectStoreExt::get / get_range / head
   |
   v
ObjectStore::get_opts
   |
   +--> backend-specific request execution
   |
   +--> shared metadata/range/retry handling
   |
   v
GetResult
   |
   +--> bytes()
   |
   +--> into_stream()
```

## 1. Why Everything Funnels Through `get_opts`

The convenience APIs:

- `get`
- `get_range`
- `head`

all reduce to `get_opts`.

Why that is good:

- there is one authoritative read contract
- conditions and ranges compose naturally
- the backend only needs to implement one main read entry point

This is a common pattern in the repository:

- a single expressive primitive
- multiple ergonomic convenience layers on top

## 2. The Role Of `GetResult`

`GetResult` is more than "here are some bytes".

It gives:

- metadata
- returned range
- attributes
- a payload that can stay streaming

This is important because object-store reads are often large and remote.

The crate avoids forcing eager buffering.

## 3. Shared GET Logic For Cloud Backends

File:

- `src/client/get.rs`

This file is one of the most important internal abstractions.

### What it does

It separates:

- provider-specific request issuing
- provider-neutral response interpretation

`GetClient` defines what a backend-specific client must do.

`GetClientExt` adds the shared logic:

- issue request
- compute effective range
- parse headers into `ObjectMeta`
- parse attributes
- stream response body
- retry body failures when possible

### Why this matters

Without this layer:

- S3, GCS, Azure, and HTTP would each duplicate range parsing
- retries would be inconsistent
- metadata extraction would drift

So this file is a major deduplication and consistency layer.

## 4. Range Semantics

File:

- `src/util.rs`

`GetRange` supports three request modes:

- bounded
- offset
- suffix

This is a good example of the crate mirroring object-store and HTTP semantics
instead of inventing something filesystem-like.

## 5. Vectored Reads

The default implementation of `get_ranges` uses `coalesce_ranges`.

That means:

- nearby requested ranges get merged into a smaller number of fetches
- fetches happen in parallel
- the response is sliced back into the original requested ranges

This is especially important for:

- Parquet readers
- columnar systems
- footer-heavy file formats

This logic belongs in shared infrastructure because it is not tied to any one
provider.

## 6. Listing As Part Of The Read Story

Object-store reading is not just `get`.
It also includes object discovery.

### Two layers

User layer:

- `list`
- `list_with_offset`
- `list_with_delimiter`

Provider pagination layer:

- `PaginatedListStore`
- `ListClient`
- `ListClientExt`

### Why this split is useful

Providers paginate differently, but applications should not have to care.

So the repo uses this shape:

```text
provider list API
   |
   v
list_request(...)
   |
   v
page stream
   |
   v
shared flattening / grouping
   |
   v
ObjectMeta stream / ListResult
```

This is a clean portability boundary.

## 7. Special Case: Local Filesystem Reads

File:

- `src/local.rs`

The local backend can return `GetResultPayload::File`.

Why that exists:

- local files can often be read efficiently without immediately turning them
  into a stream of owned byte chunks
- `bytes()` and `into_stream()` can adapt that file as needed

That is a nice optimization without changing the public contract.

## Key Architectural Insight

The read path is organized around:

- one semantic read request type
- shared range and metadata logic
- backend-specific request execution
- lazy consumption of result bytes

That is the right shape for a portable async object-store library.

## Files To Read Next

After this, read:

- `src/payload.rs`
- `src/upload.rs`
- `src/multipart.rs`

That will explain the write path with the same kind of layering.

