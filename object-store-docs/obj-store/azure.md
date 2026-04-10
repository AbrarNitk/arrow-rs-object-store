# Azure Blob Storage Notes

This file is a capability-oriented note for Azure Blob Storage.

The focus is on object semantics that matter for the Rust `object_store` crate:

- blob types
- object APIs
- versions and ETags
- optimistic and pessimistic concurrency
- upload structure
- request and throughput limits

## 1. Core Mental Model

Azure Blob Storage stores blobs inside containers inside storage accounts.

Azure has multiple blob types:

- block blobs
- append blobs
- page blobs

For `object_store` crate purposes, block blobs are the most important, because
they are the natural fit for normal object upload/download semantics.

## 2. Main Blob APIs

Azure's blob REST surface is broad.

The core object operations include:

- `Put Blob`
- `Get Blob`
- `Get Blob Properties`
- `Set Blob Properties`
- `Get Blob Metadata`
- `Set Blob Metadata`
- `Get Blob Tags`
- `Set Blob Tags`
- `Find Blobs By Tags`
- `Delete Blob`
- `Undelete Blob`
- `Snapshot Blob`
- `Copy Blob`
- `Copy Blob From URL`
- `Set Blob Tier`
- `Blob Batch`

For block blobs specifically:

- `Put Block`
- `Put Block From URL`
- `Put Block List`
- `Get Block List`

Azure also supports:

- `Lease Blob`

which is important because it gives an explicit lock-like mechanism.

## 3. Blob Types And Why They Matter

### Block blobs

Best fit for:

- ordinary file/object storage
- parallel upload through blocks
- commit-once finalization via block list

### Append blobs

Best fit for:

- append-heavy workloads
- log-style writes

### Page blobs

Best fit for:

- random page updates
- VM-disk-like usage

From the repository perspective, the Azure backend maps multipart-like writes to
block blob operations.

## 4. Consistency And Concurrency

Azure Blob Storage is designed around:

- strong consistency for reads after writes
- snapshot isolation for reads concurrent with writes

Azure's concurrency model is more explicitly documented than some providers.

It supports:

- optimistic concurrency
- pessimistic concurrency
- last writer wins

### Optimistic concurrency

Azure updates the blob's ETag on each write.

You can then:

- read the blob and capture its ETag
- write back using `If-Match`
- fail with `412 Precondition Failed` if another writer changed the blob

This is a strong fit for the Rust crate's `PutMode::Update` and `GetOptions`
style semantics.

### Pessimistic concurrency

Azure also supports leases.

That means a client can acquire a lease and require the lease ID for
write/delete operations against the locked blob.

This is something S3 does not expose in the same direct way.

## 5. Versioning

Azure supports blob versioning.

Important points:

- versioning automatically maintains previous versions of a blob
- every write operation creates a new version when versioning is enabled
- versions have version IDs
- the current version and previous versions are distinct
- deleting a blob turns the current version into a previous version
- existing versions remain accessible even if versioning is later disabled
- blob versions are immutable

Operational note:

- Microsoft recommends keeping fewer than 1,000 versions per blob because very
  large version counts can increase list latency

This is one of the most important provider facts to remember when reasoning
about cost and list performance.

## 6. ETags In Azure

Azure uses ETags as the main optimistic concurrency identity.

Key points:

- every stored object has an ETag-like identifier for conditional logic
- the ETag changes on each write
- `If-Match` can protect updates from overwriting a changed blob

In practice, Azure ETag usage is very friendly to object-store abstractions.

## 7. Upload Model

Azure's block blob upload model is conceptually multipart-like:

```text
Put Block
Put Block
Put Block
   |
   v
Put Block List
   |
   v
final blob becomes visible as committed block list
```

This maps nicely onto the Rust crate's multipart abstraction.

Important scale limits for standard blob storage:

- up to 50,000 blocks in a block blob
- block size up to 4,000 MiB
- maximum block blob size about 190.7 TiB
- a single `Put Blob` write can be up to 5,000 MiB on recent service versions

Append blob and page blob limits differ.

## 8. Batch And Copy Operations

Azure has a few capabilities that are easy to forget but matter for system
design:

- `Blob Batch` for sending multiple API calls in one request
- synchronous `Copy Blob From URL` for source blobs up to 256 MiB
- `Find Blobs By Tags` for server-side search by index tags

This makes Azure richer than a minimal "put/get/list" mental model.

## 9. Performance And Throttling Guidance

Azure publishes scalability targets and explicit guidance on overload behavior.

Important notes:

- when partitions are overloaded, Azure can return `503 Server Busy` or
  `500 Operation Timeout`
- Microsoft recommends exponential backoff for retries
- sudden spikes should be avoided and traffic should be distributed across
  partitions

Some concrete published targets for standard blob storage include:

- single page blob target request rate: up to 500 requests per second
- block blobs scale primarily with storage account ingress/egress limits

For block blob performance, Azure also notes that larger blobs or blocks,
especially above `256 KiB`, help take advantage of high-throughput block blob
optimizations.

## 10. What Matters Most For The Rust `object_store` Crate

From the crate's perspective, Azure is interesting because it provides:

- strong consistency
- ETag-based optimistic concurrency
- lease-based pessimistic concurrency
- block-based multipart-style upload
- batch delete capability
- versioning
- tags and metadata APIs
- storage tier controls

That is why the Azure backend can support a fairly rich subset of the unified
trait.

## 11. Watchouts

- Azure is not just one object type; block, append, and page blobs differ
- versioning can increase both storage cost and list latency
- append blobs are the exception for concurrent-write friendliness
- leases are powerful but impose extra protocol/state management
- partition overload can surface as `503` or `500`, so retry policy matters

## 12. Background Reading

Useful references:

- Blob REST operations:
  `https://learn.microsoft.com/en-us/rest/api/storageservices/operations-on-blobs`
- Blob versioning:
  `https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview`
- Concurrency management:
  `https://learn.microsoft.com/en-us/azure/storage/blobs/concurrency-manage`
- Scalability targets:
  `https://learn.microsoft.com/en-us/azure/storage/blobs/scalability-targets`

Background paper:

- "Windows Azure Storage: A Highly Available Cloud Storage Service with Strong Consistency"
  (SOSP 2011)

