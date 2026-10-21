---
name: verify-webhooks
description: Build or review a server endpoint that receives send21 webhooks, verifies the X-Send21-Signature-V2 HMAC signature, refuses stale deliveries, ignores repeated deliveries and only then marks orders paid. Use when wiring send21 payment events (draft.confirmed and others) into a shop, backend or agent.
---

# Verify send21 webhooks

## When to use

- Writing a receiver for `draft.confirmed` to mark an order paid.
- Reviewing existing webhook code for send21.

## Register an endpoint

`POST /api/v1/webhooks` (scope `webhooks:manage`) with your URL and the events you want. The response contains the signing secret once: store it as a secret, never in code. Send a test event with `POST /api/v1/webhooks/{id}/test`.

Events: `draft.created`, `draft.seen`, `draft.confirmed`, `draft.amount_mismatch`, `draft.mismatch_accepted`, `draft.mismatch_rejected`, `draft.mismatch_reminder`, `draft.mismatch_unresolved`, `draft.expired`, `draft.cancelled`, `payment_request.expired`, `test`.

## Delivery facts

- `POST`, `Content-Type: application/json`, body `{ "event": "<type>", "data": { ... } }`.
- Any 2xx counts as delivered. Anything else, or no answer within 15 seconds, is retried after 1, 2, 4, 8 minutes and so on, up to 8 attempts.
- Deliveries can arrive out of order.
- Headers: `X-Send21-Event`, `X-Send21-Delivery` (unique id; retries reuse it), `X-Send21-Timestamp`, `X-Send21-Signature` (legacy) and `X-Send21-Signature-V2` (recommended).

## Verify X-Send21-Signature-V2

Format: `t=<unix seconds>,v1=<hex HMAC-SHA256 of "<t>.<delivery id>.<raw body>">`, keyed with your webhook secret.

1. Read the raw request bytes before parsing JSON.
2. Parse `t` and `v1` from the header and read `X-Send21-Delivery`.
3. Refuse deliveries more than 300 seconds (5 minutes) old.
4. Compute HMAC-SHA256 over `<t>.<delivery id>.<raw body>` and compare with `v1` in constant time.
5. Answer 401 when the header is missing or wrong.

Node.js:

```js
const sig = String(req.headers['x-send21-signature-v2'] ?? '');
const t = /(?:^|,)t=(\d+)/.exec(sig)?.[1];
const v1 = /(?:^|,)v1=([0-9a-f]{64})/.exec(sig)?.[1];
const deliveryId = String(req.headers['x-send21-delivery'] ?? '');
const fresh = t !== undefined && Math.abs(Date.now() / 1000 - Number(t)) <= 300;
const expected = crypto.createHmac('sha256', secret)
  .update(t + '.' + deliveryId + '.').update(rawBody).digest('hex');
const ok = fresh && v1 !== undefined &&
  crypto.timingSafeEqual(Buffer.from(v1, 'hex'), Buffer.from(expected, 'hex'));
if (!ok) return res.status(401).end();
```

Python:

```python
import hmac, hashlib, re, time
sig = request.headers.get("X-Send21-Signature-V2", "")
t = re.search(r"(?:^|,)t=(\d+)", sig)
v1 = re.search(r"(?:^|,)v1=([0-9a-f]{64})", sig)
delivery_id = request.headers.get("X-Send21-Delivery", "")
valid = False
if t and v1 and abs(time.time() - int(t.group(1))) <= 300:
    signed = t.group(1).encode() + b"." + delivery_id.encode() + b"." + raw_body
    expected = hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()
    valid = hmac.compare_digest(expected, v1.group(1))
```

The legacy `X-Send21-Signature` (`sha256=` plus the HMAC of the raw body only) still works, but it has no timestamp, so a copied delivery passes it again. Prefer V2.

## Handle repeats yourself

Retries reuse the same `X-Send21-Delivery` id. Your receiver must store the delivery ids it has handled and ignore any id it has already processed, so an order is never fulfilled twice. Because deliveries can arrive out of order, decide from the event and the current status, not from arrival order.

## Before you ship goods

- Act on `draft.confirmed`, after the signature check passes.
- Check that the amount, currency and `orderId` are the ones you expect. Reconcile on the exact amount and transaction id.
- `draft.amount_mismatch` is not a payment confirmation. It pays the draft only after the owner accepts it (see the `check-payment` skill).
- Never mark an order paid from the browser.
