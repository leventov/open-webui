# Migration Plan (One-Shot)

This document outlines a one-shot migration from SQLAlchemy storage to PocketBase (PB) for Open WebUI, along with validation and backout steps. No staged/feature-flag rollout in production.

## Preparation
- [ ] Stand up PB; configure admin/service credentials.
- [ ] Copy JS migrations to PB `pb_migrations/`; copy hooks to `pb_hooks/`.
- [ ] Restart PB and verify schema via Admin UI/CLI.

## Adapter Implementation
- [ ] Implement PB adapters for all domain models (users, chats, messages, tags, files, groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories).
- [ ] Update `backend/open_webui/models/*.py` internals to call PB adapters; keep public signatures consistent.
- [ ] Implement realtime bridge and files integration.

## Data Export & Import
- [ ] Export SQL data per table to NDJSON/CSV.
- [ ] Import into PB via Admin UI/CLI or batch scripts using the Python client.
- [ ] For tags: compute `id_comp = "{id_normalized}:{user_id}"` and create `chats.tags` relations.
- [ ] For files: upload file binaries and set metadata; verify `hash`.

## Validation
- [ ] Contract tests targeting PB adapters.
- [ ] End-to-end tests: CRUD, permissions, pagination, sorting.
- [ ] Performance spot-checks for hot paths; add indexes if needed.

## Cutover
- [ ] Point backend to PB-only (remove SQL connection in production config).
- [ ] Monitor errors/latency; address issues.

## Backout Plan
- [ ] Keep SQL snapshot taken just before cutover.
- [ ] If needed, revert backend to SQL config and investigate.

## Notes
- Vector search and advanced SQL features remain on PostgreSQL (unchanged).
- No per-model feature flags in production.