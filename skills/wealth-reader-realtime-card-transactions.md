---
name: wealth-reader-realtime-card-transactions
description: Enroll employees, register and verify the HMAC-signed Wealth Reader cards webhook, and backfill real-time card transactions with the since_id cursor.
api: Wealth Reader API
base_url: https://api.wealthreader.com/
operations:
  - cardsEnrollmentsCreate
  - cardsEnrollmentsStatus
  - cardsWebhookRegister
  - cardsTransactionsList
generated: '2026-09-03'
method: generated
source: openapi/wealth-reader-api-for-ai.yaml (Cards (real time) tag, cards-webhook-delivery schema); asyncapi/wealth-reader-webhooks.yml
---

# Real-time card transactions

The Cards (real time) surface is a different animal from the rest of this API:
`application/json` instead of form encoding, real HTTP status codes, string
error codes instead of integers, and a genuine push channel. It captures card
spend from the Open Sync mobile app.

## 1. Enroll the employee

`POST /cards/enrollments/` (`cardsEnrollmentsCreate`) with `api_key`, `email`
and optionally `ttl_minutes` (default 20, range 1–60). The employee then confirms
by opening the mobile app and entering that email.

This call **is idempotent** on `(api_key, email)` while the enrollment is still
pending and unexpired — repeating it returns the same request rather than making
a second one. If the email already belongs to you, it returns `status: "active"`
straight away. If it belongs to a different customer, it returns **409**.

**Rate limit: 60 calls per `api_key` per 60 seconds**, counting *every* attempt
including the 200s, 409s and 400s that create nothing — deliberately, so the
endpoint cannot be walked as an enumeration oracle. Exceeding it returns **429**
with `code: rate_limited`. There are no `RateLimit-*` headers, so track your own
call rate; you will not be warned.

There is **no cancel operation** for a pending enrollment. The only unwind is
waiting out `ttl_minutes`. Choose the TTL deliberately.

Poll `GET /cards/enrollments/` (`cardsEnrollmentsStatus`) for
`pending` → `active` → `expired` / `error`.

## 2. Register the webhook

`POST /cards/webhook/` (`cardsWebhookRegister`) with `api_key` and `webhook_url`.

- The URL must be `https://` and must resolve to a publicly routable host.
  Private, loopback, link-local (including the cloud metadata address), CGNAT,
  multicast and reserved addresses are rejected in every notation, and so is a
  host that does not resolve at all. The check runs again **before every
  delivery**, so repointing DNS at an internal address later just gets the
  delivery closed as failed with `blocked_host`.
- `rotate_secret: true` issues a new secret.
- Sending `webhook_url: null` **disables** webhooks — this is your reversal.
- **Omitting** `webhook_url` leaves the stored URL untouched, which is how you
  rotate the secret without changing the URL.

**The `webhook_secret` (64 hex characters) is returned exactly once.** On any
other call it comes back `null` and cannot be retrieved again. Capture it from
the response body before you do anything else, or you permanently lose the
ability to verify signatures and must rotate again.

## 3. Verify every delivery

Deliveries arrive as `POST` JSON with these headers:

```
User-Agent:      Wealthreader-Cards/1.0
Origin:          https://api.wealthreader.com
X-WR-Event:      card_transaction.created | card_enrollment.confirmed
X-WR-Delivery:   <32 hex>
X-WR-Signature:  sha256=<hex of hmac_sha256(raw_body, webhook_secret)>
```

Compute the HMAC over the **raw body bytes**, before any JSON parsing, and
compare **in constant time** (`hash_equals` or equivalent). The provider requires
this explicitly. Reject anything that fails, and do it before you read a single
field.

Body: `{event, delivery_id, sent_at, api_key, data}`. `data` is a
`cards-transaction` for `card_transaction.created` and an active
`cards-enrollment` for `card_enrollment.confirmed`.

**Deduplicate on `delivery_id`.** Retries resend a byte-identical body on the
ladder 1 min → 5 min → 30 min → 2 h → 24 h. Success is any HTTP 2xx; after the
last failed attempt the delivery is dead for good. Ordering is not guaranteed.

If the secret is rotated while retries are pending, those retries are signed with
the **new** secret — so keep the previous secret valid for a short overlap, or
rotate only when nothing is in flight.

## 4. Backfill and reconcile

`GET /cards/transactions/` (`cardsTransactionsList`) returns the same transaction
object the webhook carries. Use it to close gaps whenever your endpoint was down.

Query params: `api_key` (**note it travels in the query string, so it lands in
access logs and proxies** — the spec says so itself), `date_from` (default today
minus 3 days), `date_to` (default today), `email`, `since_id`, `limit` (default
500, max 1000).

Page by passing `payload.next_since_id` back as `since_id` until it comes back
`null`. Results are ordered by ascending id.

Run a periodic backfill regardless of webhook health. A push channel plus a
replayable pull channel over one object is the whole point of this design; using
only the push half throws away the guarantee.
