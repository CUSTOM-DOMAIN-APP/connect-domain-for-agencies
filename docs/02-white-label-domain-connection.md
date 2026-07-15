# White Label Domain Connection: Your Brand on Every Screen

Agencies and resellers spend years convincing clients that the platform is theirs. Then the domain setup step redirects the client to a screen with somebody else's logo on it, and the story develops a crack: who else is actually behind the thing I pay for? White label domain connection closes that gap. The client pastes their domain, approves the change from their own registrar account, and watches records, verification, and certificate complete in about 30 seconds, under your brand or under no brand at all.

This guide covers what the client experiences, then the two ways to build it: the embeddable widget and the headless API. It assumes the basics from the [main guide](../README.md).

## What "white label" means here, concretely

Not a logo swap. Every screen the client touches during connection is brandable: roughly 90 tokens covering colors, names, and copy, localized in 13 languages. The vendor's name appears nowhere the client looks, by design. The one screen you cannot brand, deliberately, is the client's own DNS provider sign-in, because that page belongs to their provider and its familiarity is exactly what makes the authorization trustworthy (more on that trust model in [doc 03](03-stop-collecting-registrar-logins.md)).

## What the client experiences

1. Inside your portal (or from a link you send), the client is asked to connect a domain.
2. They type `shop.northwind.com`. Their DNS provider is detected as they type.
3. They click Connect and sign in at their own provider to approve. No DNS panel, no record types, no copy-paste.
4. Records are written, ownership is verified, and the TLS certificate issues. Typically about 30 seconds end to end.
5. They land back in your portal with the domain live. Your brand carried every step.

If their provider cannot be automated, the flow degrades gracefully instead of dead-ending: the client gets exact records to paste and a live check that confirms the moment they resolve. Still your brand, still no ticket.

## Path one: the embeddable widget

The [connect domain widget](https://customdomain.ai/connect-domain-widget) is the whole flow above as a drop-in component: provider detection, the three connection methods, verification, certificate status, error states, and retries. You embed it, theme it, and listen for the result.

Configuration is declarative. Conceptually:

```json
{
  "branding": {
    "name": "Northwind Studio",
    "logo_url": "https://northwind.studio/logo.svg",
    "accent": "#1a4d3e",
    "locale": "en"
  },
  "client_ref": "northwind",
  "hostnames": ["shop.northwind.com"]
}
```

The `client_ref` is yours: an opaque reference tying the connection to a client in your systems. It flows through every API response and webhook, which is what makes fleet views and per-client billing possible later ([doc 04](04-bulk-operations-and-monitoring.md)).

Use the widget when clients onboard themselves, when you want the fastest possible integration, or when the connection step lives inside a portal you did not build entirely from scratch. The SDK wraps the same flow for tighter programmatic control. Full options in the [docs](https://app.customdomain.ai/docs).

## Path two: the headless API

Some agencies want no third-party UI at all: the connection step should look like every other screen in their own product. The [REST API](https://customdomain.ai/custom-domain-api) makes the entire flow headless. One request per client starts everything:

```bash
curl https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer sk_live_..." \
  -d '{
    "hostname": "shop.northwind.com",
    "client_ref": "northwind",
    "branding": "agency"
  }'
```

The response tells you what happens next:

```json
{
  "hostname": "shop.northwind.com",
  "provider": "detected-provider",
  "client_ref": "northwind",
  "state": "pending_ownership"
}
```

From here you have choices. If the client should authorize interactively, you surface the authorization URL in your own UI and let them approve at their provider. If they gave you a scoped DNS API token, records go in directly with no interactive step. Either way the connection walks the same states:

```
pending_ownership -> verifying -> live
```

Poll the connection, or better, register a webhook and let the transitions come to you. Ownership verification always precedes go-live: traffic is never served on a hostname whose control has not been proven.

## Isolation: the part clients never see but you must get right

A white label fleet has a specific nightmare: client A's configuration, grant, or certificate somehow touching client B. Per-client isolation means each client's domains, provider grants, and certificates are walled off from every other client's, keyed by that same `client_ref`. One client's connection cannot reach into another's, and an offboarding ([doc 01](01-managing-client-domains.md)) cleanly revokes one client without brushing the rest of the fleet. For your own team's access to the fleet console, SSO and SCIM are available, which tends to matter the day your largest client's security questionnaire arrives.

## Widget or API?

| | Widget | API |
|---|---|---|
| Integration effort | Hours | Days |
| UI ownership | Themed component, your brand | Entirely yours |
| Best when | Clients self-serve in your portal | The flow must be invisible inside your product, or driven by back-office tooling |
| Bulk migrations | Not its job | Built for it ([doc 04](04-bulk-operations-and-monitoring.md)) |

Most agencies end up with both: the widget for client self-serve, the API for migrations, automation, and the fleet console. And if your operations run partly through AI agents, the same flows are exposed through a hosted [MCP server](https://customdomain.ai/mcp-server) with the same verification gates.

## About Custom Domain

This guide is maintained by [Custom Domain](https://customdomain.ai), a managed domain-connection platform built for [agencies and white label platforms](https://customdomain.ai/for/agencies-white-label): connection methods covering 63 DNS and registrar providers, more than 25 of them fully auto-configured for one-click provider authorization, automatic ownership verification, and TLS issuance and renewal at a managed edge with strict per-client isolation. Start free at [app.customdomain.ai/signup](https://app.customdomain.ai/signup), or read the [docs](https://app.customdomain.ai/docs).
