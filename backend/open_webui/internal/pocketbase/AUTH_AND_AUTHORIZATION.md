# Authentication and Authorization Integration (Open WebUI ↔ PocketBase)

This document defines the recommended, native authentication and authorization model when PocketBase (PB) is the storage backend. It aligns with best practices from systems like Supabase and Convex where the database/auth layer is the canonical user system for app-level auth.

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
  - Groups stored in `groups` model; permissions aggregated via `utils/access_control.py`.
  - Per-record `access_control` fields (e.g., in models) enforced via `has_access(user_id, type, access_control)`.
- OAuth and LDAP: optional sign-in paths mint Open WebUI JWTs today.

## Default Approach: PocketBase-native Identity (recommended)
- PocketBase is the identity provider (IdP) for the application.
- Use a PocketBase Auth collection for users (PB’s first-class auth collection type), not a plain base collection. This enables login, password reset, OAuth providers, email verification, and built-in session tokens.
- The SPA authenticates with PB (email/password, OAuth, etc.) against the Auth collection and obtains a PB record token/cookie.
- Open WebUI accepts PB-issued user tokens for all app endpoints, verifies them, and derives the current user from PB. Open WebUI no longer mints its own primary user JWT for interactive sessions.

### Verification on the backend
- Open WebUI verifies PB identity via one of the following (prefer top to bottom):
  - Introspection: call PB `authRefresh` (record auth) with the presented token to validate and retrieve the user record. Cache briefly to avoid hot-looping.
  - Local verification: verify PB’s record token using a configured secret/public-key if available; still refresh periodically to honor revocations.
  - Reverse proxy trust: place PB behind a gateway that validates PB token and injects identity headers; OWUI trusts only the gateway.

### Identity and profile mapping
- Canonical user store: a PB Auth collection (commonly named `users`).
- Open WebUI’s `users` model becomes a thin projection of the Auth collection record:
  - Store `pb_user_id` (authoritative key) and only OWUI-specific profile fields.
  - Do not store credential hashes in Open WebUI; rely on PB Auth features (verification, password reset, OAuth linking).
  - Roles and groups are PB-side fields/relations; Open WebUI consumes them for authorization decisions.

### OAuth/OIDC and LDAP
- Use PB providers for OAuth/OIDC on the Auth collection. Disable overlapping providers in Open WebUI when PB mode is enabled.
- For LDAP deployments, implement PB-side integration (hook) or place PB behind an auth proxy that handles LDAP and forwards an authenticated session.

### API keys
- Keep API keys as an application feature distinct from user interactive sessions:
  - Option A (recommended): move per-user API keys to a PB collection (e.g., `api_keys` related to `users`) and verify in Open WebUI.
  - Option B (compatibility): retain current `users.api_key` field but ensure the owning user is resolved from PB identity. New keys are created only when PB user exists.

## Legacy bridging options (not recommended as default)
These may be used temporarily during migration or when PB cannot be the visible IdP.

### Option 1: Token exchange (Open WebUI JWT → PB record token)
- PB custom route `POST /api/openwebui/auth/exchange` accepts an Open WebUI JWT, validates it, maps/creates a PB `users` record, and issues a short-lived PB token (bounded by the OWUI token’s remaining TTL).

### Option 2: Align OAuth providers
- Configure the same OAuth providers in PB and OWUI and map via `pb_user_id`/email. Beware drift and duplicate sessions.

### Option 3: Reverse proxy header-based SSO
- Put PB behind a proxy that validates OWUI JWT and injects identity headers to PB.

## Authorization Mapping
- Roles: store role on PB `users` and read in Open WebUI dependencies (`get_verified_user`, `get_admin_user`).
- Groups: store groups and memberships in PB; Open WebUI aggregates permissions the same way as today.
- Per-record access: keep `access_control` JSON and enforce in Open WebUI.
- PB collection rules may additionally restrict direct PB access if you expose PB; Open WebUI remains the enforcement point for app APIs.

## Security & Operations
- Minimize long-lived PB admin/service credentials. In the PB-native model, Open WebUI typically operates on behalf of the end-user using that user’s PB token.
- If a service account is required (background jobs), provision a least-privilege PB user and store its credentials in environment variables or a secret manager, never in PB itself.
- Harden PB per production guidance (TLS, reverse proxy, narrow exposure of Admin UI, CORS).

## Migration Implications
- Existing Open WebUI JWTs and refresh cookies cannot be securely migrated to PB tokens. A one-time re-login is required after cutover.
- For local-password users whose hash algorithm is incompatible with PB, require password reset or first-login via OAuth.
- Document re-login in rollout communications and show a friendly banner post-cutover.

## Implementation Checklist (PB-native IdP)
- [ ] Add a PB-auth verification dependency in Open WebUI (introspection-first, fallback to local verification).
- [ ] Introduce `WEBUI_AUTH_MODE=pocketbase` config that disables local JWT minting and OAuth providers in OWUI.
- [ ] Map `pb_user_id` and hydrate Open WebUI’s `User` projection on demand; remove storage of credentials in OWUI.
- [ ] Move roles/groups to PB and update lookups.
- [ ] Optional: move API keys into PB (`api_keys` collection) and adjust verification.
- [ ] Update admin/config and session bootstrap endpoints to accept PB tokens and set HttpOnly cookies with PB token when appropriate.
- [ ] Communicate re-login requirement in `ROLLOUT_AND_MIGRATION.md` and `OFFLINE_MIGRATIONS.md`.