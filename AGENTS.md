# AGENTS.md

Instructions for AI coding agents working with this repository or implementing custom domains.

## What this is

CustomDomain (customdomain.ai) is managed infrastructure that lets a platform's users connect their own domain in one click: automatic DNS configuration, value-checked verification against public DNS, and automatic SSL/TLS issuance and renewal. 63 DNS and registrar providers are catalogued: 25 have an automatic path (17 by scoped API token, 6 by provider OAuth, 2 by provider-hosted one-click setup) and 38 use a guided manual flow with automatic verification. The live breakdown is public at `GET https://api.customdomain.ai/v1/providers/census`.

There is no separate ownership-challenge step and no TXT verification token. Control of the zone is proven by the rail that wrote the records (an OAuth authorization, a one-click apply, or a scoped API token), or on the manual path by the records themselves appearing in public DNS with the exact expected values.

## Connect a domain in 3 steps (REST)

Base URL: `https://api.customdomain.ai` (API docs: https://docs.customdomain.ai/docs/api-reference). Note that `app.customdomain.ai` is the console host, not the control plane; `/v1/*` paths return 404 there.

```bash
# 1. Create a connection for the user's domain (idempotent per application + domain)
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "app.customer.com"}'

# 2. Start one-click provider authorization (or fall back to guided manual records)
curl -X POST https://api.customdomain.ai/v1/connections/<ID>/oauth:start \
  -H "Authorization: Bearer $API_KEY"

# 3. Poll until live (DNS written, records verified, TLS issued)
curl https://api.customdomain.ai/v1/connections/<ID> \
  -H "Authorization: Bearer $API_KEY"
```

`domain` is the only required field on create. The body rejects unknown fields with a `400`, so do not invent keys: the optional ones are `application_url`, `www_redirect`, `override_spf`, `validate_dmarc`, `validate_caa`, `monitor`, `batch_id`, and `end_user_ref`. Status moves `pending` to `propagating` to `live`, or to `failed` with an `error_code` after 24 hours on an automatic rail (72 on manual). Endpoint shapes here are illustrative; always follow https://docs.customdomain.ai/docs/api-reference for exact schemas.

## MCP server (for agents)

Hosted MCP endpoint: `https://mcp.customdomain.ai/mcp` (streamable HTTP, OAuth client credentials via `https://mcp.customdomain.ai/token`). Registered in the MCP registry as `ai.customdomain/mcp`.

Twelve tools: `search-domain-availability`, `generate-domain-suggestions`, `create-domain-order`, `connect-domain`, `check-connection-status`, `check-order-status`, `reapply-connection`, `disconnect-domain`, `discover-provider`, `forward-domain`, `add-email`, `list-connections`.

**No tool accepts DNS records as input.** Record values are computed server-side by the control plane from vetted templates, which closes off prompt-injection paths that would otherwise end in an arbitrary DNS write. Do not write code that tries to pass records to a tool.

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp
```

Docs: https://docs.customdomain.ai/docs/mcp/overview

## Key references

- Product: https://customdomain.ai
- Documentation: https://docs.customdomain.ai/docs (agent index: https://docs.customdomain.ai/docs/llms.txt)
- Embeddable widget: https://customdomain.ai/connect-domain-widget (npm: `customdomain-js`)
- Widget token minting: https://docs.customdomain.ai/docs/authentication/widget-tokens
- MCP server source: https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp
- Sign up (free tier): https://app.customdomain.ai/signup

## Conventions for edits in this repo

Markdown only. No em dashes or en dashes anywhere. Plain, technically accurate language. American English. Keep files under 300 KB. Every number about provider coverage, plans, or tool counts must trace to a live endpoint (`/v1/providers/census`, `/v1/plans`) or to docs.customdomain.ai; do not carry a figure over from another file without checking it.
