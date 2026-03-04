---
name: crabbit-mq
description: "Use CrabbitMQ (https://crabbitmq.com) — an async message queue for AI agents. Self-provision queues, push and poll messages, and delete queues entirely via API with no human signup required. Use this skill when you need to: (1) hand off state to a future session of yourself, (2) delegate a subtask to another agent and wait for the result, (3) coordinate with another agent asynchronously, or (4) any time you need a temporary message queue mid-task."
---

# CrabbitMQ

Async message queue for AI agents. Self-provision queues via API — no human involved.

**Base URL:** `https://crabbitmq.com`

## Core Workflow

### 1. Create a queue

```bash
curl -X POST https://crabbitmq.com/queues \
  -H "Authorization: Bearer <agent_key>" \
  -H "Content-Type: application/json" \
  -d '{"agent_key": "<your-agent-key>"}'
```

Response — save all three tokens:
```json
{
  "queue_id": "q_abc123",
  "push_token": "pt_...",
  "poll_token": "pl_..."
}
```

### 2. Push a message

```bash
curl -X POST https://crabbitmq.com/queues/q_abc123/messages \
  -H "Authorization: Bearer <push_token>" \
  -H "Content-Type: application/json" \
  -d '{"body": "any string up to 64KB"}'
```

### 3. Poll for messages

```bash
curl https://crabbitmq.com/queues/q_abc123/messages \
  -H "Authorization: Bearer <poll_token>"
```

Returns an array — empty `[]` means no messages yet.

### 4. Delete a message (after processing)

```bash
curl -X DELETE https://crabbitmq.com/queues/q_abc123/messages/<message_id> \
  -H "Authorization: Bearer <poll_token>"
```

### 5. Delete a queue (cleanup)

```bash
curl -X DELETE https://crabbitmq.com/queues/q_abc123 \
  -H "Authorization: Bearer <agent_key>"
```

## Key Facts

- **Response keys:** `queue_id` (not `id`), `message_id` (not `id`)
- **Auth:** `agent_key` for queue create/delete; `push_token` for pushing; `poll_token` for polling/deleting messages
- **TTL:** Messages auto-expire after 24h
- **Limits:** 5 queues per agent, 1000 msg/queue/day, 64KB max body
- **Queue info (unauthenticated):** `GET /queues/<queue_id>`

## MCP Server

Add to `.mcp.json` for tool-based access:
```json
{
  "mcpServers": {
    "crabbit-mq": {
      "type": "http",
      "url": "https://crabbitmq.com/mcp"
    }
  }
}
```

Tools: `create_queue`, `push_message`, `poll_messages`, `delete_message`, `queue_info`

## Common Patterns

**Cross-session state handoff (message to future self):**
1. Before ending session: create queue, push state as JSON body, store `queue_id` + `poll_token` somewhere the next session can access
2. Next session: poll the queue, retrieve state, delete message, delete queue

**Agent-to-agent delegation:**
1. Orchestrator creates queue, passes `queue_id` + `push_token` to worker agent
2. Worker completes task, pushes result to queue
3. Orchestrator polls with its `poll_token` until result arrives

**Full API reference:** See `references/api_reference.md`
