# PocketBase Schema and JS Migrations Plan

This document defines the PocketBase (PB) collections, fields, indexes, and permissions to support Open WebUI models, and how to manage them via JS migrations.

References:
- PB JS migrations: `https://pocketbase.io/docs/js-migrations/`
- PB collections API: `https://pocketbase.io/docs/api-collections/`

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
  - `email` unique, `oauth_sub` unique, optional `username` unique

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
  - `tags` (relation list to `tags` collection)
- Indexes: `user_id`, `archived`, `pinned`, `updated`

### messages
- Fields: `user_id` (text), `channel_id` (text), `parent_id` (text), `content` (text), `data` (json), `meta` (json)
- Indexes: `channel_id`, `parent_id`, `user_id`, `created`

### message_reactions
- Fields: `user_id` (text), `message_id` (relation/text), `name` (text)
- Indexes: `message_id`, `user_id`

### tags
- Fields
  - `id` (PB primary id)
  - `id_normalized` (text) ← normalized tag identifier used as `id` in SQL
  - `id_comp` (text, unique) ← composite logical key: `"{id_normalized}:{user_id}"`
  - `name` (text)
  - `user_id` (text, indexed)
  - `meta` (json)

### files
- Fields: `user_id` (text), `file` (file field), `hash` (text), `data` (json), `meta` (json), `access_control` (json)
- Indexes: `user_id`

### functions, tools, groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories
- Mirror fields from `backend/open_webui/models/*.py` with PB types (text/json/bool/date as appropriate).
- Add indexes for frequent lookups (e.g., `groups` by `user_id`, `folders` by `user_id`, `models` by `user_id` and `is_active`).

## Indexes & Uniqueness
- Use PB’s unique/index support (see PB docs) to enforce constraints (e.g., `users.email`, `users.oauth_sub`, `tags.id_comp`).

## Hooks and Custom Routes (Tags)
- `pb_hooks/tag_filters.pb.js`
  - `GET /api/openwebui/chats/byTags?userId=...&tags=a,b,c&mode=all|any&skip=&limit=`
  - For `mode=all`, compute intersection in hook if needed.

## Bootstrap & Ops
- [ ] Copy/rsync `migrations/` → PB `pb_migrations/`; `hooks/` → PB `pb_hooks/`.
- [ ] Restart PB; verify migrations applied via Admin UI/CLI.
- [ ] Validate collections/fields/indexes exist.

## Data Migration (one-shot)
- [ ] Export SQL data per collection.
- [ ] Import into PB via Admin UI/CLI or API scripts (batch).
- [ ] Recompute `tags.id_comp` as `"{id_normalized}:{user_id}"` and create `chats.tags` relations.
- [ ] Upload files to PB and validate `hash`.

## Testing
- [ ] Seed migration for local dev.
- [ ] Integration tests that spin up PB, assert schema integrity and basic CRUD.