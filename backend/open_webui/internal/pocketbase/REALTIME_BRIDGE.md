# Realtime Bridge: PocketBase SSE -> Socket.IO

This document proposes a concrete Python abstraction to bridge PocketBase (PB) realtime events to Open WebUI’s existing Socket.IO layer, with concrete references to current code.

References:
- Socket.IO server and handlers: `backend/open_webui/socket/main.py`
- PB realtime (SSE): `https://pocketbase.io/docs/api-realtime/`

## What is used and where (Socket.IO today)
- Server: `python-socketio` AsyncServer exposed as an ASGI app (see `backend/open_webui/socket/main.py`).
  - Server creation and transports: lines ~22–100.
  - Redis scaling (when `WEBSOCKET_MANAGER == "redis"`): AsyncRedisManager for pub/sub fanout; session/user pools via RedisDict (lines ~35–69, 70–100).
  - ASGI app: `app = socketio.ASGIApp(sio, socketio_path="/ws/socket.io")` around lines ~131–170.
- Rooms and handlers:
  - Connect handshake with optional JWT auth attaches user to session: ~259–275.
  - `user-join` populates pools and joins user’s channels as rooms: ~278–304; `join-channels`: ~307–325; `join-note`: ~356.
  - `channel-events` verifies sender is in room and broadcasts: ~359–384.
  - Usage tracking: ~239–256.
  - Collaborative notes: `ydoc:document:update` to `doc_{document_id}` with debounced saves: ~527–567.
- Event dispatch helpers used by backend:
  - `get_event_emitter(request_info)`: emits `chat-events` to user sessions and updates DB as needed: ~643–708.
  - `get_event_call(request_info)`: request/response RPC to a specific session via `sio.call("chat-events", ...)`: ~711–724.

## Bridge Goals
- Subscribe to PB collection events (create/update/delete) via SSE.
- Normalize and forward events to the same Socket.IO rooms/namespaces (e.g., `user:{user_id}`, `channel:{channel_id}`, `note:{id}`).
- Preserve existing client contracts (event names and payload structures).

## Implementation Plan
- [ ] Create `PocketBaseRealtimeBridge` with start/stop lifecycle (initialized on FastAPI startup).
- [ ] Register SSE subscriptions for `users`, `chats`, `messages`, `message_reactions`, `files`, `tags`.
- [ ] Normalize PB events to `{ type, entity, id, data, ts }` and dispatch to:
  - `chat:created|updated|deleted` (rooms: `user:{user_id}`, `channel:{channel_id}` when applicable)
  - `message:created|updated|deleted` (room: `channel:{channel_id}`; also `user:{user_id}`)
  - `reaction:*`, `file:*`, and note-related events as applicable
- [ ] Integrate with existing helpers where possible (e.g., reuse `get_event_emitter` patterns for user-targeted events).
- [ ] Add auto-reconnect with backoff and idempotency using `(record.id, record.updated)`.
- [ ] Add metrics and logs for event throughput and handler failures.

## Subscriptions & Filtering
- Use PB filters to scope by `user_id`/`channel_id` when appropriate; otherwise filter in Python before emitting to rooms.
- Beware cross-tenant leaks; ensure room targets always respect the authenticated owner of the data.

## Event Normalization Examples
- PB → Socket.IO mapping:
  - Chats: PB `{ action: 'update', record: { id, user_id, ... } }` → emit `chat:updated` to `user:{user_id}`.
  - Messages: PB `{ action: 'create', record: { id, channel_id, user_id, ... } }` → emit `message:created` to `channel:{channel_id}` and `user:{user_id}`.
  - Reactions: PB `{ action: 'delete', record: { message_id, user_id, name } }` → emit `reaction:deleted` to `channel:{channel_id}` (channel resolved via message lookup if needed).

## Error Handling & Security
- SSE auto-reconnect with exponential backoff.
- Idempotent handlers (drop duplicates based on `record.id` and `record.updated`).
- Use a PB service account for SSE; bridge applies server-side authorization by routing only to appropriate rooms.

## Testing
- [ ] Unit: mock PB SSE events; assert normalized Socket.IO emissions.
- [ ] Integration: run PB and socket server, use test Socket.IO client; assert presence/room handling and payload formats.