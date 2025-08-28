# Offline, Bidirectional Migrations (SQLite/PostgreSQL ↔ PocketBase)

Goal: Support full, offline migrations of Open WebUI data between SQL backends (SQLite/Postgres) and PocketBase (PB) in either direction, without partial/staged operation. This enables safe experimentation and easy rollback.

## Principles
- Migrations are run with the service offline (maintenance window) to avoid dual-writes.
- Data transforms are reversible or mirrored to ensure parity.
- Scripts are idempotent and resumable.
 - Authentication tokens are not migrated; users will re-authenticate post-cutover when changing IdP.

## Mirrored Fields (required)
- Tags:
  - Legacy SQL stores tags in `chat.meta.tags` (JSON array of normalized tag ids).
  - PB model uses relations `chats.tags -> tags` (many-to-many) and also stores the normalized array in `chat.meta.tags` to preserve reversibility.
  - This mirror is mandatory to support round trips without lossy transforms.

## Direction A: SQL → PocketBase
- Use batch Python scripts to stream SQL rows and insert into PB.
- For `tags`:
  - Compute `tags.id_normalized` and `tags.id_comp = "{id_normalized}:{user_id}"`.
  - Populate `chats.tags` relation and also copy the JSON array into `chat.meta.tags` in PB.
- Files:
  - Prefer reuse of storage (local dir or S3) and migrate only metadata. Otherwise, upload binaries.
- Preserve canonical IDs where required or store mapping tables if PB ids differ.
 - Authentication:
  - User sessions: Open WebUI JWTs cannot be transformed into PB record tokens securely; purge/expire sessions on cutover.
  - Passwords: if legacy password hashes are incompatible with PB, mark accounts to require reset or plan an OAuth-first login path.

## Direction B: PocketBase → SQL
- Use batch Python scripts to export PB records and insert into SQL.
- For `tags`:
  - Read both the `chats.tags` relation and `chat.meta.tags` array. Prefer the relation for authoritative set; if missing/inconsistent, fall back to mirrored JSON array.
  - Recreate SQL composite identity: `(id=user_normalized_tag, user_id)`.
- Files:
  - If using shared storage, migrate only metadata; otherwise download and write binaries.
 - Authentication:
  - PB tokens do not migrate to Open WebUI sessions; require re-login after switching back.
  - Consider issuing a forced-signout at maintenance end.

## Prior Art
- Open WebUI already supports moving between SQLite and PostgreSQL via Alembic migrations and runtime config; content shape is identical since both are SQLAlchemy-backed.
- This proposal follows the same spirit for PB by defining a PB schema that retains key JSON fields (like `meta.tags`) to allow round-tripping.

## Validation
- Row counts per collection/table and random sampling per entity type.
- Referential integrity checks (e.g., messages→channels; chats→users; chats.tags→tags).
- Hash/size checks for files.

## Deliverables
- [ ] SQL→PB migration scripts (per collection) with dependency ordering.
- [ ] PB→SQL migration scripts (per collection) with dependency ordering.
- [ ] Mapping for IDs where necessary, or explicit decision to rely on PB ids as canonical with back-mapping.
- [ ] Verification scripts (counts, samples, relations).
- [ ] Runbooks for both directions.
 - [ ] Communication template for one-time re-login and password reset instructions.