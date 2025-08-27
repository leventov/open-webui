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

## Default Integration Approach with PocketBase
- Service-to-service model: Open WebUI remains the identity provider for user sessions to the Open WebUI API and UI. PB is a data store; we use a PB service/admin account for API calls from the server.
  - Rationale: existing auth flows (JWT/API key/OAuth/LDAP) remain unchanged for clients; PB is not directly exposed to end-users.
  - PB collection rules can still be configured, but Open WebUI enforces authorization in its service layer as today.

## Optional: Reusing Open WebUI Auth with PocketBase (when exposing PB)
If you choose to expose PB directly (e.g., Admin UI, custom PB endpoints, or third-party apps using PB SDK), you can reuse Open WebUI credentials/tokens so users don’t manage two identities.

### Option 1: Token exchange (Open WebUI JWT → PB record token)
- Flow
  - Create a PB hook/route `POST /api/openwebui/auth/exchange` that accepts an Open WebUI JWT.
  - The hook verifies the JWT either via:
    - Shared secret (same signing key) and audience/issuer check, or
    - Introspection callback to Open WebUI (`/auths/` endpoints) to validate and fetch user info.
  - The hook looks up/creates a PB `users` record mapped to Open WebUI `users.id` (store OWUI id in a dedicated field, e.g., `owui_user_id`).
  - The hook issues a PB auth token for that record (standard PB record auth token) with TTL no longer than the remaining JWT lifetime.
- Considerations
  - Enforce strict audience/issuer and clock skew; don’t exceed OWUI token expiry.
  - Provide logout/invalidate on PB side if OWUI token is revoked (short TTL preferred).
  - Map roles and minimal profile fields (name, email, avatar) to PB record for UI display; authorization remains server-side in OWUI for app APIs.

### Option 2: Align OAuth providers
- Configure the same OAuth providers in PB as in OWUI (Google, GitHub, etc.). Users log in to PB with the same provider.
- Maintain a mapping between PB record and OWUI user (`owui_user_id` field) via email or a dedicated claim.
- Pros: no custom hook; Cons: provider drift, duplicate sessions.

### Option 3: Reverse proxy header-based SSO
- Place PB behind a reverse proxy that validates the OWUI JWT and injects headers (e.g., `X-User-Id`, `X-User-Email`).
- Add a PB hook to trust those headers (from the proxy only) and map to a PB record session.
- Useful when Admin UI or certain routes must be visible to authenticated staff only.

### Option 4: LDAP reuse
- OWUI already supports LDAP sign-in. PB natively doesn’t support LDAP as a first-class provider but can leverage custom hooks.
- Implement a PB hook that binds to LDAP (mirroring OWUI LDAP codepaths) or rely on Option 1 token exchange so PB trusts OWUI’s LDAP-backed JWT.
- Community discussions for reference:
  - `https://github.com/pocketbase/pocketbase/discussions/4796`
  - `https://github.com/seriousm4x/UpSnap/discussions/76`
  - `https://github.com/pocketbase/pocketbase/discussions/6016`
  - `https://github.com/pocketbase/pocketbase/discussions/4132`

## Authorization Mapping Details
- Roles: Store `role` on `users` collection (PB). Enforce via Python checks as today.
- Groups: Mirror `groups` and membership relations in PB. Aggregation logic unchanged.
- access_control: Keep JSON on records where used; enforce in Python via `has_access` (unchanged).
- Permissions: Keep computed at runtime from groups + defaults; not enforced via PB rules in v1.

## Security & Production Considerations
- PB hardening per `going-to-production`:
  - Run PB behind a reverse proxy, enable TLS, secure admin credentials.
  - Restrict PB Admin UI exposure in production (service account only or SSO-protected).
  - Configure PB CORS as needed for server-side only access, unless exposing endpoints.
- Backend secrets:
  - Store PB service credentials securely.
  - Ensure PB client tokens are rotated/refreshed as required by the client library.

## Implementation Checklists
- Token exchange
  - [ ] PB hook route `/api/openwebui/auth/exchange` that validates OWUI JWT (shared secret or introspection).
  - [ ] Map `owui_user_id` to PB record; create if missing; issue PB token with bounded TTL.
  - [ ] Document reverse mapping and logout story.
- OAuth alignment
  - [ ] Configure same providers in PB; ensure email/subject mapping to `owui_user_id`.
- Reverse proxy SSO
  - [ ] Gate PB behind proxy that validates OWUI JWT and injects identity headers; PB hook trusts proxy and maps record.
- LDAP
  - [ ] Prefer token exchange to avoid duplicating LDAP logic; if needed, implement a PB hook for LDAP bind.