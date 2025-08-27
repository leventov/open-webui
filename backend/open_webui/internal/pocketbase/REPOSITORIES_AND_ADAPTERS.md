# Repository Interfaces and PocketBase Adapters

This document defines repository interfaces for Open WebUI domain models and the mapping from current SQLAlchemy-backed methods to PocketBase (PB) implementations using the Python client (`vaphes/pocketbase`).

## Principles
- Keep existing API shapes returned to the rest of the app unchanged.
- Repositories encapsulate all persistence details (SQL or PB) and mapping between PB records and Pydantic models.
- Enforce idempotency where atomic transactions were previously assumed implicitly.
- Provide consistent error and NotFound semantics.

## Interfaces (illustrative)
- `IUsersRepo`
  - `insert_new_user(...) -> UserModel | None`
  - `get_user_by_id(id) -> UserModel | None`
  - `get_user_by_email(email) -> UserModel | None`
  - `get_users(filter, skip, limit) -> UserListResponse`
  - `update_user_by_id(id, updates) -> UserModel | None`
  - `delete_user_by_id(id) -> bool`
- `IChatsRepo`
  - `insert_new_chat(user_id, form) -> ChatModel | None`
  - `get_chat_by_id(id) -> ChatModel | None`
  - `get_chats_by_user_id(user_id, filter, skip, limit) -> ChatListResponse`
  - `update_chat_by_id(id, chat_json) -> ChatModel | None`
  - `toggle_pinned/archived(id) -> ChatModel | None`
  - `share_chat(chat_id) -> ChatModel | None`
  - Tags:
    - `update_chat_tags(id, user_id, tag_names) -> ChatModel | None`
    - `get_chat_tags(id, user_id) -> list[TagModel]`
- `IMessagesRepo`
  - `insert_new_message(form, channel_id, user_id) -> MessageModel | None`
  - `get_message_by_id(id) -> MessageResponse | None`
  - `get_messages_by_channel_id(channel_id, skip, limit) -> list[MessageModel]`
  - `get_messages_by_parent_id(channel_id, parent_id, skip, limit) -> list[MessageModel]`
  - Reactions CRUD
- `ITagsRepo`
  - `insert_new_tag(name, user_id) -> TagModel | None`
  - `get_tag_by_name_and_user_id(name, user_id) -> TagModel | None`
  - `get_tags_by_user_id(user_id) -> list[TagModel]`
  - `delete_tag_by_name_and_user_id(name, user_id) -> bool`
- `IFilesRepo`
  - `insert_new_file(user_id, form) -> FileModel | None`
  - `get_file_by_id(id) -> FileModel | None`
  - `get_files_by_user_id(user_id) -> list[FileModel]`
  - `update_file_metadata/hash/data(id, ...) -> FileModel | None`
  - `delete_file_by_id(id) -> bool`

(Additional interfaces for functions, tools, groups, folders, models, knowledge, notes, channels, feedbacks, prompts, memories with similar CRUD.)

## Adapter Selection
- Global env `STORAGE_BACKEND=sqlalchemy|pocketbase` or per-domain overrides (e.g., `STORAGE_BACKEND_USERS=pocketbase`).
- A factory instantiates the appropriate implementation.

## PocketBase Client Wrapper
- Create a thin wrapper encapsulating:
  - Auth (admin credentials, token storage, refresh)
  - Retry with backoff on network/5xx
  - Rate limiting
  - Helpers for pagination (PB uses page/perPage)

## Mappings: SQL -> PB
- IDs
  - Use PB `id`. Where callers expect existing `id`, pass-through.
- Timestamps
  - Map PB `created`, `updated` to model `created_at`, `updated_at` (convert to epoch seconds/ns as needed).
- Filtering
  - Translate `.filter_by(...)`, `ilike`, basic ranges to PB filter strings.
  - For complex tag filters: call PB custom route (see Hooks doc) or use `expand` on relation fields and post-filter client-side.
- Pagination
  - Translate `skip/limit` to `page/perPage`.
- Sorting
  - Translate to PB `sort` parameter (e.g., `-updated` for desc).

## Error Semantics
- 404 -> return `None` for getters, `False` for deleters.
- Validation -> raise domain-specific exceptions or return `None` consistently; log with context.
- Conflict (unique violations) -> return explicit error result where used today.

## Example Method Transliteration (conceptual)
- `ChatsRepoPB.update_chat_by_id(id, chat_dict)`
  - Fetch record by `id` from `chats`.
  - Merge/replace `chat` field; update `title`, `updated`.
  - `client.collection('chats').update(id, { chat, title, ... })`
  - Map PB record back to `ChatModel`.

- `ChatsRepoPB.update_chat_tags(id, user_id, tag_names)`
  - Resolve/create tags in `tags` collection (normalize, ensure `id_comp` uniqueness).
  - Update `chats.tags` relation list with tag record IDs.
  - If advanced filter compatibility is required, optionally mirror normalized tag strings into `meta.tags` for transition.

- `FilesRepoPB.insert_new_file(user_id, form)`
  - Upload file via `collection('files').create(formData)` with file field.
  - Set `hash`, `data`, `meta`, `user_id` alongside.
  - Return mapped `FileModel`.

## Testing Strategy
- Contract tests per interface that run against both adapters (SQL and PB) to ensure parity.
- Seed PB with migrations before tests; clean collections between tests.

## Observability
- Add structured logs around adapter operations and timings.
- Collect metrics on PB latency/error rates.