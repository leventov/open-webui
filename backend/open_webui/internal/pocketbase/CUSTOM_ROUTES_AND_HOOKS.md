# Custom Routes and Hooks in PocketBase

This document proposes JavaScript routes and hooks to cover advanced behaviors not easily expressed with PocketBase filters, notably tag filtering.

## Directories
- Migrations: `backend/open_webui/internal/pocketbase/migrations/`
- Hooks: `backend/open_webui/internal/pocketbase/hooks/`
- Delivery: copy to PB server `pb_migrations/` and `pb_hooks/` or bake into a custom PB build.

## Decision: Tags Route
- [x] Implement a custom route for "has ALL tags" chat filtering.
- Route: `GET /api/openwebui/chats/byTags`
- Query params:
  - `userId` (string, required)
  - `tags` (comma-separated normalized tag identifiers)
  - `mode` (`all`|`any`, default `all`)
  - `skip`, `limit` (pagination)
- Behavior:
  - Resolve tag strings to `tags` collection record IDs.
  - Base query: `user_id == userId`.
  - Use relation filter for `any`, and for `all` compute intersection in the hook.
  - Return `{ items, total }` mapped to `ChatResponse` shape.

## Route Registration
- Use a top-level `routerAdd()` in the hook file (e.g., `pb_hooks/tag_filters.pb.js`) as per PB JS overview.
- No need for `onBeforeServe` wrappers.

## Additional Hooks
- Validation for `tags` normalization and `tags.id_comp` uniqueness.
- Optional: keep mirrored `meta.tags` during transition.

## Security
- Auth required; ensure `userId` matches requester or admin.

## Implementation Checklist
- [ ] Write `pb_hooks/tag_filters.pb.js` registering the route via top-level `routerAdd`.
- [ ] Add unit tests (where viable) and Python integration tests against the route.
- [ ] Document deployment steps (copy hooks, restart PB).

## References
- PocketBase JS overview: `https://pocketbase.io/docs/js-overview/`
- PB JS router: `https://pocketbase.io/docs/js-routing/`