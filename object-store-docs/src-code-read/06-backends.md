# 06. Backend Architecture

This module explains how the concrete backends fit into the overall design.

Primary files:

- `src/memory.rs`
- `src/local.rs`
- `src/aws/mod.rs`
- `src/aws/client.rs`
- `src/gcp/mod.rs`
- `src/gcp/client.rs`
- `src/azure/mod.rs`
- `src/azure/client.rs`
- `src/http/mod.rs`

## First Big Split

The backends fall into two broad groups:

1. direct implementations
2. provider façade implementations

## 1. Direct Implementations

### `InMemory`

File:

- `src/memory.rs`

This backend implements the object-store contract almost directly on top of
in-process data structures.

Core idea:

```text
Path -> Entry

Entry:
  data
  last_modified
  attributes
  e_tag
```

Why it matters:

- easiest backend to reason about
- best place to learn semantic behavior
- acts like a reference model for trait behavior

### `LocalFileSystem`

File:

- `src/local.rs`

This backend is still fairly direct, but it must translate filesystem behavior
into object-store behavior.

Why it exists separately from cloud client infrastructure:

- local I/O is not HTTP
- it can optimize around file handles
- path mapping rules are very different

## 2. Provider Façade Implementations

These are the cloud backends:

- `AmazonS3`
- `GoogleCloudStorage`
- `MicrosoftAzure`
- `HttpStore`

They all share a pattern:

```text
top-level backend struct
       |
       v
 provider client
       |
       +--> auth/signing
       +--> request building
       +--> response parsing
       +--> provider quirks
```

## 3. Why The Cloud Backends Are Thin At The Top

Look at:

- `src/aws/mod.rs`
- `src/gcp/mod.rs`
- `src/azure/mod.rs`

The backend type usually holds:

- an `Arc<...Client>`

That is deliberate.

The top-level backend module is mostly responsible for:

- implementing `ObjectStore`
- mapping trait methods to client actions
- orchestrating multipart state
- exposing signer or credential-facing helpers

The deep protocol work lives in the provider client module.

That keeps the architecture cleaner.

## 4. S3, GCS, And Azure Share Structure, Not Exact Semantics

The crate does not pretend all providers are identical.

Instead it shares:

- high-level trait shape
- some client infrastructure
- retry logic
- listing abstraction
- GET handling

But it keeps provider specifics where they belong:

- S3 signing and conditional behavior
- GCS generation/version handling
- Azure block upload and SAS behavior

This is the right abstraction boundary.

## 5. Multipart Comparison

This is a useful backend comparison point.

### S3

- explicit multipart session ID
- individual parts
- completion by ordered part list

### GCS

- multipart support through GCS-specific multipart flow
- version/generation semantics differ from S3

### Azure

- parts are blocks
- completion is "put block list"
- abort semantics differ because uploaded blocks naturally expire

### In-memory

- multipart is simulated in memory for semantic compatibility

This is a great example of why the crate needs a high-level multipart
abstraction.

## 6. Why `HttpStore` Is Useful

File:

- `src/http/mod.rs`

`HttpStore` shows the lower-capability boundary of the abstraction.

It proves that:

- the trait is broad enough to support generic HTTP/WebDAV systems
- but not every backend will support every richer capability such as multipart

This is why the public error model includes `NotImplemented` and
`NotSupported`.

## 7. Which Backend To Read First

Recommended order:

1. `src/memory.rs`
2. `src/local.rs`
3. `src/http/mod.rs`
4. one full cloud backend, preferably AWS
5. then compare GCS and Azure

Why this order works:

- semantic clarity first
- adaptation second
- networked/provider complexity last

## Key Architectural Insight

The repo does not have "many unrelated backends".

It has:

- a shared semantic core
- two direct teaching backends
- several provider façades built on shared infrastructure

That is a coherent backend architecture.

## Files To Read Next

Finish with the composition layer:

- `src/prefix.rs`
- `src/limit.rs`
- `src/throttle.rs`
- `src/parse.rs`
- `src/registry.rs`

