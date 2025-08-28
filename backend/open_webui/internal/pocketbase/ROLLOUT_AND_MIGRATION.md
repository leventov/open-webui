# Data Migration Execution Plan (SQLAlchemy → PocketBase)

Scope: This document focuses solely on executing a one-shot data migration from SQL-backed Open WebUI to PocketBase (PB). Architecture choices and file storage design are covered in `ARCHITECTURE.md` and `FILES_INTEGRATION.md`.

References:
- PB records API: `https://pocketbase.io/docs/api-records/`
- PB files handling: `https://pocketbase.io/docs/files-handling/`

## Prerequisites
- PB server running with schema and hooks installed (see `SCHEMA_AND_MIGRATIONS.md`).
- Backend PB adapters implemented (see `REPOSITORIES_AND_ADAPTERS.md`).
- Auth model decided (see `AUTH_AND_AUTHORIZATION.md`).
- Communication plan drafted for a one-time re-login after cutover.
- Secret storage plan in place for any PB service credentials used by background jobs (environment variables or an external secret manager).

## Execution Steps
- [ ] Freeze writes on the source (maintenance window): stop API writes or route to a maintenance page.
- [ ] Snapshot SQL DB (backup) and file storage (if local/object storage not reused).
- [ ] Initialize PB client (service/admin) and verify target collections exist.
- [ ] Migrate entities in dependency order with Python batch scripts:
  - users → groups → folders → channels → chats → messages → message_reactions → models → tools → functions → knowledge → notes → prompts → memories → files → tags and relations.
  - For each table:
    - Stream rows using server-side cursors.
    - Transform fields (timestamps, ids) to PB format.
    - Insert via PB client in batches; log errors with row keys.
- [ ] Tags and chat relations:
  - Compute `tags.id_normalized` and `tags.id_comp = "{id_normalized}:{user_id}"`.
  - Build `chats.tags` relations from legacy JSON tags.
- [ ] Files:
  - If storage can be reused (same local dir or common S3 bucket), migrate only metadata (URIs, filenames, hash) per `FILES_INTEGRATION.md`.
  - Otherwise upload binaries to PB, set metadata, and verify `hash` when available.
- [ ] Integrity checks:
  - Random sampling per collection for row count parity and spot field equality.
  - Referential checks for relations (e.g., messages → channels, chats → users, chats.tags → tags).
- [ ] Cutover:
  - Point backend to PB-only config; bring API out of maintenance.
  - In PB-native mode, invalidate existing Open WebUI sessions (JWT cookies) and require users to sign in via PB.
  - Monitor error rates and latencies.
- [ ] Backout plan:
  - If issues arise, restore SQL snapshot and revert backend config; investigate diffs.
  - Communicate a second re-login requirement if switching IdP back to SQL.

## Deliverables
- [ ] Python migration scripts for each collection (idempotent, resumable).
- [ ] Logging/reporting of migrated counts, failures, and verification results.
- [ ] Runbook describing command ordering, env vars, and expected timings.
 - [ ] User communication template: scheduled maintenance window, re-login required, password reset instructions where applicable.