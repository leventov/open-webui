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

## Python Abstraction
- `FilesRepoPB` methods:
  - `insert_new_file(user_id, form)`:
    - Create multipart form: fields for `user_id`, `hash`, `data`, `meta`, `access_control`, and the file payload for `file`.
    - `client.collection('files').create(formData)`.
    - Map record to `FileModel` (id, user_id, filename from uploaded file, meta, timestamps).
  - `get_file_by_id(id)`:
    - Fetch record; map to `FileModel`.
  - `get_download_url(id)` / `open_stream(id)`:
    - Use PB client helper to build authenticated file URL or stream.
  - `update_file_metadata/hash/data(id, ...)`:
    - Partial update of json/text fields without reuploading the file.
  - `delete_file_by_id(id)`.

## Upload Workflow
1. Client uploads to server endpoint (as today) or directly to PB (optional future path).
2. Server (repo) constructs PB multipart request with file content and metadata.
3. PB returns record containing file name/path and tokenized access.
4. Server returns `FileModelResponse` to client.

## Download Workflow
- Server-side signed/authorized URL creation via PB client; serve redirect or proxy stream (depending on auth model).

## Migration from SQL
- For each SQL file record:
  - Read file from local path/storage.
  - Upload to PB in batches.
  - Copy metadata fields.
  - Update references in other models if necessary.
- Consider checksum verification (`hash`) post-upload.

## Access Control
- Use PB collection rules; for domain-specific access, also keep `access_control` JSON and enforce in repo methods.

## Performance & Limits
- Respect PB file size limits and storage configuration.
- Consider chunked uploads if needed (PB supports standard multipart; very large files may need custom handling).

## Testing
- Integration tests uploading and downloading files, validating metadata preservation.
- Negative tests: permissions, missing files, large payload handling.