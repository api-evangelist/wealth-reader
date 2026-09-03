---
name: wealth-reader-read-financial-assets
description: Read a consenting end user's accounts, cards, loans, deposits and investment portfolios from one financial institution through the Wealth Reader API, and handle the second-factor and partial-coverage cases correctly.
api: Wealth Reader API
base_url: https://api.wealthreader.com/
operations:
  - POST /entities/
  - getEntities
  - GET /entities/category-types/
generated: '2026-09-03'
method: generated
source: openapi/wealth-reader-api-for-ai.yaml; https://www.wealthreader.com/docs/en/introduction.md; https://api.wealthreader.com/error-codes/?lang=en
---

# Read financial assets from an institution

Wealth Reader is **read-only**. It never initiates payments. Every call bills as
"products read", so treat each request as costly and never blind-retry.

## Before you call

You need an `api_key` (signup plus a technical onboarding session — there is no
self-serve key) and a `token`, the custodied handle to the end user's bank
credential. The token comes from the widget or OAuth flow; you do not handle the
user's bank password yourself.

## Pick the institution

`GET /entities/` (operationId `getEntities`) returns the catalogue. In
production always send `show_only_tested=1`.

Each entry carries `code` (what you send back), `psd2` (`"1"` regulatory
channel, `"0"` the richer non-PSD2 channel), `bic`, `country_code` and `inputs`.
**Do not assume every institution asks for a username and a password** — `inputs`
describes the exact credential form, and `second_password`, `third_password` and
`document_type` appear on a real minority of banks. If you use the widget you
never touch this.

## Read the data

`POST /entities/` with `Content-Type: application/x-www-form-urlencoded`.

| Field | Notes |
| --- | --- |
| `api_key` | your key |
| `code` | institution code from the catalogue |
| `token` | the custodied credential |
| `product_types` | comma-separated; omit for everything licensed on the key |
| `date_from` | `YYYY-MM-DD`, must be before today |
| `only_balances` | `true` skips product detail |
| `fetch_transaction_details` | see the warning below |

**This call is synchronous and can take minutes.** The bank may raise a second
factor mid-flight. Set a generous client timeout. Do not enable
`fetch_transaction_details` casually: it performs one or more extra bank
navigations *per transaction*, so execution time grows with transaction volume,
and it requires a dedicated environment.

## Read the response correctly

The envelope is `{success, payload, error, statistics}`. **Branch on `success`
and `error.code`, not on the HTTP status** — a failed bank read routinely comes
back as HTTP 200 with `success: false`.

Always store `statistics.SESSION`. Support cannot reproduce a case without it.

`payload` holds sibling arrays per product type — `accounts`, `cards`, `loans`,
`deposits`, `portfolios`, and more. Every product carries a stable 40-hex `uuid`,
so sync incrementally on `uuid` rather than diffing on amount and date.
Portfolio positions carry `ISIN` (ISO 6166), which is your join key to any
market-data source.

Do not assume every key is present. What comes back depends on `product_types`
and on what the user actually holds.

## Handle warnings — they are not failures

`statistics.warnings[]` arrives on a **successful** read. `success` is still
true and the payload is still usable. Each warning names the `product_type` it
applies to, so a partial failure is attributable. Log them; do not fail the
operation. Codes 1050–1061 are the partial-coverage family; fetch the live list
from `GET /warning-codes/?lang=en`.

## Handle errors — the `fatal` flag is the whole decision

Fetch `GET /error-codes/?lang=en` once and cache it. Each of the 40 codes carries
a `fatal` boolean and that is the only thing that decides whether a retry makes
sense.

- **`fatal: true`** — the call will fail again with the same input. Surface it
  and stop. For credential errors (2000, 2001) retrying walks the end user
  towards a **lockout at their own bank**. This is the single most damaging
  mistake an agent can make against this API.
- **`fatal: false`** — transient. Back off, or let the next scheduled run take it.

Cases worth special handling:

- **2005** — a session for this user is already active. Wait; the provider
  suggests retrying every 5 minutes. Never run two reads for one user in parallel.
- **2017 / 20171** — the bank offers several second-factor methods. Re-call with
  the same session identifier and set `otp_method` to the exact string from one
  object in `statistics.otpMethods` — the string, not the array index, and never
  a hardcoded label.
- **2020** — multi-contract user. Re-call with `contract_name` set from the
  `contract_names` object returned alongside the error.
- **3 / 9999** — the token is dead. Reopen the widget with that token so the user
  can re-authenticate. There is no un-revoke.
- **3000 / 3002** — the institution is under maintenance. There is no status page
  to check first; back off for minutes (3000) or 24 business hours (3002).
