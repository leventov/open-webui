# Rollout and Migration Plan

This document outlines the staged rollout from SQLAlchemy storage to PocketBase (PB) for Open WebUI, including data migration and testing.

## Strategy Overview
- Feature-flag PB per model group with environment variables.
- Start with read paths (shadow reads) to validate parity, then enable writes.
- Keep vector store and advanced SQL modules on PostgreSQL.

## Phases

### Phase 0: Infrastructure
- Stand up PB in dev/staging; configure admin credentials.
- Run JS migrations (`backend/open_webui/internal/pocketbase/migrations/`).
- Verify schema with health checks.

### Phase 1: Repositories and Adapters
- Implement repository interfaces and PB adapters.
- Add configuration for backend selection.
- Write contract tests that run against both SQL and PB.

### Phase 2: Users/Chats/Messages (basic)
- Enable PB for `users`, `chats`, `messages` without tag filtering parity (basic filters only).
- Perform shadow reads: on SQL write, read from PB and compare (log diffs, no user impact).
- Fix discrepancies.

### Phase 3: Tags Remodel
- Migrate `chat.meta.tags` to PB `chats.tags` relation.
- Implement custom PB route for `byTags`.
- Update repository methods to use PB relations and route for complex filters.
- Provide backfill script to populate relations based on existing SQL JSON tags.

### Phase 4: Files
- Migrate `files` to PB files collection.
- Update upload/download flows; implement proxy or signed URL policies.
- Backfill files by uploading from local storage, verifying `hash`.

### Phase 5: Realtime Bridge
- Deploy PB SSE -> Socket.IO bridge.
- Start with `messages` and `chats`; validate client behavior.
- Extend to other collections.

### Phase 6: Remaining Models
- Port groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories.
- Validate filters and pagination parity.

### Phase 7: Cutover
- Switch feature flags to PB in staging, then production.
- Keep SQL read-only for a grace period; monitor metrics.
- Optionally deprecate SQL storage paths.

## Data Migration
- Export data per table; import into PB via Admin API in batches.
- Maintain ID mapping tables if IDs change; prefer PB IDs as new canonical.
- For tags, compute `id_comp` and relations.
- For files, upload content and validate hashes.

## Testing & Validation
- Contract tests across adapters.
- End-to-end regression tests for critical flows (create/update/list/delete, permissions).
- Performance tests for high-traffic endpoints with PB indexes in place.

## Observability
- Dashboards: PB request rate, latency, error rate; bridge event throughput.
- Logs: discrepancies between SQL and PB during shadow mode.

## Backout Plan
- If PB instability occurs, flip feature flags back to SQL.
- Keep data diff logs to reconcile on retry.

## Risks
- Performance differences due to loss of DB-level indexes (mitigate with PB indexes and relations).
- Consistency during dual-write periods (ensure idempotency and retry).
- Schema drift between PB and docs (enforce with migrations at boot).