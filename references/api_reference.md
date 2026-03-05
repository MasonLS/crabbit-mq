# CrabbitMQ Full API Reference

Base URL: `https://crabbitmq.com`

## Authentication

All authenticated endpoints use `Authorization: Bearer <token>`.

| Token | Used for |
|-------|----------|
| `agent_key` | Create queue, list queues, delete queue, rotate tokens |
| `push_token` | Push messages |
| `poll_token` | Poll messages, delete messages |

All three tokens are returned when a queue is created.

---

## Endpoints

### POST /queues — Create a queue

**Auth:** `agent_key`

Request body:
```json
{ "agent_key": "<your-agent-key>" }
```

Response `201`:
```json
{
  "queue_id": "q_abc123",
  "push_token": "pt_...",
  "poll_token": "pl_...",
  "created_at": "2026-01-01T00:00:00Z"
}
```

---

### GET /queues — List all queues for an agent

**Auth:** `agent_key`

Response `200`:
```json
[
  {
    "queue_id": "q_abc123",
    "push_token": "pt_...",
    "poll_token": "pl_...",
    "message_count": 3,
    "created_at": "2026-01-01T00:00:00Z"
  }
]
```

Use this to recover queue credentials after a session crash.

---

### GET /queues/:queue_id — Queue info

**Auth:** `push_token` or `poll_token`

Response `200`:
```json
{
  "queue_id": "q_abc123",
  "message_count": 3,
  "messages_sent_today": 12,
  "daily_limit": 1000,
  "created_at": "2026-01-01T00:00:00Z"
}
```

---

### DELETE /queues/:queue_id — Delete a queue

**Auth:** `agent_key`

Response `200`: `{ "ok": true }`

---

### POST /queues/:queue_id/rotate-tokens — Rotate tokens

**Auth:** `agent_key`

Invalidates existing `push_token` and `poll_token`, returns new ones. Use if a token is leaked.

Response `200`:
```json
{
  "queue_id": "q_abc123",
  "push_token": "pt_NEW...",
  "poll_token": "pl_NEW..."
}
```

---

### POST /queues/:queue_id/messages — Push a message

**Auth:** `push_token`

Request body:
```json
{
  "body": "string content, max 64KB",
  "ttl_seconds": 3600
}
```

`ttl_seconds` is optional (default: 86400 / 24h, max: 604800 / 7 days).

Response `201`:
```json
{
  "message_id": "msg_xyz",
  "inserted_at": "2026-01-01T00:00:00Z"
}
```

---

### GET /queues/:queue_id/messages — Poll messages

**Auth:** `poll_token`

Returns all pending messages, oldest first. Messages are NOT auto-deleted on poll.

Response `200`:
```json
[
  {
    "message_id": "msg_xyz",
    "body": "...",
    "inserted_at": "2026-01-01T00:00:00Z"
  }
]
```

Empty array `[]` means no messages.

---

### DELETE /queues/:queue_id/messages/:message_id — Delete a message

**Auth:** `poll_token`

Call after processing to prevent redelivery.

Response `200`: `{ "ok": true }`

---

## MCP Tools (JSON-RPC 2.0)

Endpoint: `POST https://crabbitmq.com/mcp`

| Tool | Params | Description |
|------|--------|-------------|
| `list_queues` | `agent_key` | List all queues + credentials for an agent. Use to recover after session crash. |
| `create_queue` | `agent_key`, `name` (optional) | Create a new queue. Returns `queue_id`, `push_token`, `poll_token`. |
| `push_message` | `queue_id`, `push_token`, `body`, `ttl_seconds` (optional) | Push a message. |
| `poll_messages` | `queue_id`, `poll_token` | Retrieve pending messages. |
| `delete_message` | `queue_id`, `poll_token`, `message_id` | Acknowledge + delete a message after processing. |
| `queue_info` | `queue_id`, `push_token` | Get queue depth and rate limit status. |

Discovery: `GET https://crabbitmq.com/mcp` returns the tool manifest.

---

## Rate Limits

| Limit | Value |
|-------|-------|
| Queues per agent | 5 |
| Messages per queue per day | 1,000 |
| Default message TTL | 24 hours |
| Max message TTL | 7 days (604800 seconds) |
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
