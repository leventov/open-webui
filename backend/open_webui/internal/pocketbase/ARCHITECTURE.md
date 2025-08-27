# PocketBase Integration Architecture Overview

## Goals
- Add PocketBase (PB) as an optional storage backend for Open WebUI data models while retaining existing SQLAlchemy (SQLite/PostgreSQL) support.
- Avoid attempting an SQLAlchemy dialect for PB; instead introduce clear repository interfaces with pluggable implementations.
- Leverage PB features: records CRUD, relations, file storage, realtime (SSE), JS migrations and hooks.
- Keep vector/pgvector stack on PostgreSQL; do not port vector retrieval to PB.

## High-Level Design
- Introduce repository interfaces per domain model group (e.g., users, chats, messages, tags, files, groups, folders, models).
  - Default implementation: existing SQLAlchemy-based repositories (thin wrappers around current table classes).
  - PB implementation: new adapters using the PocketBase Python client.
- Introduce a configuration flag/env to select backend per model group (or globally), allowing staged rollout.
- Replace JSON-based tag filtering with PB-native relations and/or PB JS custom route for complex filters.
- Use PB JS migrations to define collections, indexes, and access rules. Store migrations in-repo and apply at bootstrap.
- Use PB SSE subscriptions to surface model change events to the existing Socket.IO layer.
- Replace `files` table with PB file fields; support uploads/downloads via PB API.

## Components
- Repositories Layer
  - Interfaces: `IUsersRepo`, `IChatsRepo`, `IMessagesRepo`, `ITagsRepo`, `IFilesRepo`, `IGroupsRepo`, `IFoldersRepo`, `IModelsRepo`.
  - SQLAlchemy implementations: thin wrappers that call the existing model classes in `backend/open_webui/models`.
  - PB implementations: translate repository methods to PB client operations and filters.
- PocketBase Client
  - Use `pocketbase` Python client (`vaphes/pocketbase`) with an authenticated admin or service user session.
  - Encapsulate client in a connection module with retry and token refresh.
- PB Schema & Migrations
  - JS migrations under `backend/open_webui/internal/pocketbase/migrations/` defining collections, fields, indexes, and rules.
  - Hooks under `backend/open_webui/internal/pocketbase/hooks/` for custom routes (e.g., tag filters) and server-side enrichments.
- Realtime Bridge
  - A Python service subscribes to PB SSE per relevant collection and forwards normalized events into the existing Socket.IO namespaces/rooms.
- Files Integration
  - `files` collection with file field(s), metadata fields, and access controls.
  - Upload via PB multipart endpoints; store returned file token/path in domain models when needed.

## Data Modeling Notes
- IDs and timestamps
  - Prefer PB’s `id`, `created`, `updated`. Maintain compatibility by mapping to existing response shapes in repositories.
- Tags
  - Replace `chat.meta.tags` JSON array with a PB relation field from `chat` to `tag` (many-to-many). For backward compatibility and migration, keep a temporary mirror.
  - For advanced queries (e.g., “chat has all of tags [a,b,c]”), implement a PB JS route to perform server-side filtering efficiently if PB filter syntax becomes too complex.
- Unique constraints
  - Composite `(id,user_id)` replaced by a single PB `id` (e.g., `"{user_id}:{normalized_tag}"`) and an indexed `user_id` field.
  - Enforce additional uniqueness in application code where PB can’t express it directly.

## Request Lifecycle & Consistency
- Current code commits frequently; not transaction-heavy. PB lacks multi-op transactions; repository methods should:
  - Keep operations idempotent.
  - Group related changes in single repo calls when atomicity matters (best-effort with retries).
- Global post-request commit in FastAPI is retained for SQL backends; PB adapters perform immediate HTTP operations.

## Non-Goals
- Porting vector store or advanced SQL queries (pgcrypto, lateral joins) to PB.
- Emulating an SQLAlchemy dialect over HTTP.

## Configuration & Bootstrap
- Env flags to select backend (e.g., `STORAGE_BACKEND=sqlalchemy|pocketbase`).
- On PB selection, bootstrap step runs PB JS migrations via the PB Admin API.
- Health checks ensure PB availability; fail-fast if schema missing.

## Risks & Mitigations
- Performance regressions on complex filters: mitigate with relations, indexes, and custom PB routes.
- Consistency on multi-step updates: consolidate into single repo methods; implement retries.
- Migration complexity: staged rollout per model group with dual-write or backfill scripts if needed.

## Rollout Strategy (summary)
1. Stand up PB, run migrations, and wire the PB client.
2. Implement PB repos for users/chats/messages first (without tags filter changes), feature-flagged.
3. Remodel tags to relations and enable advanced filtering via PB route.
4. Move files to PB fields/APIs.
5. Add realtime bridge subscriptions.
6. Cutover remaining models; deprecate SQL paths only if desired.