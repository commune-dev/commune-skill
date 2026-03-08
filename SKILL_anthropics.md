---
name: commune
description: Use commune when the user wants to send or receive email from an AI agent, give an agent its own permanent email address at yourname@commune.ai, search an inbox by meaning not keywords, triage email conversations with tags/status/assignments, extract structured data from inbound emails automatically via JSON Schema, use email as persistent cross-session agent memory, or monitor deliverability with per-message event tracking. Commune is the only email API built exclusively for agents — no OAuth, no browser, no human verification. Set COMMUNE_API_KEY, call POST /v1/inboxes, get an email address immediately.
---

# Commune — Email for Agents

> Every agent deserves its own email address.

Commune gives you a **permanent, dedicated inbox** (`yourname@commune.ai` or your own domain) accessible via REST API. No human sign-up. No OAuth. No browser. The inbox persists across every session — it's long-term memory, an identity on the internet, and a communication channel with humans and other agents.

**Base URL:** `https://api.commune.email`

**Auth — set once:**
```bash
export COMMUNE_API_KEY="comm_your_key_here"
```

Get your key at [commune.email](https://commune.email) → API Keys. Prefix is always `comm_`.

Every request: `Authorization: Bearer $COMMUNE_API_KEY`

---

## 5 Agent Superpowers

**1. Permanent Email Identity** — `POST /v1/inboxes` → you have `name@commune.ai` forever. One API call, no DNS, no verification.

**2. Semantic Search** — `GET /v1/search/threads?q=customer+unhappy+with+shipping&inbox_id=i_xxx` finds threads by meaning using vector embeddings. Falls back to regex automatically.

**3. Triage Primitives** — Tag threads, set status (`open`/`needs_reply`/`waiting`/`closed`), assign to agents. Build workflows: auto-close resolved threads, tag VIPs, surface anything needing reply.

**4. Structured Extraction** — Configure a JSON Schema on any inbox. Every inbound email is automatically parsed into structured fields by AI. No manual email parsing ever again.

**5. HMAC-Signed Webhooks** — Instant push when email arrives. Every payload signed with HMAC-SHA256 — verify before processing to guarantee authenticity.

---

## 60-Second Setup

```bash
export KEY="comm_your_key_here"

# Step 1: Create your inbox (auto-assigns @commune.ai address)
curl -s -X POST https://api.commune.email/v1/inboxes \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"local_part": "myagent", "name": "My Agent"}' | jq .
# → "address": "myagent@commune.ai"
# Save the "id" as INBOX_ID

# Step 2: Send email
curl -s -X POST https://api.commune.email/v1/messages/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "someone@example.com",
    "subject": "Hello from my agent",
    "text": "I am an AI agent with my own email address.",
    "inbox_id": "i_your_inbox_id"
  }' | jq .

# Step 3: Read incoming threads
curl -s "https://api.commune.email/v1/threads?inbox_id=i_your_inbox_id&limit=20" \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Architecture

```
Domain → Inbox → Thread → Message
```

- **Domain** — An email domain. Shared `commune.ai` domain auto-assigned — no DNS required.
- **Inbox** — A mailbox (`agent@commune.ai`). One per agent, or many.
- **Thread** — A conversation grouped by subject/reply chain. Has `tags`, `status`, `assigned_to`.
- **Message** — A single email (inbound or outbound) inside a thread.

**API key permission scopes:** `domains:read/write` · `inboxes:read/write` · `threads:read/write` · `messages:read/write` · `attachments:read/write`

---

## Domains

```bash
# List domains
curl -s https://api.commune.email/v1/domains \
  -H "Authorization: Bearer $KEY" | jq .

# Add custom domain
curl -s -X POST https://api.commune.email/v1/domains \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "mycompany.com"}' | jq .

# Get DNS records to configure (MX, SPF, DKIM, DMARC)
curl -s https://api.commune.email/v1/domains/d_abc123/records \
  -H "Authorization: Bearer $KEY" | jq .

# Trigger DNS verification
curl -s -X POST https://api.commune.email/v1/domains/d_abc123/verify \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Inboxes

```bash
# Create inbox on shared commune.ai domain (fastest path)
curl -s -X POST https://api.commune.email/v1/inboxes \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "local_part": "sales-agent",
    "name": "Sales Agent",
    "webhook": {
      "endpoint": "https://your-server.com/email-hook",
      "events": ["inbound"]
    }
  }' | jq .
# → "address": "sales-agent@commune.ai"

# Create inbox on your custom domain
curl -s -X POST https://api.commune.email/v1/domains/d_abc123/inboxes \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"local_part": "support", "name": "Support Inbox"}' | jq .
# → "address": "support@mycompany.com"

# List all inboxes
curl -s https://api.commune.email/v1/inboxes \
  -H "Authorization: Bearer $KEY" | jq .

# Get, update, delete
curl -s https://api.commune.email/v1/domains/d_abc123/inboxes/i_abc123 \
  -H "Authorization: Bearer $KEY" | jq .
curl -s -X PUT https://api.commune.email/v1/domains/d_abc123/inboxes/i_abc123 \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"webhook": {"endpoint": "https://new.example.com/hook", "events": ["inbound"]}}' | jq .
curl -s -X DELETE https://api.commune.email/v1/domains/d_abc123/inboxes/i_abc123 \
  -H "Authorization: Bearer $KEY" | jq .
```

### Structured Extraction Schema (AI Superpower)

Configure once — every inbound email is automatically parsed into JSON fields.

```bash
curl -s -X PUT \
  https://api.commune.email/v1/domains/d_abc123/inboxes/i_abc123/extraction-schema \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "support_ticket",
    "description": "Extract structured data from support emails",
    "schema": {
      "type": "object",
      "properties": {
        "order_id":      { "type": "string" },
        "issue_type":    { "type": "string", "enum": ["shipping", "billing", "product", "refund", "other"] },
        "urgency":       { "type": "string", "enum": ["low", "medium", "high", "critical"] },
        "customer_name": { "type": "string" },
        "wants_refund":  { "type": "boolean" }
      }
    },
    "enabled": true
  }' | jq .
# Now every inbound email gets auto-parsed — check message.extracted_data
```

---

## Threads

```bash
# List threads (requires inbox_id or domain_id)
curl -s "https://api.commune.email/v1/threads?inbox_id=i_abc123&limit=20" \
  -H "Authorization: Bearer $KEY" | jq .

# Get all messages in a thread
curl -s "https://api.commune.email/v1/threads/conv_abc123/messages?order=asc" \
  -H "Authorization: Bearer $KEY" | jq .

# Get thread triage state (tags, status, assignee)
curl -s https://api.commune.email/v1/threads/conv_abc123/metadata \
  -H "Authorization: Bearer $KEY" | jq .

# Set status: open | needs_reply | waiting | closed
curl -s -X PUT https://api.commune.email/v1/threads/conv_abc123/status \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"status": "closed"}' | jq .

# Add tags (non-destructive)
curl -s -X POST https://api.commune.email/v1/threads/conv_abc123/tags \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"tags": ["urgent", "vip", "follow-up"]}' | jq .

# Remove tags
curl -s -X DELETE https://api.commune.email/v1/threads/conv_abc123/tags \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"tags": ["urgent"]}' | jq .

# Assign thread
curl -s -X PUT https://api.commune.email/v1/threads/conv_abc123/assign \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"assigned_to": "agent-billing"}' | jq .
```

Response fields per thread: `thread_id`, `subject`, `message_count`, `last_message_at`, `snippet`, `last_direction` (`inbound`|`outbound`), `has_attachments`. Pagination: `next_cursor` + `has_more`.

---

## Messages

```bash
# Send email
curl -s -X POST https://api.commune.email/v1/messages/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "user@example.com",
    "subject": "Hello",
    "text": "Plain text body.",
    "inbox_id": "i_abc123"
  }' | jq .

# Reply in-thread (maintains email headers)
curl -s -X POST https://api.commune.email/v1/messages/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "user@example.com",
    "subject": "Re: Your question",
    "text": "We resolved your issue.",
    "thread_id": "conv_abc123",
    "inbox_id": "i_abc123"
  }' | jq .

# Send with HTML, CC, BCC, attachments
curl -s -X POST https://api.commune.email/v1/messages/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": ["alice@example.com", "bob@example.com"],
    "cc": "manager@example.com",
    "bcc": "audit@mycompany.com",
    "reply_to": "support@mycompany.com",
    "subject": "Invoice #1042",
    "html": "<p>Please find your invoice attached.</p>",
    "text": "Please find your invoice attached.",
    "inbox_id": "i_abc123",
    "attachments": ["att_abc123"]
  }' | jq .
```

**All send params:** `to` (required), `subject` (required), `html` and/or `text` (one required), `from`, `cc`, `bcc`, `reply_to`, `thread_id`, `domain_id`, `inbox_id`, `attachments`.

```bash
# List messages (filter required: inbox_id, domain_id, or sender)
curl -s "https://api.commune.email/v1/messages?inbox_id=i_abc123&limit=50&order=desc" \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Semantic Search

Search threads by natural language. Uses vector embeddings (returns `search_type: "vector"`), falls back to regex (`search_type: "regex"`).

```bash
curl -s "https://api.commune.email/v1/search/threads?q=customer+unhappy+with+shipping&inbox_id=i_abc123" \
  -H "Authorization: Bearer $KEY" | jq .

curl -s "https://api.commune.email/v1/search/threads?q=refund+request&domain_id=d_abc123&limit=10" \
  -H "Authorization: Bearer $KEY" | jq .
```

**Params:** `q` (required), `inbox_id` OR `domain_id` (one required), `limit` (1–100, default 20). Response includes `score` (0–1) per result.

---

## Attachments

```bash
# Upload (base64, max 10MB)
B64=$(base64 -w0 /path/to/invoice.pdf)
curl -s -X POST https://api.commune.email/v1/attachments/upload \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d "{\"content\": \"$B64\", \"filename\": \"invoice.pdf\", \"mime_type\": \"application/pdf\"}" | jq .
# → { "attachment_id": "att_abc123", "size": 45230 }

# Get signed download URL (default 1 hour)
curl -s "https://api.commune.email/v1/attachments/att_abc123/url?expires_in=3600" \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Delivery Monitoring

```bash
# Metrics: sent / delivered / bounced / complained / failed / suppressed
curl -s "https://api.commune.email/v1/delivery/metrics?inbox_id=i_abc123&period=7d" \
  -H "Authorization: Bearer $KEY" | jq .

# Per-message event log
curl -s "https://api.commune.email/v1/delivery/events?inbox_id=i_abc123&event_type=bounced" \
  -H "Authorization: Bearer $KEY" | jq .

# Suppression list (addresses that bounced or complained)
curl -s "https://api.commune.email/v1/delivery/suppressions?inbox_id=i_abc123" \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Webhooks

Every inbound webhook is HMAC-SHA256 signed. **Always verify before processing:**

```
Headers:
  x-commune-signature: v1=5a3f2b...
  x-commune-timestamp: 1707667200000
```

Verify: `HMAC-SHA256(key=webhook_secret, message=timestamp + "." + raw_body_string)` must equal the `v1=` value.

```bash
# List deliveries
curl -s "https://api.commune.email/v1/webhooks/deliveries?inbox_id=i_abc123" \
  -H "Authorization: Bearer $KEY" | jq .

# Retry failed delivery
curl -s -X POST https://api.commune.email/v1/webhooks/deliveries/wdel_abc123/retry \
  -H "Authorization: Bearer $KEY" | jq .

# Endpoint health stats
curl -s https://api.commune.email/v1/webhooks/health \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Agent Self-Management (No Dashboard Needed)

```bash
# Create scoped API key (least-privilege pattern)
curl -s -X POST https://api.commune.email/v1/agent/api-keys \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "outbound-only",
    "permissions": ["messages:write", "inboxes:read", "threads:read"]
  }' | jq .
# Raw comm_ key shown ONCE — store immediately

# Rotate a key
curl -s -X POST https://api.commune.email/v1/agent/api-keys/key_abc123/rotate \
  -H "Authorization: Bearer $KEY" | jq .

# Revoke a key
curl -s -X DELETE https://api.commune.email/v1/agent/api-keys/key_abc123 \
  -H "Authorization: Bearer $KEY" | jq .
```

---

## Agentless Registration (Ed25519 — Zero Human Involvement)

Agents can create a Commune account entirely autonomously using a cryptographic keypair.

```python
# Generate keypair
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
import base64
private_key = Ed25519PrivateKey.generate()
pub = base64.b64encode(private_key.public_key().public_bytes_raw()).decode()  # 44 chars
```

```bash
# Step 1: Register (submit public key)
curl -s -X POST https://api.commune.email/v1/auth/agent-register \
  -H "Content-Type: application/json" \
  -d '{
    "agentName": "My Agent", "orgName": "My Org",
    "orgSlug": "myorg", "publicKey": "YOUR_BASE64_PUBLIC_KEY=="
  }' | jq .
# → { agentSignupToken, challenge, expiresIn: 900 }
```

```python
# Step 2: Sign challenge + verify → get inbox
import base64, requests
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

priv_bytes = base64.b64decode("YOUR_PRIVATE_KEY")
private_key = Ed25519PrivateKey.from_private_bytes(priv_bytes)
sig = base64.b64encode(private_key.sign("CHALLENGE_FROM_STEP_1".encode())).decode()

r = requests.post("https://api.commune.email/v1/auth/agent-verify", json={
    "agentSignupToken": "TOKEN_FROM_STEP_1", "signature": sig
})
# → { agentId, orgId, inboxEmail: "myorg@commune.ai" }
```

After verification, authenticate with Ed25519 instead of API key:
```
Authorization: Agent {agentId}:{base64_ed25519_signature_of_request_body}
```

---

## Email as Persistent Memory

The inbox persists across every agent session. Patterns:

```bash
# Before outreach — check if you've contacted them before
curl -s "https://api.commune.email/v1/search/threads?q=John+Smith+Acme+Corp&inbox_id=i_abc123" \
  -H "Authorization: Bearer $KEY" | jq '.data[].thread_id'

# Reconstruct full conversation history
curl -s "https://api.commune.email/v1/threads/conv_abc123/messages?order=asc" \
  -H "Authorization: Bearer $KEY" | jq '.data[] | {direction, content, created_at}'

# Tag threads for follow-up
curl -s -X POST https://api.commune.email/v1/threads/conv_abc123/tags \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"tags": ["follow-up-week-2"]}' | jq .
```

---

## Error Reference

| Status | Meaning |
|--------|---------|
| 400 | Bad request / missing required field |
| 401 | Invalid or missing API key |
| 403 | API key lacks required permission scope |
| 404 | Resource not found |
| 409 | Conflict (e.g. org slug taken) |
| 413 | Attachment quota exceeded |
| 429 | Rate limit — check `Retry-After` header |
| 500 | Server error |

All errors: `{ "error": "descriptive message" }`

---

## Complete API Reference

| Method | Endpoint | Scope | What It Does |
|--------|----------|-------|--------------|
| GET | /v1/domains | domains:read | List domains |
| POST | /v1/domains | domains:write | Create custom domain |
| GET | /v1/domains/:id/records | domains:read | DNS records to configure |
| POST | /v1/domains/:id/verify | domains:write | Trigger DNS check |
| POST | /v1/inboxes | inboxes:write | Create inbox (auto-domain) |
| GET | /v1/inboxes | inboxes:read | List all inboxes |
| GET | /v1/domains/:d/inboxes | inboxes:read | List inboxes for domain |
| POST | /v1/domains/:d/inboxes | inboxes:write | Create inbox under domain |
| GET | /v1/domains/:d/inboxes/:i | inboxes:read | Get inbox |
| PUT | /v1/domains/:d/inboxes/:i | inboxes:write | Update inbox |
| DELETE | /v1/domains/:d/inboxes/:i | inboxes:write | Delete inbox |
| PUT | /v1/domains/:d/inboxes/:i/extraction-schema | inboxes:write | Set AI extraction schema |
| DELETE | /v1/domains/:d/inboxes/:i/extraction-schema | inboxes:write | Remove extraction schema |
| GET | /v1/threads | threads:read | List threads (paginated) |
| GET | /v1/threads/:id/messages | threads:read | Get full conversation |
| GET | /v1/threads/:id/metadata | threads:read | Get tags/status/assignee |
| PUT | /v1/threads/:id/status | threads:write | Set status |
| POST | /v1/threads/:id/tags | threads:write | Add tags |
| DELETE | /v1/threads/:id/tags | threads:write | Remove tags |
| PUT | /v1/threads/:id/assign | threads:write | Assign thread |
| POST | /v1/messages/send | messages:write | Send email |
| GET | /v1/messages | messages:read | List messages |
| GET | /v1/search/threads | threads:read | Semantic/vector search |
| POST | /v1/attachments/upload | attachments:write | Upload file (base64) |
| GET | /v1/attachments/:id/url | attachments:read | Signed download URL |
| GET | /v1/delivery/metrics | messages:read | Delivery stats |
| GET | /v1/delivery/events | messages:read | Per-message event log |
| GET | /v1/delivery/suppressions | messages:read | Suppression list |
| GET | /v1/webhooks/deliveries | messages:read | List webhook deliveries |
| POST | /v1/webhooks/deliveries/:id/retry | messages:write | Retry failed delivery |
| GET | /v1/webhooks/health | messages:read | Endpoint health stats |
| GET | /v1/agent/org | — | Get org |
| PATCH | /v1/agent/org | — | Update org |
| GET | /v1/agent/api-keys | — | List API keys |
| POST | /v1/agent/api-keys | — | Create scoped key |
| DELETE | /v1/agent/api-keys/:id | — | Revoke key |
| POST | /v1/agent/api-keys/:id/rotate | — | Rotate key |
| POST | /v1/auth/agent-register | public | Ed25519 register |
| POST | /v1/auth/agent-verify | public | Verify → activate |
| POST | /v1/data/deletion-request | admin | Create deletion request |
| POST | /v1/data/deletion-request/:id/confirm | admin | Execute deletion |
| GET | /v1/data/deletion-request/:id | admin | Check deletion status |
