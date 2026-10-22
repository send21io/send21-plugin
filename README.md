# send21 plugin

Crypto pay links and API for people and AI agents.

This plugin connects your agent to the send21 MCP server with sign-in (`https://send21.io/mcp/oauth`, Streamable HTTP, OAuth) and adds three skills:

| Skill | What it does |
| --- | --- |
| `create-pay-link` | Create a fiat-priced pay link for an invoice or order, paid to the owner's own approved addresses |
| `check-payment` | Check whether a pay link or draft is paid, and bring a payment with a different amount (within 10%) to the owner to accept or decline |
| `verify-webhooks` | Build a receiver that verifies `X-Send21-Signature-V2`, refuses stale deliveries and ignores repeated delivery ids |

## How it works

Agents prepare pay links and payment drafts. A person signs in their own wallet. send21 is non-custodial: it never holds keys or funds, never signs and never broadcasts, so no API key can move money.

Supported: BTC (on-chain and Lightning), USDC, USDT, EURC, SOL and ETH on Bitcoin, Solana, Ethereum, Base, Arbitrum and Polygon. USDT is supported on Solana and Ethereum only. Prices can be set in any of 161 fiat currencies.

## Setup

Install = sign in. No key to copy.

1. Install the plugin. Its `mcp.json` points at `https://send21.io/mcp/oauth` with no headers.
2. When the client connects, it opens send21 in your browser. Sign in, check what the app asks for, and approve.
3. Approve your receiving addresses in the send21 app (passkey or authenticator code). A connected app only ever uses receiving addresses you approved.

The connection is listed under [API keys](https://send21.io/api-keys) as a connected app, where Disconnect ends it at once. A connection nobody uses for 90 days ends by itself.

### Sign in from other clients

Claude Code:

```bash
claude mcp add --transport http send21 https://send21.io/mcp/oauth
# then run /mcp in Claude Code and choose send21 to sign in
```

VS Code (`.vscode/mcp.json`):

```json
{
  "servers": {
    "send21": {
      "type": "http",
      "url": "https://send21.io/mcp/oauth"
    }
  }
}
```

Any MCP client that supports OAuth: add `https://send21.io/mcp/oauth` as a remote (Streamable HTTP) MCP server with no headers.

### Prefer an API key?

API keys still work, on `https://send21.io/mcp`. Create a scoped key at https://send21.io/api-keys with only the scopes you need (`drafts:read`, `drafts:write`, `addressbook:read`, `addressbook:write`, `webhooks:manage`); it is sent as `Authorization: Bearer s21_...`. Turn off the plugin's `send21` server and add this to `.cursor/mcp.json` or `~/.cursor/mcp.json`, with `SEND21_API_KEY` set in your environment:

```json
{
  "mcpServers": {
    "send21": {
      "url": "https://send21.io/mcp",
      "headers": {
        "Authorization": "Bearer ${env:SEND21_API_KEY}"
      }
    }
  }
}
```

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
