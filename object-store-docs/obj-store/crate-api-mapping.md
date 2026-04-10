# `object_store` Crate API Mapping To Backends

This file is about the Rust crate's behavior, not the provider products in the
abstract.

Use it to answer questions like:

- which `ObjectStore` APIs are implemented natively on each backend?
- which operations are emulated?
- what batch size or concurrency does the crate use?
- where do backend-specific limitations surface?

Primary code references:

- `src/lib.rs`
- `src/aws/mod.rs`
- `src/azure/mod.rs`
- `src/gcp/mod.rs`
- `src/http/mod.rs`
- `src/local.rs`
- `src/memory.rs`

## 1. High-Level View

```text
ObjectStore method
    |
    +--> native provider API
    |
    +--> provider-specific emulation
    |
    +--> crate default behavior
```

The backend behavior differs most in:

- delete batching
- conditional writes
- multipart uploads
- list offset behavior
- copy/create semantics
- signed URL support

## 2. Bulk Delete Support

This is one of the clearest backend differences.

### Native bulk delete

- AWS (S3):
  - uses `DeleteObjects`
  - batches up to `1000` object paths per request
  - the crate buffers up to `20` delete batches concurrently

- Azure:
  - uses Blob Batch
  - batches up to `256` object paths per request
  - the crate buffers up to `20` delete batches concurrently

### Concurrent individual delete

- GCS:
  - no batch delete in this backend path
  - performs individual delete requests
  - up to `10` concurrent delete operations

- HTTP:
  - performs individual delete requests
  - up to `10` concurrent delete operations

- Local filesystem:
  - performs individual file deletions
  - up to `10` concurrent delete operations using blocking-task dispatch

### Sequential individual delete

- In-memory:
  - deletes by removing entries from the map
  - processed sequentially as the stream is consumed

## 3. `put_opts` Support By Backend

`put_opts` is the most important write API because it carries `PutMode`.

### AWS (S3)

- `Overwrite`: supported
- `Create`: supported only when conditional put support is enabled in the S3
  backend configuration
- `Update`: supported only when conditional put support is enabled, and requires
  an ETag

Additional notes:

- the backend explicitly handles some S3-compatible conditional behavior
  differences
- concurrent `If-Match` updates may retry conflict responses

### Azure

- `Overwrite`: supported
- `Create`: supported through provider logic
- `Update`: supported through provider logic

Additional notes:

- Azure is one of the stronger backends for conditional semantics
- attributes and tags are part of the backend integration

### GCS

- `Overwrite`: supported
- `Create`: supported through provider logic
- `Update`: supported through provider logic

Additional notes:

- GCS is a natural fit for conditional writes because of generation semantics

### HTTP

- `Overwrite`: supported
- `Create`: not implemented
- `Update`: not implemented

Additional notes:

- the backend explicitly rejects `PutMode` other than `Overwrite`

### Local filesystem

- `Overwrite`: supported
- `Create`: supported
- `Update`: not implemented

Additional notes:

- object attributes are not supported in this backend
- writes use staged upload and rename/link-based commit behavior

### In-memory

- `Overwrite`: supported
- `Create`: supported
- `Update`: supported

Additional notes:

- this is the best backend to read for semantic clarity

## 4. Multipart Upload Support

### Supported

- AWS (S3)
- Azure
- GCS
- Local filesystem
- In-memory

### Not supported

- HTTP

## 5. Multipart Model By Backend

### AWS (S3)

- real provider multipart upload
- explicit upload ID
- part uploads collected and ordered before completion

### Azure

- mapped to block blob APIs
- `put_part` maps to `Put Block`
- completion maps to `Put Block List`
- abort has no real provider-side cleanup call in this backend model

### GCS

- uses GCS multipart initiation, part upload, completion, and cleanup flow

### Local filesystem

- multipart is implemented through staged local file behavior
- multipart attributes are not supported

### In-memory

- multipart is simulated in memory

### HTTP

- `put_multipart_opts` returns `NotImplemented`

## 6. Listing Support

### `list`

Supported by all current backends in the crate.

### `list_with_delimiter`

Supported by:

- AWS
- Azure
- GCS
- HTTP
- Local filesystem
- In-memory

### `list_with_offset`

Important backend-specific notes:

- AWS:
  - normally uses native provider support
  - for S3 Express, the backend falls back to client-side filtering because
    `start-after` is not supported there

- Azure:
  - normally uses native provider support
  - for Azurite emulator, the backend falls back to client-side filtering

- GCS:
  - uses backend list-offset support

- Local filesystem:
  - implemented locally in the filesystem walker logic

- In-memory:
  - inherited from listing semantics in map order

- HTTP:
  - no dedicated backend override for offset; behavior comes from the trait
    shape and backend listing capabilities

## 7. Copy Support

### Native or provider-backed copy

- AWS
- Azure
- GCS
- HTTP

These all implement `copy_opts` in backend-specific ways.

### Local filesystem

- implements `copy_opts`
- has special filesystem constraints
- repository docs note that cross-filesystem copy is not supported because the
  implementation uses hard links

### In-memory

- implements copy directly by duplicating entry content in the map

## 8. Rename Support

Important subtlety:

- `rename_opts` has a default trait implementation
- default behavior is `copy` then `delete`

This means:

- not every backend has its own native rename implementation
- rename semantics may not be atomic at the "copy and delete source together"
  level

That is a crate-level semantic caveat worth remembering.

## 9. Signed URL Support

Signed URL support is not part of `ObjectStore`.
It is provided by the separate `Signer` trait for cloud backends.

### Supported

- AWS
- Azure
- GCS

### Not part of this backend layer

- HTTP
- Local
- In-memory

## 10. Attributes And Tags

This is another area where support differs.

### Richer support

- AWS
- Azure
- GCS

These backends map attributes and, where supported, tags into provider request
headers or request bodies.

### Limited or missing support

- HTTP:
  - supports attributes on put in a limited HTTP-oriented way
  - no multipart support

- Local filesystem:
  - `put_opts` with attributes is not implemented
  - `put_multipart_opts` with attributes is not implemented

- In-memory:
  - stores attributes in-memory with the object entry

## 11. Backend Capability Snapshot

```text
Method / Backend     S3   Azure   GCS   HTTP   Local   Memory
-------------------------------------------------------------
put overwrite         Y      Y      Y     Y      Y        Y
put create            C      Y      Y     N      Y        Y
put update            C      Y      Y     N      N        Y
multipart             Y      Y      Y     N      Y        Y
bulk delete           Y      Y      N     N      N        N
list delimiter        Y      Y      Y     Y      Y        Y
signed URL            Y      Y      Y     N      N        N

Y = supported
N = not supported
C = conditional/config-dependent in this backend implementation
```

## 12. Best Way To Use This Doc

Read this file together with:

- `obj-store/s3.md`
- `obj-store/azure.md`
- `obj-store/gcs.md`

That gives you both sides:

- what the provider offers
- what this crate actually maps and exposes

