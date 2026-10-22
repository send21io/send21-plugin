---
name: create-pay-link
description: Create a send21 crypto pay link for an invoice or order, priced in a fiat currency, paid straight to the owner's own wallet. Use when the user asks to bill someone, send an invoice, request a payment, or take a crypto payment for an order.
---

# Create a pay link for an invoice

send21 prepares the pay link. The payer signs and sends from their own wallet. send21 never holds keys or funds, and this skill never moves money.

## When to use

- "Invoice my client 49 EUR, they can pay in USDC or bitcoin"
- "Make a pay link for order 1042"
- "Request 1,500 SEK in EURC"

## Before you start

1. Call `get_account_status`. It says what still stands between you and a live pay link (for example an unconfirmed owner or no approved receiving address) and the next steps.
2. Check `receivingAddressPolicy` in that status. With `ApprovedOnly` (the default for new accounts), call `list_address_book` and use only receiving addresses the owner has approved; any other address returns `422 address_not_approved` and nothing is created. With `InlineAllowed` (older accounts), the owner's own receiving address can be passed directly in `options`, but only an address the owner gave you.
3. If the owner has no approved address for a currency they want, call `propose_receiving_address`. Tell the user that the owner must sign in to send21 and approve it with a passkey or authenticator code, comparing the address character by character. No API key can approve an address. Do not continue with that address until it shows as approved.
4. If you have no key at all, `create_account` with the owner's email returns a bootstrap key. Until the owner takes the account over from the emailed link (valid 72 hours), creating anything returns `403 owner_not_confirmed`. An email that already has an account gets `409 account_exists`: ask the owner for a key instead.

## Create the pay link

Call `create_payment_request` (scope `drafts:write`) with:

- `fiatCurrency` and `fiatAmount`: any of 161 fiat currencies. The rate locks when the payer picks a currency.
- `options`: one entry per accepted currency and network, each with the owner's approved address. Read `send21://docs/currencies` for the exact currency and network values. Supported: BTC (on-chain or Lightning to a Lightning address), USDC, USDT, EURC, SOL and ETH on Bitcoin, Solana, Ethereum, Base, Arbitrum and Polygon. USDT is only on Solana and Ethereum.
- `orderId` and `memo`: your invoice or order reference. It is echoed in every webhook.
- `reuseOpenForOrder: true` when the user may ask twice for the same order: the open link for the same `orderId` is returned instead of a second one.
- Optional `declineNote` (up to 200 characters): what happens if the payer does not pay. The pay page shows it as "If you don't pay:".
- Optional `shortfallAbsorbUpToPercent` (0-10): accept payments that arrive slightly short, for example when an exchange takes its fee from the amount. Overpayments always wait for a person.
- `idempotencyKey`: a key of your own per invoice, so a retry returns the same link instead of a second one. Over the REST API (`POST /api/v1/payment-requests`), send it as the `Idempotency-Key` header.
- Optional `expiryHours`: how long the link stays open (default 24).

The response contains `payPath`. The link to share is `https://send21.io` followed by that path. The payer scans one standard QR code (BIP-21, Solana Pay, EIP-681 or a Lightning invoice) from any wallet.

## After creating

- Show the user the link, the amount and currency, the accepted currencies, the expiry (`expiresAt`), and any `changeSignals` (`new_address`, `new_asset`, `unusual_amount`). The pay page shows the same lines to the payer.
- Do not report the invoice as paid until send21 reports it confirmed. Use the `check-payment` skill or a signed `draft.confirmed` webhook.

## Never

- Never ask for or accept private keys, seed phrases or signing material. send21 never asks for them; anything that does is an attack.
- Never put an address in `options` that the owner has not approved or given you as their own.
