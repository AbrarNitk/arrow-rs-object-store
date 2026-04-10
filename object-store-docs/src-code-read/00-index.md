# `object_store` Source-Code Reading Track

This folder is the code-first reading path for understanding the repository.

This is not the blog track and not the video track. It is the "read the code,
understand the architecture, and know why the modules exist" track.

Use it together with the master note in `../docs-ai.md`.

## How To Use This Track

Read the modules in order.

Each module tries to answer four questions:

- what part of the crate this module explains
- why that layer exists
- which files to read in the repository
- what design choices are worth noticing

## Recommended Order

1. `01-crate-entry-and-shape.md`
2. `02-objectstore-contract.md`
3. `03-data-model.md`
4. `04-read-path.md`
5. `05-write-path-and-multipart.md`
6. `06-backends.md`
7. `07-wrappers-and-runtime.md`

## High-Level Map

```text
                    +---------------------------+
                    |       src/lib.rs          |
                    | public API + trait model  |
                    +-------------+-------------+
                                  |
                 +----------------+----------------+
                 |                                 |
      +----------v----------+           +----------v----------+
      |   shared semantics  |           |   shared transport  |
      | path, payload, util |           | client, retry, list |
      +----------+----------+           +----------+----------+
                 |                                 |
         +-------+--------+               +--------+-------+
         |                |               |                |
 +-------v------+ +-------v------+ +------v------+ +------v------+
 |   memory     | |    local     | |     aws     | | gcp/azure   |
 | direct impl  | | direct impl  | | provider    | | provider    |
 +--------------+ +--------------+ | client impl | | client impl |
                                    +-------------+ +-------------+
                                                  |
                                      +-----------v-----------+
                                      | wrappers + runtime    |
                                      | prefix/limit/parse    |
                                      +-----------------------+
```

## Reading Strategy

Do not start with the cloud backends.

Start with:

- `src/lib.rs`
- `src/path/mod.rs`
- `src/payload.rs`
- `src/util.rs`
- `src/memory.rs`

That gives you the semantic model before network and provider complexity show
up.

## What To Keep Asking While Reading

- Why is this in the public API instead of a helper?
- Why is this a trait, not a struct?
- Why is this split into a low-level and convenience layer?
- Which parts are provider-independent?
- Which parts are S3/GCS/Azure-specific?
- Where is the portability line, and where do provider differences leak
  through?

