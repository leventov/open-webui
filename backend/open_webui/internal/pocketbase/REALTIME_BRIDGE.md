# Realtime Bridge: PocketBase SSE -> Socket.IO

This document proposes a concrete Python abstraction to bridge PocketBase (PB) realtime events to Open WebUI’s existing Socket.IO layer.

## Goals
- Subscribe to PB collection events (create/update/delete) via SSE.
- Normalize and forward events to Socket.IO rooms/namespaces used by the app.
- Allow selective subscriptions per collection and per user scope.

## Implementation Plan
- [ ] Create `PocketBaseRealtimeBridge` component with start/stop lifecycle.
- [ ] Register subscriptions for `users`, `chats`, `messages`, `message_reactions`, `files`, `tags`.
- [ ] Implement event normalization to `{ type, entity, id, data, ts }`.
- [ ] Emit to rooms: `user:{user_id}`, `channel:{channel_id}`, etc.
- [ ] Add auto-reconnect with backoff and idempotency using `(record.id, record.updated)`.
- [ ] Metrics and logs for event rates and handler errors.

## Architecture
- Maintains a PB client connection and SSE subscriptions per collection.
- Converts PB events to domain events with payloads matching existing Socket.IO listeners.
- Provides start/stop lifecycle methods tied to application startup/shutdown.

## Subscriptions
- Collections: `users`, `chats`, `messages`, `message_reactions`, `files`, `tags` (extendable).
- Use PB subscription filters where possible; filter in Python otherwise.

## Event Normalization
- PB event format: `{action: 'create'|'update'|'delete', record: {...}}`.
- Map PB `record` to Pydantic model via repository mappers.
- Emit Socket.IO events like `chat:updated`, `message:created`, etc.

## Error Handling & Security
- SSE auto-reconnect with backoff.
- Idempotent handlers.
- Use admin/service account; route emissions by `user_id`/`channel_id` to avoid leakage.

## Testing
- [ ] Mock PB SSE and assert Socket.IO emissions.
- [ ] Integration test with PB and a Socket.IO test client.