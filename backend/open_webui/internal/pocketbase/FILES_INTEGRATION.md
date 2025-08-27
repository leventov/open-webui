# Files Integration with PocketBase

This document details how to migrate and operate file storage using PocketBase (PB) file fields, replacing the existing `files` SQL table and local path handling.

## Goals
- Store file binary content in PB’s file storage via file fields.
- Keep metadata (`hash`, `data`, `meta`, `access_control`) on the PB record.
- Provide simple Python abstractions for upload/download/URL generation.

## Collection: `files`
- Fields
  - `user_id` (text, indexed)
  - `file` (file field, required)
  - `hash` (text)
  - `data` (json)
  - `meta` (json)
  - `access_control` (json)
- Rules
  - Restrict read/write by owner or admins; align with existing behavior.

## Implementation Plan
- [ ] Define `files` collection in JS migrations (fields, indexes, rules).
- [ ] Implement `FilesRepoPB`:
  - `insert_new_file(user_id, form)` via multipart create.
  - `get_file_by_id(id)` mapping to `FileModel`.
  - `get_download_url(id)` and `open_stream(id)` helpers using PB client.
  - `update_file_metadata/hash/data(id, ...)` via partial update.
  - `delete_file_by_id(id)`.
- [ ] Replace SQL file usages in model methods with PB repo calls.
- [ ] Implement a migration script to upload existing files and port metadata.

## Upload & Download Workflows
- Upload (server-mediated): server constructs multipart request; PB returns record with file path/token.
- Download: generate authenticated URL via PB client and redirect, or proxy stream through our server for uniform auth.

## Migration from SQL (one-shot)
- [ ] Read file blobs from current storage using `path`.
- [ ] Upload to PB; set metadata fields.
- [ ] Verify checksums using `hash` (if present).
- [ ] Update cross-references if any.

## Testing
- [ ] Integration tests for upload/download and metadata consistency.
- [ ] Negative tests for permissions and large files.