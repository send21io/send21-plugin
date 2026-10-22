# send21 plugin

Crypto pay links and API for people and AI agents.

This plugin connects your agent to the send21 MCP server (`https://send21.io/mcp`, Streamable HTTP) and adds three skills:

| Skill | What it does |
| --- | --- |
| `create-pay-link` | Create a fiat-priced pay link for an invoice or order, paid to the owner's own approved addresses |
| `check-payment` | Check whether a pay link or draft is paid, and bring a payment with a different amount (within 10%) to the owner to accept or decline |
| `verify-webhooks` | Build a receiver that verifies `X-Send21-Signature-V2`, refuses stale deliveries and ignores repeated delivery ids |

## How it works

Agents prepare pay links and payment drafts. A person signs in their own wallet. send21 is non-custodial: it never holds keys or funds, never signs and never broadcasts, so no API key can move money.

Supported: BTC (on-chain and Lightning), USDC, USDT, EURC, SOL and ETH on Bitcoin, Solana, Ethereum, Base, Arbitrum and Polygon. USDT is supported on Solana and Ethereum only. Prices can be set in any of 161 fiat currencies.

## Setup

Auth: API key or OAuth sign-in. The plugin uses an API key (below). Clients that support MCP sign-in (OAuth), such as Claude, Claude Code, Cursor, VS Code and Muse Code, can use `https://send21.io/mcp/oauth` with no headers instead: you sign in to send21 in the browser and approve the app, no key to copy. Connected apps only ever use receiving addresses the owner approved. Details: https://send21.io/docs/mcp ("Sign in instead of a key").

1. Sign in at https://send21.io and create a scoped API key at https://send21.io/api-keys. Pick only the scopes you need (`drafts:read`, `drafts:write`, `addressbook:read`, `addressbook:write`, `webhooks:manage`).
2. Install the plugin and set `SEND21_API_KEY` (Plugins → Configure). The key is sent as `Authorization: Bearer s21_...`.
3. Approve your receiving addresses in the send21 app. New accounts let API keys use approved addresses only.

No account yet? The `create_account` tool signs the owner up with their email and returns a bootstrap key; the owner takes the account over from the emailed link. `tools/list` and `get_fee_schedule` work without a key.

Try it without an account on testnets: https://send21.io/demo

## Docs

- MCP server: https://send21.io/docs/mcp
- Full docs for agents: https://send21.io/llms-full.txt
- Webhooks: https://send21.io/docs/webhooks
- OpenAPI: https://send21.io/openapi.json

Listed in the official MCP Registry as `io.github.send21io/mcp`.

## Logo

`assets/logo.svg` (square, 641.61 x 641.61 viewBox) and `assets/logo.png` (512 x 512).

## Contact

info@send21.io
