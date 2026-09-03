---
name: wealth-reader-manage-tokens
description: List, revoke and reassign the custodied bank-credential tokens held under a Wealth Reader api_key, and understand which of those actions can be taken back.
api: Wealth Reader API
base_url: https://api.wealthreader.com/
operations:
  - queryTokensByApiKey
  - revokeToken
  - reasignToken
generated: '2026-09-03'
method: generated
source: openapi/wealth-reader-api-for-ai.yaml; conventions/wealth-reader-conventions.yml; https://api.wealthreader.com/error-codes/?lang=en
---

# Manage custodied tokens

A `token` is Wealth Reader's opaque handle to one end user's credential at one
institution. It is what lets you refresh their data later without asking for a
password again — and it is the thing you must be most careful with.

All three operations are `POST` with
`Content-Type: application/x-www-form-urlencoded`.

## List what you hold

`POST /tokens/` (`queryTokensByApiKey`) with `api_key` and an optional `page`.
Results come in blocks of **500 per page**, default page 1. No total and no
next-page field is returned — page until a short block comes back.

Each row gives `token`, `entity_code`, `created_at`, `accesed_at`,
`times_accesed`, `latest_code` and `latest_session`.

`latest_code` is the useful one. Cross-reference it against
`GET /error-codes/?lang=en`: a row sitting on a `fatal: true` code (3, 2000,
2001, 9999) is dead weight — every scheduled refresh against it will fail and,
for credential errors, pushes the user towards a bank lockout. Sweep those into
a re-authentication queue rather than retrying them.

## Revoke — and understand that this is one-way

`POST /tokens/revoke/` (`revokeToken`) with `api_key` and `token`.

**There is no un-revoke.** The token becomes error code 9999, "Token
invalidated by client request", and the provider's own remediation is that the
client must generate a new one — which requires the end user to authenticate at
their bank again, in person, through the widget. Revocation is the reversal *of*
tokenisation; nothing reverses it.

Never revoke as a cleanup step, a retry strategy, or a response to a transient
error. Revoke when the user asks to disconnect, or when you are deliberately
ending the relationship. Confirm with a human first.

## Reassign

`POST /tokens/reasign/` (`reasignToken`) with `api_key_source`, `api_key_target`
and `token`. Moves a token between two of your keys.

The inverse call — the same request with the two keys swapped — is symmetric and
will move it back, but the provider does not document that as a supported
reversal and states no window for it. Treat it as reversible in practice and
unguaranteed on paper.

## Reading the responses

Revoke and reassign return `{success, message}` on 200. These endpoints do use
real HTTP statuses (400 bad parameters, 401 unauthorised or already revoked, 500
server error) — unlike `POST /entities/`, which signals failure inside a 200.

There is no `Idempotency-Key` header anywhere in this API. A repeated revoke of
an already-revoked token returns 401, so it is safe to re-send, but a repeated
reassign is *not* a no-op — it will move the token again if the keys still line
up. Track what you have sent.
