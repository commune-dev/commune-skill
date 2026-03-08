# Commune Skill — Marketplace Distribution Plan
*Generated: 2026-02-20 | Goal: Maximum agent-ecosystem coverage*

---

## Status Overview

| Marketplace | Type | Status | Priority |
|---|---|---|---|
| ClawHub (clawhub.ai) | Upload form | ✅ v1.0.1 LIVE | P0 |
| Zo Computer (zocomputer/skills) | GitHub PR | ✅ PR #16 open | P0 |
| communedotemail/commune-skill | GitHub repo (canonical) | 🔲 Create | P0 |
| anthropic/skills | GitHub PR | 🔲 Pending | P1 |
| microsoft/skills | GitHub PR | 🔲 Pending | P1 |
| skills.sh | Auto-discovery (needs public repo) | 🔲 After repo created | P1 |
| skillsmp.com | Auto-indexed from GitHub | 🔲 After repo created | P1 |
| skillhub.club | Auto-indexed from GitHub | 🔲 After repo created | P1 |
| mcp.so (chatmcp/mcp-directory) | GitHub PR | 🔲 Pending | P2 |
| awesome-openclaw-skills | GitHub PR | 🔲 Pending | P2 |
| awesome-agent-skills repos | GitHub PR (multiple) | 🔲 Pending | P2 |
| skilldock.io | Web form / GitHub | 🔲 Pending | P2 |
| claudeskills.info | Web form | 🔲 Pending | P2 |
| skillhub.club | Web form | 🔲 Pending | P2 |
| SkillDock | Web form | 🔲 Pending | P2 |

---

## The Canonical GitHub Repo (MUST DO FIRST)

Before submitting to auto-indexed marketplaces, commune needs a public GitHub repo.

**Repo to create:** `communedotemail/commune-skill`

Structure:
```
communedotemail/commune-skill/
├── SKILL.md          ← main skill file (already written)
├── INSTALL.md        ← installation guide (already written)
├── scripts/
│   └── commune.py    ← CLI helper (already written)
└── references/
    └── api.md        ← API reference (already written)
```

Once live at `https://github.com/communedotemail/commune-skill`:
- **skills.sh** auto-discovers it (Vercel's agent skills directory)
- **skillsmp.com** auto-indexes it (200K+ skill marketplace)
- **skillhub.club** auto-indexes it (7K+ AI-evaluated skills)
- Anyone can install via `openclaw skills add communedotemail/commune-skill`

---

## Tier 1 — PR-Based GitHub Submissions

### 1. anthropic/skills
- **Repo:** https://github.com/anthropics/skills
- **Action:** Fork → create `skills/commune/SKILL.md` → PR
- **Why:** Official Anthropic repo — highest credibility

### 2. microsoft/skills
- **Repo:** https://github.com/microsoft/skills
- **Action:** Fork → create `skills/commune/` folder → PR
- **Why:** Microsoft Foundry ecosystem reach — enterprise agents

### 3. mcp.so directory
- **Repo:** https://github.com/chatmcp/mcp-directory
- **Action:** Add commune to directory list (as a tool/REST API wrapper)
- **Note:** Not an MCP server, but can be listed as a REST API tool

---

## Tier 2 — Web Form Submissions

### 4. skillsmp.com
- **URL:** https://skillsmp.com/submit (or auto-indexed)
- **Action:** Submit GitHub repo URL after communedotemail/commune-skill is live

### 5. skillhub.club
- **URL:** https://www.skillhub.club/submit
- **Action:** Submit GitHub repo URL

### 6. skilldock.io
- **URL:** https://skilldock.io/
- **Action:** Publish via their interface

### 7. claudeskills.info
- **URL:** https://claudeskills.info/
- **Action:** Web form submission

---

## Tier 3 — Awesome List PRs (Community Discovery)

These don't host skills themselves — they're curated GitHub lists that serve as discovery.
Being on these lists means agents searching for tools will find commune.

### 8. VoltAgent/awesome-openclaw-skills
- **Repo:** https://github.com/VoltAgent/awesome-openclaw-skills
- **Entry to add:** `- [commune](https://github.com/communedotemail/commune-skill) — Agent-native email inbox with full send/receive, semantic search, triage, and webhooks`

### 9. punkpeye/awesome-mcp-servers
- **Repo:** https://github.com/punkpeye/awesome-mcp-servers
- **Entry:** List commune as an email API integration

### 10. travisvn/awesome-claude-skills (if real)
- **Repo:** https://github.com/travisvn/awesome-claude-skills
- **Entry:** Standard commune skill entry

### 11. heilcheng/awesome-agent-skills
- **Repo:** https://github.com/heilcheng/awesome-agent-skills
- **Entry:** Standard commune skill entry

---

## The OpenClaw Install Command (Fixed)
```bash
# Correct install:
openclaw skills add shanjairaj7/commune

# After communedotemail repo is live:
openclaw skills add communedotemail/commune-skill
```

---

## SKILL.md Content for Copy-Paste
The SKILL.md is at: `/commune-skill/SKILL.md` (802 lines, fully self-contained)

For any marketplace that needs the content pasted directly, use the full SKILL.md.

---

## Notes on MCP Registries (smithery.ai, glama.ai, registry.modelcontextprotocol.io)

These require an actual MCP server (not just a SKILL.md). Commune doesn't currently have an MCP server implementation.

**Future task:** Build `commune-mcp` package that wraps the REST API as an MCP server, then submit to:
- registry.modelcontextprotocol.io
- smithery.ai
- glama.ai/mcp/servers

This would unlock a whole additional ecosystem of Claude Desktop / Cursor / Continue.dev users.
