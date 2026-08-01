---
name: Process incoming transactions
description: >-
  Poll Procuros for unprocessed incoming documents from trade partners, ingest them into your ERP,
  and acknowledge each as processed (individually or in bulk) using the Procuros API v2.
api: openapi/procuros-openapi-original.yml
operations: [v2_list_received_transactions, v2_mark_transaction_processed, v2_bulk_mark_transactions_processed, v2_report_error]
---

# Process incoming transactions (Procuros API v2)

Use this to pull inbound EDI documents and confirm processing. The API is poll-based (no webhooks).

## Auth & environment
- Auth: `Authorization: Bearer <api-token>`, HTTPS only.
- Base URL: `https://api.procuros.io/v2` (or `https://api.procuros-staging.io/v2` for testing).

## Steps
1. **List unprocessed inbound documents** — `GET /v2/transactions` (`v2_list_received_transactions`). Only the latest version of each document is returned (older pending versions are auto-removed). Use cursor pagination: pass `per_page` (1-100, default 25) and follow `nextCursor` / `nextPageUrl` while `hasMore` is `true`.
2. **Ingest each transaction** into your ERP, reading `procurosTransactionId`, `type`, and `content`. Use `isLatestVersion` / `replacesProcurosTransactionId` if you track version chains.
3. **Acknowledge processing**:
   - One at a time — `PUT /v2/transactions/{procurosTransactionId}` (`v2_mark_transaction_processed`).
   - In bulk — `POST /v2/transactions/bulk/mark-processed` (`v2_bulk_mark_transactions_processed`) with a list of ids. Prefer bulk to avoid `429`.
4. **Report data problems** — when a document can't be processed (missing/duplicate GLN or GTIN, unworkable data), `POST /v2/errors` (`v2_report_error`) with `errorType` `DATA` and a clear `errorReason` so it enters exception handling instead of silently failing.

## Conventions
- Cursor pagination: `items`, `hasMore`, `nextCursor`, `nextPageUrl`.
- On `429`, back off and switch to the bulk endpoint.
- Errors are `{ message, errors? }` (422 carries per-field `errors`).
