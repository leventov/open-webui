# Authentication and Authorization Integration (Open WebUI ↔ PocketBase)

This document explains how Open WebUI’s existing authentication (JWT/API key/OAuth/LDAP) and authorization (roles, groups, access_control, permissions) interact with PocketBase (PB) when PB becomes the storage backend.

References:
- PB authentication: `https://pocketbase.io/docs/authentication/`
- PB production guidance: `https://pocketbase.io/docs/going-to-production/`

## Current Open WebUI AuthN/AuthZ (summary)
- Identity and sessions:
  - Backend issues JWTs (`open_webui/utils/auth.py`: `create_token`, `decode_token`) with `id` claim referencing `users.id`.
  - Cookie `token` (HTTP-only) is set on signin/signup (see `backend/open_webui/routers/auths.py`), plus bearer Authorization supported.
  - API keys begin with `sk-` and are stored on the user record; requests can use API keys instead of JWT.
- Roles: `pending` | `user` | `admin` (see `users.py` model and `auths.py` flows).
- Groups and permissions:
  - Groups stored in `groups` model; permissions aggregated via `utils/access_control.py` (combine group permissions, fallback to defaults).
  - Per-record `access_control` fields (models like `models.py`) checked via `has_access(user_id, type, access_control)`.
- OAuth and LDAP: optional sign-in paths build users and still mint Open WebUI JWTs.

## Integration Approach with PocketBase
- Service-to-service model: Open WebUI remains the identity provider for user sessions to the Open WebUI API and UI. PB is a data store; we use a PB service/admin account for API calls from the server.
  - Rationale: existing auth flows (JWT/API key/OAuth/LDAP) remain unchanged for clients; migrating identity to PB Auth isn’t required and would be disruptive.
  - PB collection rules can still be configured, but Open WebUI enforces authorization in its service layer as today.

### What we do in PB
- Admin/service auth for backend:
  - Backend initializes a PB client with admin/service credentials. Token is stored server-side and refreshed transparently (see REPOSITORIES_AND_ADAPTERS.md wrapper checklist).
- Collections rules:
  - Start permissive to the service user (admin); tighten later if we offload some read filters to PB rules.
  - Do not rely on PB as the end-user identity/authorization decision-maker for application requests.

### What we keep in Open WebUI
- JWT issuance/verification and API key auth remain as-is (`utils/auth.py`, `routers/auths.py`).
- Role checks, permissions aggregation, and `access_control` checks remain in Python (`utils/access_control.py`).
- Group membership management remains in Open WebUI’s `groups` repo (stored in PB collections).

## Optional Future: PB as Identity Provider
- PB supports record auth and OAuth providers. If desired later:
  - Use PB Auth for user login; store PB record `id` alongside Open WebUI `users.id` (mapping field) or make PB the canonical user store.
  - Adjust JWT creation to mint tokens from PB (or proxy PB tokens through the backend).
  - Mirror/replace groups and access rules with PB rules.
- Not part of the initial plan; initial plan uses PB only as storage with a service account.

## Authorization Mapping Details
- Roles: Store `role` on `users` collection (PB). Enforce via Python checks as today.
- Groups: Mirror `groups` and membership relations in PB. Aggregation logic unchanged.
- access_control: Keep JSON on records where used; enforce in Python via `has_access` (unchanged).
- Permissions: Keep computed at runtime from groups + defaults; not enforced via PB rules in v1.

## Security & Production Considerations
- PB hardening per `going-to-production`:
  - Run PB behind a reverse proxy, enable TLS, secure admin credentials.
  - Restrict PB Admin UI exposure in production (service account only).
  - Configure PB CORS as needed for server-side only access.
- Backend secrets:
  - Store PB service credentials securely.
  - Ensure PB client tokens are rotated/refreshed as required by the client library.

## Implementation Checklist
- [ ] Create PB service/admin account and configure credentials for the backend.
- [ ] Implement PB client wrapper with login/refresh and retries.
- [ ] Store roles/groups/access_control fields in PB collections.
- [ ] Keep Open WebUI JWT/API key flows unchanged.
- [ ] Add tests: ensure authorization checks continue to function with PB-backed repos (groups, permissions, access_control).