# PocketBase Integration Architecture Overview

## Decisions (updated)
- [x] Do NOT emulate `SessionLocal`/`get_db` for PocketBase.
  - Re-implement the existing "table" modules under `backend/open_webui/models/` to call PocketBase adapters internally (duck-typing the SQLAlchemy Session API is not safe nor desirable given the breadth of methods used: `db.get`, `db.add`, `db.query(...).filter_by(...).order_by(...).limit(...)`, `db.commit`, `db.refresh`, etc.).
  - Keep the external Python API surface of these modules stable so the rest of the backend doesn’t change.
- [x] One-time backend migration; no live flags in production.
  - We will not support per-model or per-request switching in production. The architecture assumes a single cutover from SQL (sqlite/pg) to PocketBase; after migration, the system operates PB-only.
- [x] Tags modeling: replace JSON-based `chat.meta.tags` with PB-native relations (`chats.tags -> tags` many-to-many) and provide one PB custom route for “has ALL tags”.
- [x] Migrations/hooks delivery: JS migrations and hooks live in-repo and must be copied into the PocketBase server filesystem (`pb_migrations/` and `pb_hooks/`) or built into a custom PB binary. There is no Admin API to install migrations.
- [x] Uniqueness/indexes: enforce via PB schema (unique indexes supported). No application-level uniqueness shims needed unless expressly noted.
- [x] Transactions: current Open WebUI code has no multi-entity transactions; methods commit individually. PB’s lack of multi-op transactions is acceptable for parity.
  - Reference: commits in model methods (e.g., `backend/open_webui/models/messages.py` lines 118–123) and global HTTP post-request commit middleware:
```1162:1167:backend/open_webui/main.py
@app.middleware("http")
async def commit_session_after_request(request: Request, call_next):
    response = await call_next(request)
    # log.debug("Commit session after request")
    Session.commit()
    return response
```

## High-Level Implementation Strategy
- [ ] Implement PocketBase adapters (per model group) and map to the existing Pydantic models.
- [ ] Update each `backend/open_webui/models/*.py` module to use the PB adapters internally (preserve public method signatures and return types).
- [ ] Replace tag storage with PB relations and update list/filter methods to use relations and the custom route.
- [ ] Deliver PB JS migrations and hooks into `pb_migrations/` and `pb_hooks/` on the PB server; document ops steps.
- [ ] Add a realtime bridge from PB SSE to our Socket.IO layer.
- [ ] Replace `files` SQL table with PB file fields and adjust workflows.

## Components
- Repositories/Adapters Layer
  - Interfaces: `IUsersRepo`, `IChatsRepo`, `IMessagesRepo`, `ITagsRepo`, `IFilesRepo`, `IGroupsRepo`, `IFoldersRepo`, `IModelsRepo`.
  - SQL implementation (for local parity/testing), PB implementation (production).
- PocketBase Client
  - Use `vaphes/pocketbase` Python client with admin/service auth, retries, and pagination helpers.
- PB Schema & Migrations
  - JS migrations under `backend/open_webui/internal/pocketbase/migrations/`.
  - Hooks under `backend/open_webui/internal/pocketbase/hooks/` (custom routes, validations).
- Realtime Bridge
  - Subscribe to PB SSE and normalize to Socket.IO events.
- Files Integration
  - `files` collection with file field(s) and metadata.

## Data Modeling Notes (updated)
- IDs and timestamps
  - Use PB’s `id`, `created`, `updated`. Repositories map to existing `created_at`/`updated_at` fields (epoch) for compatibility.
- Tags (decision)
  - PB relation `chats.tags` replaces JSON array at `chat.meta.tags`.
  - Provide a PB custom route for "has ALL tags" semantics.
- Unique constraints (updated)
  - For `tags`, SQL used composite primary key `(id, user_id)` where `id` is the normalized tag key. In PB, create `id_normalized` and a unique field `id_comp = "{id_normalized}:{user_id}"` plus an indexed `user_id`.
  - PB unique indexes cover additional uniqueness cases.

## Request Lifecycle & Consistency (updated)
- No multi-entity transactions exist in current code; operations commit per-method.
- HTTP middleware post-commit reference: `backend/open_webui/main.py` lines 1162–1167 (see snippet above). Socket handlers do not rely on middleware and the write methods they call commit internally.

## Non-Goals
- Port vector/pgvector or advanced SQL (pgcrypto, lateral joins) to PB.
- Build an SQLAlchemy dialect for PB.

## References
- PB API records: `https://pocketbase.io/docs/api-records/`
- PB realtime: `https://pocketbase.io/docs/api-realtime/`
- PB files: `https://pocketbase.io/docs/files-handling/`
- PB collections: `https://pocketbase.io/docs/api-collections/`
- PB JS migrations: `https://pocketbase.io/docs/js-migrations/`