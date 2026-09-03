---
name: wealth-reader-onboard-widget-integration
description: Stand up a Wealth Reader iframe widget integration end to end - authorise the domain, mount the widget, receive the backend callback, and acknowledge it correctly so the user sees success.
api: Wealth Reader API
base_url: https://api.wealthreader.com/
operations:
  - addDomain
  - POST /entities/
generated: '2026-09-03'
method: generated
source: https://www.wealthreader.com/docs/en/iframe-integration-1-of-2-frontend.md; https://www.wealthreader.com/docs/en/iframe-integration-2-of-2-backend.md; https://www.wealthreader.com/docs/en/flow.md; openapi/wealth-reader-api-for-ai.yaml
---

# Onboard an iframe widget integration

The iframe is how most web clients integrate. It keeps you out of scope for
handling raw bank credentials: the widget renders the per-institution credential
form, drives consent and second factors, and you never see a password.

## The order of events matters

1. Your page loads the widget with an `operation_id` you generate.
2. The user picks a bank, consents, and completes 2FA. The widget handles all of it.
3. Wealth Reader sends the **full JSON to your backend callback first**.
4. **Only if** your callback answered `200` with `{"status":"ok"}` does the
   iframe `postMessage` `"flow completed"` to your frontend.

The callback is the load-bearing path. `postMessage` is a UI signal and carries
no bank data. If your callback is down, the user never sees success — even though
the read itself worked.

## 1. Authorise the domain — before anything else

`POST /domains/` (`addDomain`), form-encoded:

```
method=add
api_key=YOUR_API_KEY
domain=https://example.com
url_callback=https://example.com/wr/callback
tokenize=1
```

For the OAuth path add `access_type=oauth`, and `domain` must be the **same URL**
you use as `redirect_uri` and `url_callback`.

`tokenize=1` means the user authenticates once and you get a reusable `token`
back for later refreshes. `tokenize=0` means you must supply an existing token
on every request.

Until the domain is registered the widget refuses to run. There is **no
`removeDomain` operation** — editing and testing domains happens in the client
area at <https://www.wealthreader.com/clients/>. Plan for that: this step is not
fully reversible through the API.

## 2. Mount the widget

```html
<script>
  const wr_conf = {
    operation_id: crypto.randomUUID(),
    entities_to_display: [],
    wait_full_response: true
  };

  window.addEventListener("message", (event) => {
    if (event.origin !== "https://widget.wealthreader.com") return;
    if (event.data === "flow completed") { /* close selector */ return; }
    if (typeof event.data !== "string") return;
    try {
      const m = JSON.parse(event.data);
      if (m.error) console.log(m.error.code, m.error.message);
    } catch (e) { /* ignore other iframe messages */ }
  });
</script>

<iframe id="wr-iframe" title="Wealth Reader widget" width="100%"
        frameBorder="0" referrerpolicy="origin"></iframe>
<script src="https://widget.wealthreader.com/js/load.js"></script>
```

Always check `event.origin`. The loader finds the iframe by `id="wr-iframe"` and
sizes it from the viewport, so give it vertical room or it will be clipped.

Generate a **fresh `operation_id` per operation**. It is the only bridge between
what happened in the browser and the payload that lands on your server.

Leave `wait_full_response: true`. Turning it off saves seconds and costs you all
transactions. Other useful `wr_conf` fields: `entities_to_display`,
`default_login`, `psd2` / `nonpsd2`, `product_types`, `date_from`, `language`,
`token` (for re-authentication of a dead token).

## 3. Receive and acknowledge the callback

Expose an HTTPS endpoint accepting `POST` with a JSON body. The body is the same
JSON as `POST /entities/`.

Persist first, then answer:

```
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok"}
```

Any other status or any other body and the frontend is never told the flow
finished.

Store `statistics.operation_id`, `statistics.token`, `statistics.code` and
`statistics.SESSION`.

**Treat `operation_id` as idempotent.** A repeated delivery must not create two
operations in your system. The provider states this as your obligation — nothing
on their side enforces it.

**Note what is not there:** this callback carries a complete bank data set and it
is **unsigned**. There is no HMAC and no shared secret; authenticity rests only
on prior domain registration and an unguessable `operation_id`. Compare with the
cards webhook, which is HMAC-signed. Bind the incoming `operation_id` to one you
actually issued, and reject anything you did not originate.

## 4. Test

Use the mock tokens on `POST /entities/`: `MOCKDATA` (clean success), `MOCKOTP`
(second-factor challenge), `MOCKLOGINKO` (login error). There is no separate
sandbox host and no test-mode key — these run against production.

Then confirm your callback returns exactly `{"status":"ok"}`, that a duplicate
delivery creates nothing new, and that a dead token reopens the widget for
re-authentication.
