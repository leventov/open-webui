# Repository Interfaces and PocketBase Adapters

This document defines repository interfaces for Open WebUI domain models and the mapping from current SQLAlchemy-backed methods to PocketBase (PB) implementations using the Python client (`vaphes/pocketbase`).

## Strategy (explicit)
- [x] Do NOT emulate `SessionLocal()` / `get_db` for PB.
  - We will not duck-type SQLAlchemy sessions; instead, we will implement PB adapters and update the existing model modules to call them inside their methods, preserving public signatures.
- [x] No per-model feature flags in production. A single backend is targeted post-migration (PB).

## Principles
- Keep existing API shapes returned to the rest of the app unchanged.
- Repositories encapsulate all persistence details (SQL or PB) and mapping between PB records and Pydantic models.
- PB adapters must be idempotent and handle retries.
- Provide consistent error and NotFound semantics.

## Interfaces (illustrative)
- `IUsersRepo`, `IChatsRepo`, `IMessagesRepo`, `ITagsRepo`, `IFilesRepo`, `IGroupsRepo`, `IFoldersRepo`, `IModelsRepo`.

## PB Client Wrapper
- [ ] Implement a PB client wrapper: auth, retries, pagination helpers.

## Mappings: SQL -> PB
- IDs: use PB `id` and map types.
- Timestamps: PB `created`/`updated` → epoch fields as needed.
- Filtering: translate common filters to PB strings.
- Tags filtering (decision): use PB relations + one custom route for “has ALL” semantics.
- Pagination: `skip/limit` ↔ `page/perPage`.
- Sorting: PB `sort` (e.g., `-updated`).

## Error Semantics
- 404 → `None`/`False` per method kind.
- Validation/Conflict → return explicit error or `None` consistently; log context.

## Implementation Checklist
- [ ] Implement `UsersRepoPB` and map to `UserModel` APIs.
- [ ] Implement `ChatsRepoPB` (CRUD, share, pin/archive) and tag relation management.
- [ ] Implement `MessagesRepoPB` and reactions.
- [ ] Implement `TagsRepoPB` with `id_comp` uniqueness and normalization.
- [ ] Implement `FilesRepoPB` with multipart upload and download URL helpers.
- [ ] Wire `backend/open_webui/models/*.py` to call PB repos internally without changing public signatures.
- [ ] Add contract tests running against SQL and PB adapters for parity.

## Notes on Safety
- Duck-typing SQLAlchemy sessions is not safe due to breadth of query API used in current modules.
- Encapsulating PB logic in adapters and updating only the internals of the model modules keeps risk low and scope contained.