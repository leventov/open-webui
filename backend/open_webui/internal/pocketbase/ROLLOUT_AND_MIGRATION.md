# Migration Plan (One-Shot)

This document outlines a one-shot migration from SQLAlchemy storage to PocketBase (PB) for Open WebUI using Python batch scripts. It complements the architecture and schema docs.

References:
- PB records API: `https://pocketbase.io/docs/api-records/`
- PB files handling: `https://pocketbase.io/docs/files-handling/`

## Preparation
- [ ] Stand up PB; configure admin/service credentials.
- [ ] Copy JS migrations to PB `pb_migrations/`; copy hooks to `pb_hooks/`.
- [ ] Restart PB and verify schema via Admin UI/CLI.

## Adapter Implementation
- [ ] Implement PB adapters for all domain models.
- [ ] Update `backend/open_webui/models/*.py` internals to call PB adapters; keep public signatures consistent.
- [ ] Implement realtime bridge and files integration.

## Data Export & Import (Python batch)
- [ ] Write Python batch scripts that iterate SQL rows via server-side cursors and insert into PB using the Python client (avoid intermediate CSV/NDJSON when possible).
- [ ] For tags: compute `id_comp = "{id_normalized}:{user_id}"` and create `chats.tags` relations.
- [ ] For files: if storage can be reused (same local dir or S3), migrate metadata only; otherwise upload binaries and set metadata; verify `hash` where present.

## Validation
- [ ] Contract tests targeting PB adapters.
- [ ] End-to-end tests: CRUD, permissions, pagination, sorting.
- [ ] Performance spot-checks for hot paths; add indexes if needed.

## Cutover
- [ ] Point backend to PB-only (remove SQL connection in production config).
- [ ] Monitor errors/latency; address issues.

## Backout Plan
- [ ] Keep a SQL snapshot taken just before cutover.
- [ ] If needed, revert backend to SQL config and investigate.