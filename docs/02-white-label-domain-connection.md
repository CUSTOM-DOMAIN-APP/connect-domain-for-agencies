# White Label Domain Connection: Your Brand on Every Screen

Agencies and resellers spend years convincing clients that the platform is theirs. Then the domain setup step redirects the client to a screen with somebody else's logo on it, and the story develops a crack: who else is actually behind the thing I pay for? White label domain connection closes that gap. The client pastes their domain, approves the change from their own registrar account, and watches records, verification, and certificate complete in about 30 seconds, under your brand or under no brand at all.

This guide covers what the client experiences, then the two ways to build it: the embeddable widget and the headless API. It assumes the basics from the [main guide](../README.md).

## What "white label" means here, concretely

Not a logo swap. Every screen the client touches during connection is themable: 85 raw design tokens (modal width, colors, radii, shadows) which the quick keys for `colors`, `font`, `borderRadius` and `logo` map onto, a separate per-locale `customCopy` map for wording, icon slot overrides, optional dark-mode tokens, and a `hideLogo` flag that removes the powered-by footer. The vendor's name appears nowhere the client looks.

Two limits are worth knowing before you promise anything:

- **The client's own DNS provider sign-in cannot be branded, deliberately.** That page belongs to their provider, and its familiarity is exactly what makes the authorization trustworthy (more on that trust model in [doc 03](03-stop-collecting-registrar-logins.md)).
- **One case is entitlement-gated.** A `whiteLabel` object you pass to `open()` yourself is your own client-side configuration and is always applied. But if a connect is forwarded as a share link and a teammate resumes it through `loadSharedFlow`, the control plane strips branding from that resumed session unless the workspace is on the Enterprise plan. The flow still completes; it just falls back to default styling. If share links are part of your client handoff, price that in.

Reference for the whole surface: [widget SDK reference](https://docs.customdomain.ai/docs/widget-sdk/reference).

## What the client experiences

1. Inside your portal (or from a link you send), the client is asked to connect a domain.
2. They type `shop.northwind.com`. Their DNS provider is detected as they type.
3. They click Connect and sign in at their own provider to approve. No DNS panel, no record types, no copy-paste.
4. Records are written, verified against public DNS, and the TLS certificate issues. Typically about 30 seconds end to end.
5. They land back in your portal with the domain live. Your brand carried every step.

If their provider cannot be automated, the flow degrades gracefully instead of dead-ending: the client gets exact records to paste and a live check that confirms the moment they resolve. Still your brand, still no ticket.

## Path one: the embeddable widget

The [connect domain widget](https://customdomain.ai/connect-domain-widget) is the whole flow above as a drop-in component: provider detection, the connection methods, verification, certificate status, error states, and retries. You install it as `customdomain-js`, mint a short-lived widget token on your server, embed it, theme it, and listen for the result.

```js
window.customdomain.open({
  applicationId,                 // your application id
  token,                         // short-lived widget JWT, minted server-side
  domain: "shop.northwind.com",  // optional prefill
  applicationName: "Northwind Studio",
  locale: "en",
  endUserRef: "northwind",       // your own id for this client
  whiteLabel: {
    colors: { primary: "#1a4d3e", background: "#ffffff", text: "#111827" },
    logo: "https://example.com/logo.svg",
    borderRadius: "10px",
    hideLogo: true,
    tokens: { "modal-width": "460px" },
  },
  onSuccess: ({ domain, jobId, setupType }) => { /* record it */ },
});
```

The browser never holds an API key. The only credential that reaches it is the widget JWT: short-lived (60 minutes by default), scoped to one application, and optionally bound to a single hostname at mint time, so a token issued for one client's domain cannot be replayed against another's. See [widget tokens](https://docs.customdomain.ai/docs/authentication/widget-tokens).

`endUserRef` is yours: an opaque reference tying the connection to a client in your systems. It is sent to the control plane as the connection's `end_user_ref`, set once at create and never mutated, and it is what makes per-client fleet views and per-client billing tractable later ([doc 04](04-bulk-operations-and-monitoring.md)).

Use the widget when clients onboard themselves, when you want the fastest possible integration, or when the connection step lives inside a portal you did not build entirely from scratch. Pass an array to `prefilledDomain` to walk a client through several domains in one run rather than reopening the widget per domain. Full options in the [docs](https://docs.customdomain.ai/docs/widget-sdk/reference).

## Path two: the headless API

Some agencies want no third-party UI at all: the connection step should look like every other screen in their own product. The [REST API](https://customdomain.ai/custom-domain-api) makes the entire flow headless. One request per client starts everything, and `domain` is the only required field:

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "shop.northwind.com",
    "end_user_ref": "northwind"
  }'
```

The response tells you what happens next:

```json
{
  "id": "con_...",
  "application_id": "app_...",
  "domain": "shop.northwind.com",
  "provider_id": "cloudflare",
  "setup_type": "automatic",
  "status": "pending",
  "end_user_ref": "northwind",
  "records": [
    { "type": "CNAME", "host": "shop.northwind.com", "value": "edge.customdomain.ai", "ttl": 3600 }
  ]
}
```

`setup_type` is chosen by provider detection and tells you which rail to offer. From here you have choices. If the client should authorize interactively, you start the provider authorization and surface its URL in your own UI. If they gave you a scoped DNS API token, you apply the records directly with no interactive step. Either way the connection walks the same states:

```
pending -> propagating -> live
```

with `failed` as the terminal case when records never appear: 24 hours on an automatic rail, 72 hours on the manual path, and an `error_code` of `propagation_timeout` or `setup_incomplete` saying which. A `failed` connection clears itself if the records later show up.

Poll the connection, or better, register a webhook and let the transitions come to you. Verification always precedes go-live: the edge does not serve a hostname whose connection has not reached `live`, and reaching `live` requires every declared record to resolve to its exact expected value, not merely to resolve.

## Isolation: the part clients never see but you must get right

A white label fleet has a specific nightmare: client A's configuration, grant, or certificate somehow touching client B. It is worth being precise about which boundaries are real, because two things here are often described as though they were the same and they are not.

**The real boundary is tenant plus application.** Every object belongs to a tenant, and every connection belongs to an application inside it. A widget JWT is scoped to one application (and optionally one hostname) and can only see that application's connections; an `sk_` API key sees the whole tenant's. Cross-tenant ids return `404` rather than `403`, so an id from another tenant does not even confirm the object exists. If you want a hard wall between clients, give each client its own application and mint tokens per client. That is the configuration that makes "client A cannot reach client B" a property of the system rather than a property of your code.

**`end_user_ref` is attribution, not a wall.** It labels a connection so the console can render who connected it and so you can search and group by client. It does not scope a credential. Do not build an authorization decision on it.

For your own team's access to the console, authorization is by member role on a four-step ladder (`viewer`, `member`, `admin`, `owner`) enforced at the control plane rather than only in the console UI, so a `member` calling the API directly is refused with a `403`. Only an owner can create or demote an owner, and the last owner cannot be removed. Be straight with clients about the limits of that: **SSO and SCIM are not built, and no plan grants them.** Console sign-in is email and password, optional Google sign-in, and WebAuthn passkeys. If a client's security questionnaire requires SAML or SCIM provisioning for vendor consoles, this is a gap you should disclose rather than discover in the review. See [plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas) and [security overview](https://docs.customdomain.ai/docs/security/overview).

## Widget or API?

| | Widget | API |
|---|---|---|
| Integration effort | Hours | Days |
| UI ownership | Themed component, your brand | Entirely yours |
| Best when | Clients self-serve in your portal | The flow must be invisible inside your product, or driven by back-office tooling |
| Bulk migrations | Not its job | Built for it ([doc 04](04-bulk-operations-and-monitoring.md)) |

Most agencies end up with both: the widget for client self-serve, the API for migrations, automation, and the fleet console. And if your operations run partly through AI agents, the same flows are exposed through a hosted [MCP server](https://customdomain.ai/mcp-server), where no tool takes DNS records as input, so records are always computed server-side.

## About CustomDomain

This guide is maintained by [CustomDomain](https://customdomain.ai), a managed domain-connection platform built for [agencies and white label platforms](https://customdomain.ai/for/agencies-white-label): connection methods covering 63 DNS and registrar providers, 25 of them fully auto-configured (one-click provider authorization or a scoped API token) and 38 guided manual, value-checked verification, and TLS issuance and renewal at a managed edge with tenant and application scoped isolation. Start free at [app.customdomain.ai/signup](https://app.customdomain.ai/signup), or read the [docs](https://docs.customdomain.ai/docs).
