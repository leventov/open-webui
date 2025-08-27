# PocketBase Schema and JS Migrations Plan

This document defines the PocketBase (PB) collections, fields, indexes, and permissions to support Open WebUI models, and how to manage them via JS migrations.

## Structure
- Migrations directory (in this repo): `backend/open_webui/internal/pocketbase/migrations/`
- Hooks directory (in this repo): `backend/open_webui/internal/pocketbase/hooks/`
- Delivery to PB: copy/rsync these to the PocketBase server’s `pb_migrations/` and `pb_hooks/` directories (or bake into a custom PB build). There is no Admin API to install migrations.
- Each migration file: versioned JS with `migrate((db) => { ... }, (db) => { ... })` up/down.

## Collections

### users
- Fields
  - `name` (text, required)
  - `email` (email/text, required, unique)
  - `username` (text, optional, unique? if desired)
  - `role` (text)
  - `profile_image_url` (text)
  - `bio` (text, optional)
  - `gender` (text, optional)
  - `date_of_birth` (date, optional)
  - `info` (json)
  - `settings` (json)
  - `api_key` (text, unique?)
  - `oauth_sub` (text, unique)
- Indexes
  - `email` unique
  - `oauth_sub` unique
  - optional: `username` unique

### chats
- Fields
  - `user_id` (text, indexed)
  - `title` (text)
  - `chat` (json)
  - `meta` (json)
  - `share_id` (text, unique?)
  - `archived` (bool)
  - `pinned` (bool)
  - `folder_id` (text)
  - `tags` (relation list to `tags` collection)  ← JSON tags replaced by relations (decision)
- Indexes
  - `user_id`
  - `archived`, `pinned`
  - `updated` (PB-managed) usable with sort

### messages
- Fields: `user_id` (text), `channel_id` (text), `parent_id` (text), `content` (text), `data` (json), `meta` (json)
- Indexes: `channel_id`, `parent_id`, `user_id`, `created`

### message_reactions
- Fields: `user_id` (text), `message_id` (relation/text), `name` (text)
- Indexes: `message_id`, `user_id`

### tags
- Fields
  - `id` (PB primary id)
  - `id_comp` (text, unique) ← composite logical key: `"{id_normalized}:{user_id}"`
  - `id_normalized` (text) ← normalized tag identifier used as `id` in SQL
  - `name` (text)
  - `user_id` (text, indexed)
  - `meta` (json)
- Rationale
  - SQL used composite PK `(id, user_id)` where `id` is the normalized tag key. PB doesn’t support composite PK, so we keep `id_comp` unique and index `user_id`.

### files
- Fields: `user_id` (text), `file` (file field), `hash` (text), `data` (json), `meta` (json), `access_control` (json)
- Indexes: `user_id`

### Remaining models
- functions, tools, groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories
- Mirror existing fields with appropriate PB types; add indexes for frequent lookups.

## Indexes & Uniqueness
- Use PB’s index/unique support for constraints (no app-side uniqueness shims required).
- Add unique constraints mirroring SQL where applicable (e.g., `users.email`, `users.oauth_sub`, `tags.id_comp`).

## Hooks and Custom Routes (Tags)
- Route file under `hooks/advanced_filters.js`:
  - `GET /api/openwebui/chats/byTags?userId=...&tags=a,b,c&mode=all|any&skip=&limit=`
  - Behavior:
    - Resolve `tags` input to `tags` record IDs.
    - Query `chats` with base `user_id == userId` and relation filter on `tags`.
    - For `mode=all`, intersect results client-side in the hook to enforce “ALL” semantics if native filter falls short.

## Bootstrap & Ops
- [ ] Copy/rsync `migrations/` → PB `pb_migrations/`; `hooks/` → PB `pb_hooks/`.
- [ ] Restart PB; verify migrations applied via logs/Admin UI.
- [ ] Validate collections/fields/indexes exist.

## Data Migration (one-shot)
- [ ] Export SQL data per collection.
- [ ] Import into PB via Admin UI/CLI or API scripts (batch).
- [ ] Recompute `tags.id_comp` as `"{id_normalized}:{user_id}"` and create `chats.tags` relations.
- [ ] Upload files to PB and validate `hash`.

## Testing
- [ ] Seed migration for local dev.
- [ ] Integration tests that spin up PB, assert schema integrity and basic CRUD.