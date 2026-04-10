# Object Store Provider Notes

This folder is for provider capability notes.

The goal here is different from the source-code reading track:

- `src-code-read/` explains how this repository is implemented
- `obj-store/` explains what the backing storage systems themselves provide

Use these notes when you want to answer questions like:

- what operations does S3 actually provide?
- what is Azure Blob Storage's version model?
- when does an ETag mean content hash, and when does it not?
- which service gives strong consistency?
- which service has explicit request-rate guidance?
- how do conditional updates work?

## Files

- `crate-api-mapping.md`
- `s3.md`
- `azure.md`
- `gcs.md`

## Reading Tip

Read these docs together with:

- `src/aws/mod.rs`
- `src/azure/mod.rs`
- `src/gcp/mod.rs`

That way you can compare:

- provider capability
- crate abstraction
- implementation mapping
