# Custom Routes and Hooks in PocketBase

This document proposes JavaScript routes and hooks to cover advanced behaviors not easily expressed with PocketBase filters, notably tag filtering.

## Directories
- Migrations: `backend/open_webui/internal/pocketbase/migrations/`
- Hooks: `backend/open_webui/internal/pocketbase/hooks/`

## Tag Filtering Route
- Route: `GET /api/openwebui/chats/byTags`
- Query params:
  - `userId` (string, required)
  - `tags` (comma-separated string of normalized tag ids or names)
  - `mode` (`all`|`any`, default `all`)
  - `skip`, `limit` (optional pagination)
- Behavior:
  - Resolve tag identifiers to tag record IDs.
  - Query `chats` collection with base filter `user_id == userId`.
  - If using relations: filter by `tags` relation (PB supports `tags.id ?= ...` style). For `all` semantics, either intersect client-side or perform iterative queries and intersect IDs.
  - Return `{ items, total }` with normalized payload (matching `ChatResponse`).

## Additional Hooks
- Optional: maintain mirrored `meta.tags` on `chats` create/update for temporary compatibility.
- Validation hooks: ensure `tags` normalized ids and `id_comp` uniqueness in `tags`.
- Files post-upload: compute and store `hash` if not provided (optional).

## Security
- Restrict custom route to authenticated requests; validate that `userId` matches requester or has admin privileges.
- Rate-limit if needed via hook context.

## Testing Strategy
- Unit tests for route handler logic in JS VM (where possible).
- Integration tests from Python hitting the custom route and asserting results parity with the legacy SQL filters.

## Deployment
- Include hooks with migrations bundle; `routerAdd` in `onBeforeServe` to register the route.
- Version route implementation to allow iterative improvements.