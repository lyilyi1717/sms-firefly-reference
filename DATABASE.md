# Database Reference

**Connection:** PostgreSQL at `192.168.8.10:5434`  
**Database:** `n8n`  
**Container:** `n8n-postgresql18`  
Credentials in `SECRETS.md` (local) / `CONFIG.template.md` (template).

---

## Tables

### `sms_queue`

Incoming SMS items. Scheduler reads from here every 30s.

```sql
sms_id          TEXT UNIQUE,          -- MacroDroid-generated: "<unix_ms>_<sender>"
sender          TEXT,                 -- e.g. "SNB-AlAhli", "AlRajhi"
body            TEXT,                 -- raw SMS body (Arabic + English)
received_at     TEXT,                 -- unix ms as TEXT (not TIMESTAMPTZ!)
queued_at       TIMESTAMPTZ,
processed       BOOL DEFAULT false,
status          TEXT,                 -- pending | ai_needed | processing | done | non_txn | error | firefly_error
retry_count     INT DEFAULT 0,
last_attempted_at TIMESTAMPTZ,
processed_at    TIMESTAMPTZ
```

**Cast received_at:** `to_timestamp(received_at::numeric / 1000)`  
**Status notes:**
- `ai_needed` = legacy alias for `pending`; treated identically by scheduler
- `processing` items older than 10min with retry_count<3 are re-picked (stall recovery)
- `error` = hit max retries (retry_count reached 3)
- `firefly_error` = Firefly III POST failed; `processed=false` so scheduler auto-retries up to 3x

---

### `transactions`

Confirmed transactions logged after Firefly POST succeeds.

```sql
sms_id           TEXT,
sender           TEXT,
raw_body         TEXT,
amount           NUMERIC,
merchant         TEXT,
account_last4    TEXT,
bank_name        TEXT,
category         TEXT,
transaction_type TEXT,             -- debit | credit
confidence       FLOAT,
tier             TEXT,             -- regex | merchant_lookup | ai | places
firefly_id       TEXT,             -- Firefly transaction ID
needs_review     BOOL,
reviewed         BOOL,
tg_message_id    TEXT,
tg_chat_id       TEXT,
created_at       TIMESTAMPTZ
```

**tier='places'** = 261 legacy transactions from old Google Places API implementation. Nodes removed; label kept as historical.

---

### `regex_patterns`

Auto-learned and manually corrected per-sender regex patterns.

```sql
sender           TEXT,
pattern          TEXT,             -- regex string matched against SMS body
category         TEXT,
transaction_type TEXT,
hit_count        INT,
created_at       TIMESTAMPTZ,
UNIQUE(sender, pattern)
```

- Inserted by `Postgres: Auto Pattern` after every confirmed transaction
- Corrected patterns written by Telegram correction flow
- Matched by `Code: Test Patterns` at confidence=0.95

---

### `merchant_categories`

Fuzzy merchant-to-category mapping using pg_trgm similarity.

```sql
merchant         TEXT,
category         TEXT,
hits             INT DEFAULT 1,
source           TEXT,             -- seed | auto | correction
last_seen        TIMESTAMPTZ,
UNIQUE(merchant, category)
-- INDEX: GIN on merchant gin_trgm_ops (pg_trgm)
```

- 752 merchants seeded from transaction history
- `auto`: hits+1 per confirmed transaction
- `correction`: hits+3 (higher weight for human corrections)
- Match threshold: `similarity(merchant, input) > 0.35`
- Length guard: `LENGTH(merchant) >= 3`

**pg_trgm query (UNION ALL sentinel — always returns exactly 1 row):**

```sql
WITH match AS (
  SELECT category, hits,
    GREATEST(similarity(merchant, '{{ $json.merchant_lookup_key_safe }}'),
             similarity(merchant, '{{ $json.merchant_raw_safe }}')) AS score
  FROM merchant_categories
  WHERE GREATEST(similarity(merchant, '{{ $json.merchant_lookup_key_safe }}'),
                 similarity(merchant, '{{ $json.merchant_raw_safe }}')) > 0.35
    AND LENGTH(merchant) >= 3
    AND '{{ $json.merchant_lookup_key_safe }}' != ''
  ORDER BY score DESC, hits DESC
  LIMIT 1
)
SELECT * FROM match
UNION ALL
SELECT NULL::text AS category, 0 AS hits, 0.0 AS score
WHERE NOT EXISTS (SELECT 1 FROM match)
LIMIT 1
```

> **CRITICAL:** The UNION ALL sentinel is required. Without it, a 0-row result silently kills all downstream nodes in n8n — Ollama never runs, items stall in `processing`, hit max retries, become `error`.

---

### `non_txn_patterns`

Learned non-transaction patterns (OTPs, promos, balance alerts, etc.).

```sql
pattern          TEXT UNIQUE,
description      TEXT,
created_at       TIMESTAMPTZ
```

- Checked by `Postgres: Learned Non-Txn` before any categorization
- New patterns added when user clicks "Not a Txn" in Telegram

---

## Useful Queries

```sql
-- Queue status summary
SELECT status, COUNT(*) FROM sms_queue GROUP BY status ORDER BY COUNT(*) DESC;

-- Pending items
SELECT sms_id, sender, status, retry_count, queued_at
FROM sms_queue WHERE processed=false ORDER BY queued_at;

-- Recent transactions by tier
SELECT tier, COUNT(*), AVG(confidence) FROM transactions
WHERE created_at > NOW()-INTERVAL '7 days' GROUP BY tier;

-- Top merchants
SELECT merchant, category, hits, source FROM merchant_categories
ORDER BY hits DESC LIMIT 20;

-- Stalled items (may need manual reset)
SELECT * FROM sms_queue
WHERE status='processing' AND last_attempted_at < NOW()-INTERVAL '15 minutes';

-- Reset stalled items
UPDATE sms_queue SET status='pending', retry_count=0
WHERE status='processing' AND last_attempted_at < NOW()-INTERVAL '15 minutes';

-- firefly_error items (auto-retried by scheduler now — only needed if pre-patch items exist)
SELECT sms_id, sender, body FROM sms_queue WHERE status='firefly_error';
UPDATE sms_queue SET processed=false, processed_at=NULL, retry_count=0, status='pending'
WHERE status='firefly_error';
```

### Scheduler pickup conditions (current)

```sql
WHERE processed = false
  AND (
    status IN ('pending', 'ai_needed')
    OR (status = 'processing' AND retry_count < 3
        AND (last_attempted_at IS NULL OR last_attempted_at < NOW() - INTERVAL '10 minutes'))
    OR (status = 'firefly_error' AND retry_count < 3)
  )
ORDER BY queued_at ASC
LIMIT 1
```
