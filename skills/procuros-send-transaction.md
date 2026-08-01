---
name: Send a transaction to a trade partner
description: >-
  Create and send a business document (order, invoice, shipping notice, etc.) to a Procuros trade
  partner and confirm it was accepted, using the Procuros API v2.
api: openapi/procuros-openapi-original.yml
operations: [v2_ping, v2_send_transaction, v2_show_transaction, v2_report_error]
---

# Send a transaction to a trade partner (Procuros API v2)

Use this to push an outgoing EDI document to a partner.

## Auth & environment
- Auth: `Authorization: Bearer <api-token>` (static token per ERP Connection, from Procuros support). HTTPS only.
- Staging base URL: `https://api.procuros-staging.io/v2`; Production: `https://api.procuros.io/v2`. Tokens are per-environment. Build and test on staging first.

## Steps
1. **Verify connectivity** — `GET /v2/ping` (`v2_ping`). Expect `204`. A `401` means the token is wrong for this environment.
2. **Send the document** — `POST /v2/transactions` (`v2_send_transaction`) with a `SentTransaction` body: set `type` (e.g. `ORDER`, `INVOICE`, `SHIPPING_NOTICE`) and the typed `content` (header + line items + parties: buyer/seller/shipTo/technicalSender).
3. **Handle the response**:
   - `2xx` — accepted.
   - `400 "Payload already received"` — the API deduped a duplicate send; treat as already-sent, do not blindly retry.
   - `422` — read the `errors` map (field → messages), fix the offending fields, resend.
   - `500/502/503/504` — transient; mark for later retry.
4. **Confirm** (optional) — `GET /v2/all-transactions/{procurosTransactionId}` (`v2_show_transaction`) to read the resulting transaction `status`.
5. **Report a build error** — if you cannot construct a valid payload (e.g. missing GTIN/GLN in your ERP), `POST /v2/errors` (`v2_report_error`) with `errorType` (`DATA` | `INTERNAL`), `errorReason`, `transactionIdentifier`, `transactionType` so it enters Procuros exception handling.

## Conventions
- Datetimes are ISO8601, Europe/Berlin timezone.
- The API is not RFC 9457; errors are `{ message, errors? }`.
- No client idempotency key — rely on server-side duplicate detection (400).
