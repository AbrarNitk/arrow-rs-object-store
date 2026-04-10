# Google Cloud Storage Notes

This file is a capability-oriented note for Google Cloud Storage.

The focus is on the parts that matter when comparing provider semantics and
when understanding the Rust `object_store` abstraction.

## 1. Core Mental Model

Google Cloud Storage stores objects in buckets and exposes:

- object data
- object metadata
- generations
- metagenerations
- optional Object Versioning

One of the most important conceptual differences from S3 and Azure is that GCS
has a strong first-class notion of:

- `generation`
- `metageneration`

These are often more reliable concurrency handles than ETags.

## 2. Main Object APIs

At a high level, the operations you care about are:

- upload object
- download object
- get metadata
- update metadata
- delete object
- list objects
- compose objects
- copy/rewrite objects

GCS exposes both JSON API and XML API behavior, and not every option is
identical across the two.

## 3. Consistency Model

Cloud Storage provides strong global consistency for:

- object read-after-write
- object read-after-metadata-update
- object read-after-delete
- object listing
- bucket listing and key bucket metadata reads

Practical meaning:

- after a successful write, the new object state is immediately readable
- object listing reflects successful creates and deletes
- object operations are atomic for individual objects

Important caveat:

- cached public objects can still appear stale based on `Cache-Control`

This is a provider behavior worth remembering when comparing "storage
consistency" with "HTTP cache consistency".

## 4. Generation And Metageneration

Every Cloud Storage object has:

- `generation`
- `metageneration`

### `generation`

This identifies the version of an object and exists even if bucket-level Object
Versioning is not enabled.

Important points:

- each replacement gets a new unique generation
- a generation value for a specific object version never changes
- generation numbers are unique identifiers, not ordered counters you should
  reason about globally

### `metageneration`

This identifies the metadata version for a given generation.

Important points:

- starts at `1` for each new generation
- increments when metadata for that generation changes
- only makes sense together with generation

This pair is one of GCS's strongest design features for conditional operations.

## 5. Object Versioning

GCS supports bucket-level Object Versioning.

Important points:

- when enabled, replacing or deleting a live object retains a noncurrent version
- noncurrent versions keep the same object name but are uniquely identified by
  generation number
- noncurrent versions are not returned unless explicitly requested
- disabling Object Versioning stops new noncurrent versions from accumulating,
  but existing noncurrent versions remain

Current Google guidance is also worth noting:

- Google recommends soft delete instead of Object Versioning when the goal is
  protection from accidental or malicious deletion

So for GCS, "versioning" and "recovery" are related but not identical topics.

## 6. ETags In GCS

GCS objects have ETags, but they are less portable than many developers assume.

Important points:

- all objects have an ETag
- the same object can have different ETag values depending on whether it is
  requested through the XML API or JSON API
- Google advises that users should generally not make assumptions about ETag
  values beyond the fact that they change when underlying data or metadata
  changes

Additional nuance:

- in some XML API cases, the ETag can equal the MD5 value
- this is not a safe general rule

Practical takeaway:

- prefer generation and metageneration for correctness
- treat ETag as secondary

## 7. Preconditions

GCS has one of the strongest documented precondition models.

Supported immutable properties for preconditions include:

- generation
- metageneration
- ETag
- `Last-Modified` for certain XML API reads

Why this matters:

- safe read-modify-write
- safe retries
- create-if-absent semantics
- source and destination safety for copy/compose workflows

Important practical guidance from Google:

- you should generally prefer generation and metageneration preconditions over
  ETag preconditions

Also important:

- preconditions cannot be used in XML API multipart uploads

## 8. Compose And Multipart-Like Patterns

GCS supports composition of objects.

The compose operation:

- concatenates source objects into a new destination object
- accepts between 1 and 32 source objects
- requires all sources to be in the same bucket
- requires all sources to have the same storage class

Why compose matters:

- it enables parallel upload workflows
- it enables limited append-like behavior
- it creates multipart-like higher-level workflows even outside a classic
  provider-native multipart API

Google also documents XML API multipart uploads separately, so there are
multiple chunked-upload stories in GCS depending on API path.

## 9. Request Rate And Scaling Guidance

GCS publishes explicit request-rate guidance.

For a bucket's initial IO capacity, Google documents approximately:

- 1,000 object write requests per second
- 5,000 object read requests per second

Other important notes:

- buckets auto-scale as load grows
- request rates should ramp up gradually after quiet periods or for new key
  ranges
- repeated writes to the same object name have a smaller limit

This is one of the clearest provider throttling stories and is directly useful
for ingestion-system design.

## 10. What Matters Most For The Rust `object_store` Crate

From the crate's perspective, GCS is especially interesting because of:

- strong global consistency
- generation/metageneration semantics
- explicit precondition support
- object composition
- versioning via noncurrent generations
- rich metadata model

The Rust backend can use these capabilities to offer safe conditional updates
and robust multipart-like flows while still exposing the common trait.

## 11. Watchouts

- generation is the real object identity primitive, not just ETag
- versioning and soft delete are separate concepts
- ETag differs across JSON and XML APIs
- compose is powerful, but it has source-count and same-bucket constraints
- public caching can surface stale reads even when the storage layer itself is
  strongly consistent

## 12. Background Reading

Useful references:

- Cloud Storage consistency:
  `https://cloud.google.com/storage/docs/consistency`
- Object metadata:
  `https://cloud.google.com/storage/docs/metadata`
- Object Versioning:
  `https://cloud.google.com/storage/docs/object-versioning`
- Request preconditions:
  `https://cloud.google.com/storage/docs/request-preconditions`
- Request-rate guidance:
  `https://cloud.google.com/storage/docs/request-rate`
- Composite objects:
  `https://cloud.google.com/storage/docs/composite-objects`

Background systems reading:

- "The Google File System" (SOSP 2003) as historical storage-background reading
  rather than a Cloud Storage product specification

