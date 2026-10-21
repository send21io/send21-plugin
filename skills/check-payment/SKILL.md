---
name: check-payment
description: Check whether a send21 pay link or payment draft has been paid, and handle a payment that arrived with a different amount (within 10% of the billed amount) by showing it to the owner to accept or decline. Use when the user asks "did they pay?", "has invoice X arrived?", or when a draft.amount_mismatch event arrives.
---

# Check a payment and handle a different amount

## When to use

- "Has the client paid invoice 1042?"
- "What happened with this pay link?"
- A `draft.amount_mismatch` webhook arrived, or a status shows `amountMismatch`.

## Check where it stands

- Pay link: call `get_payment_request` with its id. It returns the payment status.
- Draft: call `get_draft_status` (cheap snapshot) or `get_draft` (full detail).
- The MCP prompt `check_payment` (argument `id`) walks through the same steps and explains the result in plain words.

Draft statuses: `AwaitingPayment`, `Seen` (seen on-chain, not yet confirmed), `Confirmed`, `Expired`, `Cancelled`. Treat only `Confirmed` as paid. Before a payment is marked confirmed, send21 checks the transaction at a second source; a replaced or reorged-out payment sends the draft back to `AwaitingPayment`.

Poll gently: the limit is 300 requests per minute per key, then 429. Prefer webhooks (see the `verify-webhooks` skill) over tight polling.

## A payment with a different amount

A transfer within 10% of the billed amount (over or under) is never confirmed on its own. Transfers more than 10% off are not matched to the draft at all.

When it happens:

- The draft keeps its status (`AwaitingPayment`), and `amountMismatch.reviewState` is `pending`. `draft.amount_mismatch` is sent once.
- A pay-link payment of the exact amount that arrives more than 10 minutes after its rate lock ended is treated the same way, with `reviewReason: "rate_lock_expired"`.
- The money is already in the receiver's wallet. send21 never holds funds.

What to do:

1. Show the owner what was billed and what arrived, using `get_draft` or the MCP prompt `handle_amount_mismatch` (argument `draftId`). It shows both amounts and how to accept or decline; it never decides.
2. Ask the owner to decide. Do not decide for them.
3. The owner's decision is recorded with the REST API (or in the send21 app):
   - Accept: `POST /api/v1/drafts/{id}/accept-received` with an `acceptReason`. The draft continues as paid and `draft.mismatch_accepted` is sent.
   - Decline: `POST /api/v1/drafts/{id}/reject-received` with a `rejectReason` (required). `draft.mismatch_rejected` is sent.
4. A declined payment is refunded by the receiver, outside send21. send21 cannot send money back.

If nobody decides: 2 hours before the draft expires, one `draft.mismatch_reminder` is sent (and an email). At expiry the draft becomes `Expired` with `reviewState: unresolved`, and `draft.mismatch_unresolved` is sent. It can still be accepted afterwards.

If the owner set a `shortfallPolicy`, short payments within that percent are accepted automatically (`acceptReason: "auto: shortfallPolicy"`). Overpayments always wait for a person.

## Never

- Never mark an order paid from a status you did not verify, and never from the payer's browser.
- Never accept or decline a different amount without the owner's explicit decision.
