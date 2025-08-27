# PocketBase Schema and JS Migrations Plan

This document defines the PocketBase (PB) collections, fields, indexes, and permissions to support Open WebUI models, and how to manage them via JS migrations.

## Structure
- Migrations directory: `backend/open_webui/internal/pocketbase/migrations/`
- Hooks directory: `backend/open_webui/internal/pocketbase/hooks/`
- Each migration file: versioned JS with `migrate((db) => { ... }, (db) => { ... })` up/down.
- A bootstrap step will run these migrations via PB Admin API or CLI.

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
- Rules
  - Read/Write rules to match existing access model (start permissive; tighten later).

### chats
- Fields
  - `user_id` (text)
  - `title` (text)
  - `chat` (json)
  - `meta` (json)
  - `share_id` (text, unique?)
  - `archived` (bool)
  - `pinned` (bool)
  - `folder_id` (text)
  - `tags` (relation list to `tags` collection)  ← replaces JSON tag array
- Indexes
  - `user_id`
  - `archived`, `pinned`
  - `updated` (PB-managed) with `user_id` composite order via query
- Rules
  - Read allowed to owner; shared chats via `share_id` custom logic if needed.

### messages
- Fields
  - `user_id` (text)
  - `channel_id` (text)
  - `parent_id` (text)
  - `content` (text)
  - `data` (json)
  - `meta` (json)
- Indexes: `channel_id`, `parent_id`, `user_id`, `created`

### message_reactions
- Fields
  - `user_id` (text)
  - `message_id` (relation to `messages` or text)
  - `name` (text)
- Indexes: `message_id`, `user_id`

### tags
- Fields
  - `id_comp` (text, unique) ← normalized composite key: `${user_id}:${normalized_name}` (primary unique)
  - `user_id` (text, indexed)
  - `name` (text)
  - `meta` (json)
- Notes
  - PB primary `id` remains; we ensure uniqueness on `id_comp` and use it as logical key.

### files
- Fields
  - `user_id` (text)
  - `file` (file field, required)
  - `hash` (text)
  - `data` (json)
  - `meta` (json)
  - `access_control` (json)
- Indexes: `user_id`

### functions, tools, groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories
- Translate 1:1 fields to PB fields (text/json/bool/date as appropriate).
- Add minimal indexes that correspond to frequent lookups.

## Migrations
- Create collections and fields with precise types and options (required, unique, multiple for relations).
- Add indexes via `@request.db.collection.update()` or schema helper where supported.
- Set collection rules (list/view/create/update/delete expressions) aligning with current access; start permissive for admin bootstrap.

## Hooks and Custom Routes
- Place custom route(s) in `hooks/advanced_filters.js`:
  - Route: `GET /api/openwebui/chats/byTags` with params `userId`, `tags` (array), `mode=all|any`.
  - Implement efficient filtering: PB query + server-side verification (fallback) or SQL via `expand` where possible.
- Optionally add hooks for cascading updates (e.g., maintain mirrored tag arrays for backward compatibility during transition).

## Bootstrap & Ops
- On startup in PB mode:
  - Ensure admin credentials configured.
  - Run JS migrations if pending (invoke PB CLI or Admin API).
  - Verify required collections exist with expected fields.

## Data Migration Strategy (from SQL)
- Export per-collection data from SQL and import into PB via Admin API.
- For `tags`:
  - Create `id_comp = "${user_id}:${normalized(name)}"`.
  - Build relations from `chats.tags` using existing JSON array or via tag names.
- For `files`:
  - Upload file content to PB and update metadata.

## Security & Access
- Start with admin-only rules; introduce per-collection rules after repositories are validated.
- Auth: Either use PB auth for end-users or continue issuing JWT via Open WebUI and access PB as service with internal authorization checks.

## Testing
- Provide a seed migration for local dev.
- Write integration tests that spin up PB, run migrations, and assert schema integrity.