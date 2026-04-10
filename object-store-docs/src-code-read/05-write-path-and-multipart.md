# 05. Write Path And Multipart

This module explains how writes are modeled.

Primary files:

- `src/lib.rs`
- `src/payload.rs`
- `src/upload.rs`
- `src/multipart.rs`
- `src/memory.rs`
- `src/local.rs`

## The Write Path In One Diagram

```text
application
   |
   +--> put(...)
   |      |
   |      v
   |   put_opts(...)
   |
   +--> put_multipart(...)
          |
          v
   put_multipart_opts(...)
          |
          v
     MultipartUpload
          |
   +------+------+ 
   |             |
 put_part      complete/abort
```

## 1. Atomic Write Semantics

The crate wants object-level atomicity.

That means:

- either the whole object is visible
- or the write fails
- partial writes should not be externally observable

This is a core reason the API is object-store-shaped instead of
stream-handle-shaped.

## 2. `PutMode` Is A Big Deal

File:

- `src/lib.rs`

`PutMode` expresses three write intents:

- overwrite existing object
- create only if absent
- update only if current version/etag matches

This makes the write path suitable for:

- metadata log updates
- compare-and-swap loops
- optimistic concurrency

The write API is not just "store this blob".
It is "store this blob under precise object-state assumptions".

## 3. Why `PutPayload` Is Chunked

File:

- `src/payload.rs`

`PutPayload` is a collection of `Bytes`, not a single contiguous byte array.

Why that design is useful:

- lets producers hand over already-owned chunks
- avoids forced copies
- works naturally with multipart upload and buffered generation

## 4. Why `PutPayloadMut` Exists

Also in `src/payload.rs`.

This builder exists because writers often generate data incrementally.

Instead of repeatedly growing one large `Vec<u8>`, it:

- allocates fixed-size blocks
- flushes full blocks to `Bytes`
- preserves chunk structure until freeze time

This is a performance-aware internal model that still stays ergonomic.

## 5. Multipart Has Two Layers

This is one of the best structural ideas in the repo.

### Layer A: `MultipartUpload`

File:

- `src/upload.rs`

This is the user-facing upload object.

The caller gets:

- `put_part`
- `complete`
- `abort`

### Layer B: `MultipartStore`

File:

- `src/multipart.rs`

This is the low-level backend contract.

The backend exposes:

- `create_multipart`
- `put_part`
- `complete_multipart`
- `abort_multipart`

### Why two layers?

Because the crate wants:

- a portable user API
- a backend-oriented protocol abstraction

The low-level layer is about provider capabilities.
The high-level layer is about ergonomic use.

## 6. `WriteMultipart` Is The Practical Bridge

File:

- `src/upload.rs`

`WriteMultipart` turns multipart upload into something closer to a writer
workflow.

What it does:

- buffers bytes until chunk size is reached
- starts part uploads in parallel
- uses a `JoinSet` to track in-flight tasks
- exposes `wait_for_capacity` for backpressure
- flushes final chunk and completes upload

This is the highest leverage write helper in the crate.

```text
producer bytes
    |
    v
WriteMultipart buffer
    |
    +--> chunk full? --> spawn part upload
    |
    v
wait for in-flight completion
    |
    v
complete multipart session
```

## 7. How The Simpler Backends Clarify The Model

### In-memory backend

File:

- `src/memory.rs`

This backend is great for understanding semantics because it strips away
network complexity.

It shows:

- how `PutMode` is enforced
- how synthetic etags are assigned
- how multipart parts are stored before completion

### Local filesystem backend

File:

- `src/local.rs`

This backend is good for understanding adaptation:

- how local files are forced into object-store semantics
- how atomic behavior is preserved
- how a filesystem implementation differs from a cloud implementation

## Key Architectural Insight

The write path is not one feature.

It is three connected layers:

1. semantic write modes
2. efficient payload ownership model
3. multipart orchestration for large objects

That combination is why the crate is useful for both small metadata writes and
large data-file uploads.

## Files To Read Next

Now move to backend structure:

- `src/memory.rs`
- `src/local.rs`
- `src/aws/mod.rs`
- `src/gcp/mod.rs`
- `src/azure/mod.rs`
- `src/http/mod.rs`

