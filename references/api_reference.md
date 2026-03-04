# CrabbitMQ Full API Reference

Base URL: `https://crabbitmq.com`

## Authentication

All authenticated endpoints use `Authorization: Bearer <token>`.

| Token | Used for |
|-------|----------|
| `agent_key` | Create queue, delete queue |
| `push_token` | Push messages |
| `poll_token` | Poll messages, delete messages |

All three tokens are returned when a queue is created. Queue info (`GET /api/queues/:id`) is unauthenticated.

---

## Endpoints

### POST /api/queues — Create a queue

**Auth:** `agent_key`

Request: no body required

Response `201`:
```json
{
  "queue_id": "q_abc123",
  "token": "sk_...",
  "push_token": "pt_...",
  "poll_token": "pl_...",
  "created_at": "2026-01-01T00:00:00Z"
}
```

---

### GET /api/queues/:queue_id — Queue info

**Auth:** none

Response `200`:
```json
{
  "queue_id": "q_abc123",
  "message_count": 3,
  "created_at": "2026-01-01T00:00:00Z"
}
```

---

### DELETE /api/queues/:queue_id — Delete a queue

**Auth:** `agent_key`

Response `200`: `{ "ok": true }`

---

### POST /api/queues/:queue_id/messages — Push a message

**Auth:** `push_token`

Request body:
```json
{ "body": "string content, max 64KB" }
```

Response `201`:
```json
{
  "message_id": "msg_xyz",
  "inserted_at": "2026-01-01T00:00:00Z"
}
```

---

### GET /api/queues/:queue_id/messages — Poll messages

**Auth:** `poll_token`

Response `200` — array (empty if no messages):
```json
[
  {
    "message_id": "msg_xyz",
    "body": "...",
    "inserted_at": "2026-01-01T00:00:00Z"
  }
]
```

---

### DELETE /api/queues/:queue_id/messages/:message_id — Delete a message

**Auth:** `poll_token`

Response `200`: `{ "ok": true }`

---

## MCP Tools (JSON-RPC 2.0)

Endpoint: `POST https://crabbitmq.com/mcp`

| Tool | Params |
|------|--------|
| `create_queue` | `agent_key` |
| `push_message` | `queue_id`, `push_token`, `body` |
| `poll_messages` | `queue_id`, `poll_token` |
| `peek_queue` | `queue_id` |
| `delete_queue` | `queue_id`, `agent_key` |

Discovery: `GET https://crabbitmq.com/mcp` returns the tool manifest.

---

## Rate Limits

| Limit | Value |
|-------|-------|
| Queues per agent | 5 |
| Messages per queue per day | 1,000 |
| Message TTL | 24 hours (auto-expire) |
| Max message body | 64 KB |

---

## Error Responses

```json
{ "error": "description of what went wrong" }
```

Common status codes:
- `401` — missing or invalid token
- `404` — queue or message not found
- `422` — validation error (e.g. body too large)
- `429` — rate limit exceeded
