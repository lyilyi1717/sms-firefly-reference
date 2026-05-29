# SMS → Firefly III — Project Reference

Saudi bank SMS messages are received via MacroDroid on Android → forwarded to an n8n webhook → categorized by a 3-tier AI pipeline → posted to Firefly III as transactions → confirmed via Telegram.

**Last updated:** 2026-05-29  
**Status:** Production (running every 30s)

---

## Quick Links

| Service | URL |
|---------|-----|
| n8n workflow editor | http://192.168.8.10:5678 |
| Firefly III | http://192.168.8.10:8181 |
| Dashboard | http://192.168.8.10:8283 |
| Ollama | http://192.168.8.10:11434 |
| Qdrant | http://192.168.8.10:6333 |

All services run on Unraid at `192.168.8.10`.

---

## Files in This Repo

| File | Purpose |
|------|---------|
| `README.md` | This file — overview and quick links |
| `ARCHITECTURE.md` | 3-tier pipeline, n8n workflow node flow |
| `DATABASE.md` | PostgreSQL table schemas |
| `WORKFLOW_NODES.md` | n8n node code patterns, IF/Postgres/HTTP structure |
| `OPERATIONS.md` | Deployment, dashboard rebuild, workflow export |
| `KNOWN_ISSUES.md` | Bug history, fixes, active quirks |
| `PATCH_HISTORY.md` | Chronological record of all deployed patches |
| `CONFIG.template.md` | Credential placeholders (fill in SECRETS.md locally) |
| `SECRETS.md` | **Local only — gitignored** — actual API keys |

---

## Credential Setup

Copy `CONFIG.template.md` → `SECRETS.md` and fill in your values.  
`SECRETS.md` is in `.gitignore` and will never be committed.

---

## Workflow IDs

| ID | Name | Purpose |
|----|------|---------|
| `IUliVtDElkxMnl9t` | SMS Categorize Firefly | Main processor (schedule 30s) |
| `smscorrection01` | SMS Correction | Telegram callback handler |
| `W6vGjhatADpLi4jU` | SMS Receiver | Webhook receiver → enqueues |

---

## Infrastructure Stack

- **n8n** — workflow automation (self-hosted, Unraid Docker)
- **PostgreSQL 18** — queue + transaction log + learned patterns (container: `n8n-postgresql18`, port 5434)
- **Firefly III** — personal finance app, receives categorized transactions via REST API
- **Ollama / gemma3:1b** — local LLM for AI-tier categorization (~42s per SMS)
- **Qdrant** — vector DB for embedding corpus (upsert-only; search bypassed — see KNOWN_ISSUES)
- **Telegram** — confirmation + correction keyboard sent after each transaction
- **MacroDroid** — Android automation; intercepts SMS → HTTP POST to n8n webhook
