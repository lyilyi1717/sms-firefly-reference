# Known Issues & Quirks

## Active Quirks (not bugs — by design)

### Qdrant search bypassed
`nomic-embed-text` produces near-identical embeddings for all bank SMS — useless for similarity search.  
**Do not** rebuild the Qdrant search path until a better embedding model is available.  
Upsert still runs (corpus is being populated), but `Qdrant: Search` → `Code: Apply Qdrant` → `IF: Qdrant Hit` are orphaned nodes that are wired but never reached.

### `received_at` is TEXT unix milliseconds
Not a TIMESTAMPTZ — stored as TEXT. Cast everywhere: `to_timestamp(received_at::numeric / 1000)`.

### `ai_needed` status = legacy alias
Treated identically to `pending` by the scheduler. Leftover from old architecture. No fix needed.

### `continueOnFail: true` swallows errors
All Postgres nodes have `continueOnFail: true`. Errors are silently ignored. After any deployment:
1. Check a live execution with `?includeData=true`
2. Inspect each Postgres node's output for error objects

### Places tier label (261 transactions)
`tier='places'` = legacy Google Places API lookup. Nodes removed. Those 261 rows keep the label as historical record — not a bug, don't fix.

### Max retries logic
`Postgres: Mark Processing` sets `status='error'` when `retry_count >= 2` BEFORE the increment.  
Post-increment value = 3. Max 3 attempts total, not 2.

### Qdrant upsert ID
Uses `parseInt(sms_id)` to extract numeric prefix from sms_id like `"1776633743571_SNB-AlAhli"`.  
Old transactions have string IDs in Qdrant — only 1 vector was stored successfully before this fix.

---

## Fixed Bugs

### [FIXED] Filter: Drop OTP let all OTPs through
**Patch:** `patch_fix_ingestion.js`  
**Root cause:** `Postgres: Learned Non-Txn` overwrites `$json` with its query result `{matched, skip_reason}`. Downstream nodes using `$json.body` got `undefined`. `Filter: Drop OTP` was checking `String(undefined)` = `'undefined'` — never matched OTP patterns — so all OTPs/promos passed through as transactions.  
**Fix:** `Filter: Drop OTP` now reads body via `$('Parse SMS').item.json` and uses it as output base to preserve all SMS fields for downstream nodes.

### [FIXED] Merchant Lookup 0-row result silently killed execution
**Patch:** `patch_fix_ingestion.js`  
**Root cause:** When `Postgres: Merchant Lookup` returned 0 rows (no match > 0.35 threshold), n8n silently stopped execution — `Code: Apply Merchant Lookup`, `IF: Merchant Hit`, and `Ollama: Chat` never ran. Items stalled in `processing`, hit max 3 retries, became `error`.  
**Fix:** UNION ALL sentinel row in the pg_trgm query ensures always exactly 1 row returned. See DATABASE.md for full query.

### [FIXED] Auto Pattern regex patterns never saved
**Patch:** `patch_fix_auto_pattern.js`  
**Root cause:** `Postgres: Auto Pattern` threw "invalid syntax" because n8n's expression renderer can't handle `$('Node Name')` references chained with `.replace(/'/g,"''")` inside a PL/pgSQL `DO $$ ... $$` block. With `continueOnFail:true`, it silently skipped — regex patterns were never being saved after any transaction.  
**Fix:** New node `Code: Prep Auto Pattern` pre-extracts values into `$json.v_hint`, `$json.v_cat`, `$json.v_type`. The `Postgres: Auto Pattern` query uses simple `{{ $json.v_* }}` references. `Code: Prep Auto Pattern` is inserted between `Postgres: Log` and `Postgres: Auto Pattern`.

### [FIXED] Foreign currency transactions (49 items)
**Script:** `fix_foreign_txns.js`  
49 transactions in GBP/EUR/USD were miscategorized or had wrong amounts due to currency detection gap. One-time bulk fix applied.

---

## Debugging Checklist

When items stall in `processing`:
1. Check `retry_count` — if ≥ 3, status should be `error`
2. Check `last_attempted_at` — if >10min ago and status=`processing`, scheduler should re-pick it
3. Check last execution with `?includeData=true` — look for which node was last to run
4. Common causes:
   - Merchant Lookup 0-row (pre-sentinel) → FIXED
   - Ollama timeout/crash → check `http://192.168.8.10:11434`
   - Firefly III down → check `http://192.168.8.10:8181`
   - Postgres connection issue → check n8n credential `qx7cEqJpZFgDgCax`
