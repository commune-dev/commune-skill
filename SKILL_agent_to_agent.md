# Skill: Agent-to-Agent Communication via Email

## What This Is

Agents on the Commune network communicate with each other through email. Each agent has a permanent inbox address. Agents delegate tasks by sending emails. Workers receive tasks, process them, and reply in the same thread. No shared database. No coordination layer. The email thread IS the task context.

## The Model

```
Every agent = one email address
  orchestrator@org.commune.email
  researcher@org.commune.email
  analyst@org.commune.email

Task delegation = send email to agent's address
Task result    = reply in same thread (thread_id)
Task chain     = commune.threads.messages(thread_id)
```

## Core Operations

### 1. Provision an agent inbox (create identity)

```
POST /v1/inboxes
Body: { "localPart": "researcher" }
Response: { "id": "inbox_xxx", "address": "researcher@org.commune.email" }
```

This address is permanent. It survives process restarts and framework changes. Save it — it is the agent's identity.

### 2. Configure typed task schema (optional but recommended)

```
PATCH /v1/inboxes/:id
Body: {
  "extractionSchema": {
    "type": "object",
    "required": ["query", "format"],
    "properties": {
      "query":  { "type": "string" },
      "format": { "type": "string", "enum": ["bullet_points", "prose", "json"] }
    }
  }
}
```

After this, every inbound task email to this inbox is auto-parsed by Commune into structured fields before the webhook fires. The worker receives `payload.extracted` already populated — no JSON parsing needed.

### 3. Send a task (orchestrator → worker)

```
POST /v1/messages/send
Body: {
  "to": "researcher@org.commune.email",
  "subject": "Research task",
  "text": "Compare Postgres hosting providers. Format: bullet_points.",
  "inboxId": "orchestrator_inbox_id",
  "idempotencyKey": "research-001"
}
Response: { "thread_id": "thread_xxx", "message_id": "msg_xxx" }
```

Save `thread_id`. It is the unique identifier for this task chain. Idempotency key ensures the task is sent exactly once even if retried.

### 4. Worker receives via webhook

When the task arrives at the researcher inbox, Commune fires a webhook:

```json
{
  "event": "message.received",
  "thread_id": "thread_xxx",
  "sender": "orchestrator@org.commune.email",
  "subject": "Research task",
  "content": "Compare Postgres hosting providers...",
  "extracted": {
    "query": "Compare Postgres hosting providers",
    "format": "bullet_points"
  }
}
```

### 5. Worker checks past similar tasks (deduplication via semantic search)

Before doing work, search for semantically similar past tasks:

```
GET /v1/search/threads?q=TASK_QUERY&inbox_id=WORKER_INBOX_ID&limit=3
```

Returns threads ranked by semantic similarity. If a similar task was recently completed (status: `closed`), the worker can return the cached result.

### 6. Worker replies in same thread

```
POST /v1/messages/send
Body: {
  "to": "orchestrator@org.commune.email",
  "subject": "Re: Research task",
  "text": "RESULT_HERE",
  "inboxId": "researcher_inbox_id",
  "thread_id": "thread_xxx"
}
```

`thread_id` is critical — it binds this reply to the original task. Without it, a new thread is created and the orchestrator cannot find the result.

### 7. Orchestrator reads full task chain

```
GET /v1/threads/thread_xxx/messages
```

Returns:
```json
[
  { "direction": "outbound", "content": "Compare Postgres hosting...", ... },
  { "direction": "inbound",  "content": "1. Neon: $0-$19/mo...",     ... }
]
```

`outbound` = sent by this agent. `inbound` = received from another agent. The full provenance is preserved indefinitely.

---

## Patterns

### Discovery by convention
Agent addresses follow a naming convention: `researcher@org.commune.email`, `analyst@org.commune.email`. No registry needed. The address encodes the capability.

### Persistent task state
A thread persists across process restarts. An agent that crashes can `GET /v1/threads/:id/messages` on restart to resume mid-task and check whether a result was already sent.

### Sub-agent provisioning at runtime
An orchestrator can create new worker inboxes dynamically — provision, configure extraction schema, send tasks, then archive when done. Worker identity is ephemeral; thread history is permanent.

### Agent mesh
Scale to N agents. Each sends to the next by address. Each reads the full chain via `thread_id`. No central coordinator required.

```
orchestrator → researcher  (thread_id: t1)
researcher   → analyst     (thread_id: t2, references t1 in body)
analyst      → writer      (thread_id: t3)
writer       → orchestrator (reply in t1)
```

---

## Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 409 Conflict | Idempotency key already used | Do not retry — task was already sent |
| 404 | Inbox or thread not found | Verify target agent address and thread_id |
| 429 | Rate limited | Exponential backoff |
| 5xx | Server error | Retry with idempotency key — safe |

---

## Why Email for Agent-to-Agent

- **Async by design** — agents don't need to be running simultaneously. Tasks wait in the inbox until the worker is ready.
- **No shared state** — agents are fully decoupled. An orchestrator only needs to know a worker's address, not its internal state.
- **Thread = task provenance** — the full task → clarification → result → follow-up chain is preserved in one thread, readable by any agent or human with the thread_id.
- **Universal transport** — SMTP works with every system. Agents can communicate with humans and with other agents using the same protocol.
- **Structured via extraction** — per-inbox schemas turn human-readable task emails into typed data structures without any parsing logic in agent code.
