# Realtime Bridge: PocketBase SSE -> Socket.IO

This document proposes a concrete Python abstraction to bridge PocketBase (PB) realtime events to Open WebUI’s existing Socket.IO layer.

## Goals
- Subscribe to PB collection events (create/update/delete) via SSE.
- Normalize and forward events to Socket.IO rooms/namespaces used by the app.
- Allow selective subscriptions per collection and per user scope.

## Architecture
- Component: `PocketBaseRealtimeBridge`
  - Maintains a PB client connection and SSE subscriptions per collection.
  - Converts PB events to domain events with payloads matching existing Socket.IO listeners.
  - Provides start/stop lifecycle methods tied to application startup/shutdown.

## Subscriptions
- Collections to subscribe: `users`, `chats`, `messages`, `message_reactions`, `files`, `tags`, others as needed.
- PB supports subscription filters; where not sufficient, filter on the Python side before emitting.
- For user-scoped rooms, use event payload `user_id` to emit only to `room=f"user:{user_id}"` or to channel-specific rooms (e.g., `channel:{channel_id}`).

## Event Normalization
- PB event format: `{action: 'create'|'update'|'delete', record: {...}}`.
- Map PB `record` to Pydantic model via repository mappers.
- Emit Socket.IO events like:
  - `chat:updated`, `chat:created`, `chat:deleted`
  - `message:*`, `reaction:*`, `file:*`
- Event envelope: `{ type, entity, id, data, ts }` for consistency and client-side handling.

## Python Abstraction (conceptual)
- `PocketBaseRealtimeBridge` responsibilities:
  - `register_handler(collection: str, handler: Callable[[PBEvent], None])` to allow feature modules to attach logic.
  - `subscribe(collection: str, filter: Optional[str])` to start SSE.
  - `emit_socket(event_name: str, payload: dict, room: Optional[str])` bridging to existing `sio` instance.
- Integration points:
  - On FastAPI startup: initialize PB client, create bridge, register default handlers per collection, start subscriptions.
  - On shutdown: stop subscriptions, close client.

## Example Event Flow
1. PB `messages` record updated (new reply).
2. SSE handler maps to domain `MessageModel` and computes derived fields if needed (e.g., reply count changes for parent).
3. Emit `message:updated` to `room=channel:{channel_id}` and `room=user:{user_id}`.

## Error Handling & Resilience
- SSE stream auto-reconnect with backoff.
- Idempotent handlers (ignore duplicate events by `record.id` + `updated` timestamp).
- Log and metrics for event throughput and failures.

## Security
- PB Admin or service account used for SSE; events are not directly exposed to clients until filtered/emitted by our server.
- Ensure we do not leak cross-tenant data: always route by `user_id`/`channel_id` rooms.

## Testing
- Mock PB SSE stream; simulate events and assert Socket.IO emissions.
- Integration test in a dev env with PB running and Socket.IO test client.

## Rollout
- Start with `messages` and `chats` collections; validate client experience.
- Add other collections incrementally.