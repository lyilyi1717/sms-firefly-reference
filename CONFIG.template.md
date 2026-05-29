# Configuration Template

Copy this file to `SECRETS.md` and fill in the actual values.  
`SECRETS.md` is gitignored — it will never be committed.

---

## Infrastructure

| Service | URL | Notes |
|---------|-----|-------|
| n8n | http://192.168.8.10:5678 | Change IP if Unraid moves |
| Firefly III | http://192.168.8.10:8181 | |
| PostgreSQL | 192.168.8.10:5434 | container: n8n-postgresql18 |
| Ollama | http://192.168.8.10:11434 | |
| Qdrant | http://192.168.8.10:6333 | collection: txn_categories |
| Dashboard | http://192.168.8.10:8283 | |

---

## API Keys

```
N8N_API_KEY=<your-n8n-api-key>
FIREFLY_TOKEN=<your-firefly-personal-access-token>
TELEGRAM_BOT_TOKEN=<your-telegram-bot-token>
TELEGRAM_CHAT_ID=<your-telegram-chat-id>
```

### How to get each:

**n8n API key:**  
n8n → Settings → API → Create API Key

**Firefly III token:**  
Firefly III → Profile → OAuth → Personal Access Tokens → Create New Token

**Telegram bot token:**  
BotFather → /newbot → copy token

**Telegram chat ID:**  
Send a message to your bot → `https://api.telegram.org/bot<TOKEN>/getUpdates` → find `chat.id`

---

## PostgreSQL

```
PG_HOST=192.168.8.10
PG_PORT=5434
PG_USER=admin
PG_PASSWORD=<your-postgres-password>
PG_DATABASE=n8n
```

n8n credential ID for the Postgres connection: `qx7cEqJpZFgDgCax` (name: "Postgres account")  
This ID is hardcoded in workflow nodes — do not change without updating all Postgres nodes.

---

## Workflow IDs

```
WORKFLOW_MAIN=IUliVtDElkxMnl9t        # SMS Categorize Firefly (scheduler 30s)
WORKFLOW_CORRECTION=smscorrection01    # Telegram callback handler
WORKFLOW_RECEIVER=W6vGjhatADpLi4jU    # Webhook SMS receiver
```
