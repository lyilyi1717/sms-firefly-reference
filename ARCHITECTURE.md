# Architecture — SMS → Firefly III

## 3-Tier Categorization Pipeline

```
MacroDroid (Android)
    │ HTTP POST → n8n Webhook (SMS Receiver)
    ▼
sms_queue table  (status=pending)
    │ Scheduler: every 30s, picks 1 item
    ▼
┌─────────────────────────────────────────────────────┐
│ TIER 1: REGEX                                       │
│ regex_patterns table · confidence=0.95 · ~44% hit  │
├─────────────────────────────────────────────────────┤
│ TIER 2: MERCHANT LOOKUP                             │
│ merchant_categories · pg_trgm score>0.35            │
│ confidence = 0.4 + (score × 0.55) · ~35% expected  │
├─────────────────────────────────────────────────────┤
│ TIER 3: OLLAMA AI                                   │
│ gemma3:1b · ~42s/request · remaining ~20%           │
└─────────────────────────────────────────────────────┘
    │
    ▼
HTTP POST → Firefly III
    │
    ├── Postgres: Log transaction
    ├── Auto-learn: regex_patterns + merchant_categories + Qdrant
    └── Telegram: confirm message with inline correction keyboard
```

---

## n8n Main Workflow — Node Flow (exact names)

```
Schedule Trigger
    └─► Postgres: Get Queue Item
            └─► IF: Has Item
                  ├─[false]─► (stop — no item)
                  └─[true]──► Postgres: Mark Processing
                                └─► IF: Max Retries  (status === 'error')
                                      ├─[true]──► HTTP: Notify Max Retries
                                      └─[false]─► Config
                                                    └─► Parse SMS
                                                          └─► Postgres: Learned Non-Txn
                                                                └─► Filter: Drop OTP
                                                                      └─► IF: Non-Txn?
                                                                            ├─[non-txn]─► HTTP: Notify Non-Txn
                                                                            │               └─► Postgres: Mark Non-Txn
                                                                            └─[txn]─────► Regex Extract
                                                                                            └─► Postgres: Ensure Tables
                                                                                                  └─► Postgres: Load Patterns
                                                                                                        └─► Code: Test Patterns
                                                                                                              └─► IF: Regex Hit
                                                                                                                    ├─[true]──► Code: Resolve Account
                                                                                                                    └─[false]─► Code: Prep Merchant
                                                                                                                                  └─► Postgres: Merchant Lookup
                                                                                                                                        └─► Code: Apply Merchant Lookup
                                                                                                                                              └─► IF: Merchant Hit
                                                                                                                                                    ├─[true]──► Code: Resolve Account
                                                                                                                                                    └─[false]─► Ollama: Chat
                                                                                                                                                                  └─► Code: Parse AI
                                                                                                                                                                        └─► Code: Resolve Account
```

```
Code: Resolve Account
    └─► IF: Skip Firefly?
          ├─[true]──► Postgres: Mark Non-Txn
          └─[false]─► HTTP: POST Firefly
                        └─► Postgres: Log
                              └─► Code: Prep Auto Pattern
                                    └─► Postgres: Auto Pattern
                                          └─► Postgres: Update Merchant Memory
                                                └─► Ollama: Embed Final
                                                      └─► Qdrant: Upsert
                                                            └─► Postgres: Review Count
                                                                  └─► Code: Build Confirm Msg
                                                                        └─► HTTP: Telegram Confirm
                                                                              └─► Postgres: Store Msg ID
                                                                                    └─► Postgres: Mark Processed
```

**Orphaned nodes** (wired but unreachable — legacy search path):
`Ollama: Embed`, `Qdrant: Ensure Collection`, `Qdrant: Search`, `Code: Apply Qdrant`, `IF: Qdrant Hit`

---

## Auto-Learning Loop

After every confirmed transaction:

1. **regex_patterns** — `Postgres: Auto Pattern` inserts/upserts a pattern for this sender+hint
2. **merchant_categories** — `Postgres: Update Merchant Memory` increments hits or inserts (source=auto, hits+1; correction, hits+3)
3. **Qdrant corpus** — `Ollama: Embed Final` → `Qdrant: Upsert` stores embedding (nomic-embed-text; search bypassed due to poor quality)

---

## Telegram Correction Flow

Each confirmed transaction sends an inline keyboard with options:
- Category corrections → callback to `smscorrection01` workflow
- "Not a Txn" → marks non_txn, learns pattern
- Corrections update merchant_categories with source=correction (hits+3 weight)

---

## Scheduler Behaviour

- Runs every 30s
- Picks **1 item** per run: `processed=false AND status IN ('pending','ai_needed')` OR `status='processing' AND retry_count<3 AND last_attempted_at < NOW()-10min`
- `Mark Processing` increments retry_count; sets status=`error` if `retry_count >= 2` **before** increment (max 3 total attempts)
- Stalled items (status=processing, >10min old) are re-picked automatically
