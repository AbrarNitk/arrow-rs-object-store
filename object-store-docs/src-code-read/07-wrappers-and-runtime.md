# 07. Wrappers And Runtime Construction

This module explains the last major architectural layer:

- wrappers around stores
- URL-driven store creation
- store registry and resolution

Primary files:

- `src/prefix.rs`
- `src/limit.rs`
- `src/throttle.rs`
- `src/parse.rs`
- `src/registry.rs`

## Why This Layer Exists

The core trait and the backends are not enough for real applications.

Applications also need:

- path scoping
- concurrency control
- latency simulation
- runtime construction from config
- routing from URL to correct store instance

This module is where the crate becomes an operational system component, not
just a trait library.

## 1. Wrapper Pattern

The wrappers are classic decorators over `ObjectStore`.

### Shape

```text
application
   |
   v
wrapper store
   |
   v
inner store
   |
   v
backend implementation
```

Each wrapper adds behavior without changing the underlying backend.

## 2. `PrefixStore`

File:

- `src/prefix.rs`

What it does:

- prepends a fixed prefix to every path
- strips that prefix back out on results

Why it is useful:

- logical sub-store views
- multi-tenant or per-dataset scoping
- cleaner application code

Why it is architecturally nice:

- proves that `Path` is a first-class semantic object
- proves the trait is composable

## 3. `LimitStore`

File:

- `src/limit.rs`

What it does:

- limits concurrent requests with a semaphore

Why it matters:

- protects downstream systems
- prevents runaway concurrency
- makes resource usage more predictable

Important subtlety:

For streamed/list operations, the permit stays alive for the lifetime of the
stream wrapper.

That is exactly the right behavior and worth noticing while reading the code.

## 4. `ThrottledStore`

File:

- `src/throttle.rs`

What it does:

- injects deterministic delays into operations

Why it exists:

- testing
- benchmarking
- simulating remote latency

This wrapper shows that the crate is designed to be testable and observable as
well as functional.

## 5. URL Parsing

File:

- `src/parse.rs`

This file is the bridge from storage URLs to concrete backend instances.

It contains:

- `ObjectStoreScheme`
- `parse_url`
- `parse_url_opts`

Why this matters:

- applications often discover storage through configuration or object URLs
- they do not want to hard-code a backend constructor everywhere

This is what gives the crate a practical runtime portability story.

## 6. Registry-Based Resolution

File:

- `src/registry.rs`

`ObjectStoreRegistry` and `DefaultObjectStoreRegistry` provide:

- store registration
- longest-prefix URL matching
- lazy store creation
- store reuse

### Mental model

```text
URL
 |
 v
registry lookup by:
  - scheme
  - authority
  - longest matching path prefix
 |
 v
(store, relative path)
```

This is especially useful for systems that access many datasets or buckets via
URLs.

## 7. Why This Layer Completes The Architecture

Without this layer, the crate would still be useful, but it would remain a
lower-level storage library.

With this layer, it becomes much more usable for:

- query engines
- data processing systems
- metadata services
- configurable CLI tools

## Final Architecture Summary

```text
1. public trait layer
   - ObjectStore
   - ObjectStoreExt

2. semantic/shared logic layer
   - path
   - payload
   - util
   - upload
   - list
   - client

3. backend layer
   - memory
   - local
   - aws/gcp/azure/http

4. composition/runtime layer
   - prefix
   - limit
   - throttle
   - parse
   - registry
```

That four-layer model is the simplest faithful summary of the repository.

## What To Do After This

Once you finish this reading track, you should be ready to do three things:

1. write architecture notes from memory
2. compare backends with confidence
3. start turning the material into blog and video outlines later

