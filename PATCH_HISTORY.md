# Patch History

Chronological record of all deployed patches. Each entry: file, date, what changed, why.

---

## 2026-05-29

### `patch_fix_firefly_error.js`
Two-node fix for `firefly_error` items being permanently abandoned:  
1. `Postgres: Get Queue Item` — added `OR (status='firefly_error' AND retry_count<3)` to WHERE clause.  
2. `Postgres: Mark Processed` — changed from always `processed=true` to CASE: `processed=false` / `processed_at=NULL` when `firefly_status='error'`, so failed items stay pickable.  
One-time SQL reset applied for 65 accumulated items from 2026-05-24 Firefly outage.  
**Impact:** Firefly failures now auto-retry up to 3x then fall to `status='error'`. No more silent abandonment.

### `patch_fix_auto_pattern.js`
Adds `Code: Prep Auto Pattern` node before `Postgres: Auto Pattern`.  
Pre-extracts pattern values into `$json.v_hint/v_cat/v_type` so the Postgres node can use simple `{{ $json.v_* }}` expressions instead of `$('Node').item.json.field.replace(...)` which breaks inside PL/pgSQL DO blocks.  
**Impact:** Regex patterns now actually save after every transaction. Previously silently skipped by `continueOnFail:true`.

### `patch_fix_ingestion.js`
Two fixes in one:  
1. `Filter: Drop OTP` rewritten to read SMS body from `$('Parse SMS').item.json` (not `$json.body` which was undefined after `Postgres: Learned Non-Txn` overwrote `$json`).  
2. `Postgres: Merchant Lookup` query rewritten with UNION ALL sentinel to guarantee exactly 1 row returned, preventing silent execution stop when no merchant matches.

### `patch_correction_merchant.js`
Telegram correction flow now writes merchant corrections to `merchant_categories` with `source=correction, hits+3` for higher weighting vs auto-learned entries.

### `patch_merchant_lookup.js`
Initial merchant lookup tier implementation. pg_trgm similarity search with 0.35 threshold. 752 merchants seeded.

---

## 2026-05-28

### `patch_learned_nontxn.js`
Adds `non_txn_patterns` table check before OTP filter. When user clicks "Not a Txn" in Telegram, pattern is stored and checked on future SMS to skip processing immediately.

### `patch_show_body.js`
Dashboard now shows raw SMS body in transaction detail view.

### `patch_store_msgid.js`
`Postgres: Store Msg ID` node saves Telegram message ID to `transactions.tg_message_id` so corrections can edit the original message.

### `patch_review_queue.js`
Adds `needs_review` flag and review queue to dashboard. Transactions with low confidence flagged for manual review.

### `patch_dedup.js`
Deduplication check in SMS Receiver — prevents duplicate sms_id from being enqueued.

### `patch_filter_loan.js`
Adds loan/transfer detection to Parse SMS — loan payment SMSs correctly categorized as transfers, not expenses.

### `patch_categories.js`
Expands category list to match Firefly III account structure. Maps Arabic category keywords to Firefly category names.

---

## 2026-05-27

### `patch_stability.js`
Error handling improvements — `IF: Max Retries` check, Telegram notification for max-retry items, `continueOnFail:true` audit across all Postgres nodes.

### `patch_comprehensive.js`
Major workflow restructure. Added 3-tier architecture (regex → merchant → AI). Replaced flat AI-only approach.

### `patch_markdone_fix.js`
Fix: `Postgres: Mark Processed` was setting `processed=true` but not updating `status='done'`. Both now set together.

### `patch_filter_fix.js`
Fix: OTP filter regex updated — some bank OTP patterns were not matching. Added SNB-AlAhli and AlRajhi OTP formats.

### `patch_full_redesign.js`
Full workflow redesign — introduced queue-based architecture (`sms_queue` table), replaced direct-process approach. Basis of current architecture.

---

## 2026-05-28 (one-off scripts)

### `fix_foreign_txns.js`
One-time bulk fix for 49 GBP/EUR/USD transactions that had wrong categories/amounts due to missing currency detection.

### `fix_firefly_categories.js`
One-time: synced Firefly III category names to match `transactions.category` values in Postgres.

### `fix_merchants.sql`
One-time SQL: seeded `merchant_categories` with 752 merchants from historical transaction data.

### `categories_update.sql`
One-time SQL: updated category mappings for consistency across all tiers.

---

## Earlier

### `patch_foreign_currency.js`
Added currency detection (GBP/EUR/USD) to `Parse SMS`. Extracts currency symbol + converts to SAR equivalent for Firefly posting.

### `patch_receiver.js`
SMS Receiver (`W6vGjhatADpLi4jU`) improvements — better dedup logic, sender normalization.

### `patch_telegram_categories.js`
Telegram correction keyboard updated with full category list matching Firefly III accounts.

### `patch_auto_pattern.js`
Initial auto-pattern learning — after each confirmed transaction, regex hint stored in `regex_patterns`. (Later fixed by `patch_fix_auto_pattern.js`.)

### `patch_main.js`
Early base patch on main workflow — initial Regex tier implementation.
