# CustomDomain for agencies and white-label platforms

This repository is a practical guide to connecting and managing client custom domains at scale, written for agencies, resellers, and white label platforms that run websites, stores, and apps for many clients at once. It explains how DNS configuration, proof of control, and TLS issuance actually work, the three ways a client can authorize a domain connection without handing over registrar passwords, and how to keep a fleet of client domains healthy long after launch day. It is maintained by the team at [CustomDomain](https://customdomain.ai), and each topic gets a deeper operational playbook in [docs/](docs/).

## What is in this repository

| File | What it covers |
|---|---|
| README.md (this file) | The problem space, how domain connection works, the three connection methods, pricing, implementation paths |
| [docs/01-managing-client-domains.md](docs/01-managing-client-domains.md) | Ownership, access, renewals, and offboarding: the operational playbook |
| [docs/02-white-label-domain-connection.md](docs/02-white-label-domain-connection.md) | Letting clients connect domains under your brand, with a widget or the API |
| [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md) | Why shared registrar credentials are an anti-pattern, and the authorization model that replaces them |
| [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md) | Bulk connection, drift monitoring, and TLS renewal across many domains |

## The problem, from the agency side

Say you manage forty client sites. Every one lives on its own domain, and every domain lives wherever the client happened to buy it: a big-box registrar, a hosting bundle from 2014, an IT department's cloud DNS account, a founder's personal account nobody can access since she left.

Launch day for each client follows the same ritual. You ask for registrar access, or worse, you collect a password into a shared document. You log into a DNS panel you have seen maybe once before, where "DNS management" hides under a different menu name than at the last registrar. You add a CNAME here and an A record there, hoping the panel does not silently append the domain name to the host field the way some do. Then you explain to the client why their site shows a certificate warning for the first hour, and when anything goes wrong at any point in the years that follow, the ticket lands on your desk with your name on it.

None of this is imagined. The Domain Connect knowledge base (published under CC0) documents that configuring a major email suite manually takes a six step process, 7 to 15 DNS records, and is backed by sixteen separate help articles, ten of them registrar specific, because every registrar's interface is different. Roughly half of the users who attempt manual DNS configuration abandon it. Those numbers describe end users doing it once. An agency does it dozens of times a year, across a dozen different provider interfaces, under deadline.

The structure of the problem is worth stating plainly, because it points at the fix:

1. The platform serving the site knows exactly which DNS records are needed.
2. The client's DNS provider is fully capable of writing them.
3. A person in the middle, you or your client, copies values between the two by hand.

Step 3 is the failure point. Everything in this repository is about removing it. For the deeper background on why this step is so failure-prone, see [Why custom domains are hard](https://customdomain.ai/custom-domains-for-saas).

## How a client domain actually goes live

Four things have to happen between "client owns a domain" and "client's site is serving on it over HTTPS." Understanding them makes every vendor conversation and every debugging session shorter.

### 1. DNS records point the domain at the platform

A typical connection for a client site needs one or two records. Here is a realistic set for a client subdomain plus its apex:

```
TYPE   NAME                 VALUE                  TTL
CNAME  shop.northwind.com   edge.customdomain.ai   3600
A      northwind.com        203.0.113.10           300
```

The CNAME routes traffic for the subdomain, and it is the record you want wherever you have a choice: it follows the target if the platform's addresses ever change. The apex (the bare `northwind.com`) cannot hold a CNAME, because RFC 1034 section 3.6.2 forbids other data at a name that carries one and the apex must carry SOA and NS. So the apex gets one of three things instead: a subdomain used in its place, the provider's own apex alias if it has one (Cloudflare calls it CNAME flattening, others call it ALIAS or ANAME), or A records you accept responsibility for updating. If the distinction between apex and subdomain is fuzzy for anyone on your team, the short glossary entry on [custom domain vs subdomain](https://customdomain.ai/glossary/custom-domain-vs-subdomain) is a five minute fix, and the mechanics are worked through in the docs guide on [setting up a custom domain](https://docs.customdomain.ai/docs/guides/set-up-a-custom-domain).

### 2. Control of the domain is proven before traffic is served

Serving traffic for a hostname without confirming that whoever connected it controls the zone is how platforms end up hosting content on domains their real owners never approved. The proof is not a separate challenge step, and it is worth knowing that, because a lot of vendor documentation (including older versions of this file) describes a TXT ownership token that CustomDomain does not issue.

Control is proven by the mechanism that wrote the records. A provider OAuth authorization, a provider-hosted one-click apply, or a scoped API token each demonstrate that the person approving controls the zone. On the manual path there is no authorization to lean on, so proof is the records themselves appearing in public DNS with the exact values expected: a CNAME must resolve to the intended target and an A record must contain the exact address. "The name resolves to something" is never accepted, because the zone is controlled by the party being verified. The edge refuses to serve a hostname whose connection has not reached `live`. The full model is in the docs under [connections](https://docs.customdomain.ai/docs/concepts/connections) and [security](https://docs.customdomain.ai/docs/security/overview).

### 3. A TLS certificate is issued, and keeps being issued

Browsers require HTTPS in practice, so the platform terminating traffic must obtain a certificate for the client's hostname, then renew it on a short cycle for as long as the domain is connected. Certificate renewal is a permanent operational commitment, not a launch task.

One failure here is worth learning before it bites you, because it looks like success. A CAA record (RFC 8659) lets a domain owner list which certificate authorities may issue for the name, and honoring it is mandatory for public CAs. If a client's zone still carries `CAA 0 issue "digicert.com"` from a vendor relationship that ended in 2019, issuance fails even though every DNS record you added is correct and resolving. The symptom arrives minutes later as a TLS error, in a different part of the stack from where you were working. `dig +short CAA northwind.com` before you promise a launch time. [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md) covers what renewal means across a fleet.

### 4. Propagation runs on TTLs, not folklore

The advice to "wait 24 to 48 hours" is mostly folklore. There is no propagation event: authoritative nameservers have the record the moment it is saved, and each recursive resolver learns about it independently when whatever it holds expires. A brand new record name usually has no stale value to expire, so it can resolve within seconds to minutes; a changed record keeps serving its old value until the old TTL runs out.

There is one counterintuitive consequence. Checking too early makes the wait longer, because a resolver asked for a name before the record exists caches the negative answer, and the lifetime of that negative cache comes from the SOA minimum field (RFC 2308 section 5), commonly an hour. So: lower TTLs ahead of any planned cutover, check against the authoritative nameserver rather than refreshing your own resolver, and key completion off the connection reaching `live` rather than off a timer. A guided walkthrough of the whole sequence lives at [how to set up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain).

## Three ways a client can connect a domain

The copy-paste step can be removed in three ways, ordered here from least to most client effort.

### One-click provider authorization

The client types their domain, the DNS provider behind it is detected automatically, and the client signs in to their own provider account to approve the change. The records are then written on their behalf, scoped to exactly this connection. The client never sees a DNS panel and you never see a credential.

This covers two mechanisms that look identical to the client. Where a provider implements the Domain Connect protocol, an open standard maintained by a community of developers across multiple companies, the approval rides on the provider's own hosted page applying a signed, pre-vetted template. Where a provider exposes an OAuth API instead, the authorization happens at the provider and the callback writes the records with a one-time token that is never stored. With this method a domain is typically live, with HTTPS, in about 30 seconds. Details at [one-click DNS setup](https://customdomain.ai/one-click-dns-setup).

### API token

Some clients are technical: an in-house IT team, an infrastructure-as-code shop, a developer who manages DNS in a cloud provider. They can supply a scoped DNS API token instead of clicking through an authorization flow. The token writes the records once and is discarded afterwards unless the client explicitly opts into having it remembered, and it can be limited and revoked on their side at any time.

### Guided manual with automatic verification

When a provider cannot be automated, the client gets the exact records to paste, character for character, and a live check watches until the records resolve to the expected values. No screenshots of a DNS panel, no "check back tomorrow." Manual is a real path with a defined end, not a dead end. It is also the majority path across the ecosystem, for every vendor, which is why the quality of the manual flow matters more than a coverage headline.

| | One-click authorization | API token | Guided manual |
|---|---|---|---|
| Client effort | Approve at their own provider | Provide a scoped token once | Paste records into their DNS panel |
| Providers (of 63) | 8 (6 provider OAuth, 2 provider-hosted one-click) | 17 | 38 |
| Best for | Clients whose provider supports it | Technical clients, infra-managed DNS | Providers that cannot be automated |
| Credentials you handle | None | None (token stays scoped, revocable, used once by default) | None |
| Typical time to live | About 30 seconds | Minutes | Depends on the client, verified automatically |

Across CustomDomain, these three methods cover 63 DNS and registrar providers. Exactly 25 have an automatic path (17 by scoped API token, 6 by provider OAuth, 2 by provider-hosted one-click setup) and the other 38 are guided manual with automatic verification. Those numbers come from the live census at `GET https://api.customdomain.ai/v1/providers/census`, which is public and worth checking yourself rather than taking from a marketing page.

A note on taxonomy, because it will confuse you when you move between this repository and the product docs: the docs describe **four** connect rails, splitting what this guide calls one-click authorization into provider OAuth and provider-hosted one-click setup. Three methods and four rails are the same system counted two ways. The rail-level view is at [connect flow overview](https://docs.customdomain.ai/docs/connect-flow/overview).

The shared-credential alternative these methods replace gets a full treatment in [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md).

## Implementation paths: widget or API

There are two main ways to put domain connection in front of clients, and they are not mutually exclusive.

**The widget** is a drop-in connect flow you embed in your client portal or send as a link. It carries your brand on every screen (85 raw design tokens for colors, spacing, radii and shadows, plus per-locale copy overrides, your logo, your font, and a hideable powered-by footer), walks the client through detection, authorization, verification, and certificate issuance, and reports state back to you. It is the fastest path when clients onboard themselves. Install it as [`customdomain-js`](https://www.npmjs.com/package/customdomain-js) from npm; the same package is what you call from React. See the [connect domain widget](https://customdomain.ai/connect-domain-widget) and the [SDK reference](https://docs.customdomain.ai/docs/widget-sdk/reference).

**The API** is fully headless. One request starts everything, and the only required field is the domain:

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "shop.northwind.com",
    "end_user_ref": "northwind",
    "batch_id": "migration-2026-q3"
  }'
```

`end_user_ref` is your own identifier for the client this connection belongs to, set once at create and never mutated; `batch_id` correlates every domain connected in one session. Both are optional and both are what make per-client views and migration reconciliation possible later ([doc 04](docs/04-bulk-operations-and-monitoring.md)). The body is decoded with unknown fields rejected, so a typo is a `400` rather than a silently ignored field.

The response is the connection plus its authoritative desired record set:

```json
{
  "id": "con_...",
  "application_id": "app_...",
  "domain": "shop.northwind.com",
  "provider_id": "cloudflare",
  "setup_type": "automatic",
  "status": "pending",
  "records": [
    { "type": "CNAME", "host": "shop.northwind.com", "value": "edge.customdomain.ai", "ttl": 3600 }
  ]
}
```

Provider detection, record writing, verification, and TLS issuance follow from there; you watch the connection walk `pending` to `propagating` to `live` through the console or webhooks. Creation is idempotent per application plus domain, so calling it again on an existing connection replays it with a `200` instead of creating a duplicate. The REST API covers connections, DNS records, verification, TLS, drift checks, webhooks, and registrar search and purchase. See the [custom domain API](https://customdomain.ai/custom-domain-api) and the [full API reference](https://docs.customdomain.ai/docs/api-reference).

Two more surfaces round it out: a console for connecting and inspecting domains with no code, and a hosted [MCP server](https://customdomain.ai/mcp-server) (registered as `ai.customdomain/mcp`) so an AI agent can run the same flows with the same gates a human gets. None of its tools accepts DNS records as input, which is what keeps a prompt-injection path from ending in an arbitrary DNS write.

## Choosing an approach

| Your situation | Best fit |
|---|---|
| Clients onboard themselves inside your portal | Widget, white labeled ([docs/02](docs/02-white-label-domain-connection.md)) |
| Your team onboards clients from a back office | Console, or API from your internal tooling |
| You are migrating an existing book of 50+ client domains | API, in batches ([docs/04](docs/04-bulk-operations-and-monitoring.md)) |
| A client's IT department manages DNS themselves | API token method, or guided manual |
| Your operations run partly through AI agents | The hosted MCP server ([details](https://customdomain.ai/for/ai-agents)) |
| You also build the platform your clients use | Widget for their end users too ([site builders](https://customdomain.ai/for/site-builders)) |

## What it costs, and how it compares

Prices below are the live plan catalog, read from `GET https://api.customdomain.ai/v1/plans` on 2026-08-19.

| Plan | Price | Domain connections included | Notes |
|---|---|---|---|
| Free | $0 | 10 per year, 1 per month | The only tier refused at its quota |
| Startup | $149/mo | 600 per year, 50 per month | Self-serve |
| Growth | $649/mo | 600 per year, 50 per month | Adds the reverse-proxy and certificate APIs |
| Premium | Contact sales | 12,000 per year | Adds drift monitoring as a preview |
| Enterprise | Custom | 12,000+ per year | Adds the white-label entitlement on shared and resumed flows |

Read the monthly number, not the annual one, because that is what the meter compares against: a tier is sold per year and metered per UTC calendar month, at `ceil(domains_per_year / 12)`. For an agency the practical question is how many *new* connections you create in a busy month, not how many domains you manage in total, since an existing connection costs nothing to keep. No paid tier is refused a connection past quota; overage is recorded and surfaced rather than blocked. Free is the only hard cap.

**Compared with Entri.** Entri is the established vendor in this category and the one you are most likely evaluating alongside this. Its published pricing, as read on 2026-08-19, is Startup at $249/mo for 600 domains per year, with Growth, Premium and Enterprise all routed to "Talk to Sales", and no free tier. That is the whole of the price comparison: cheaper paid entry, and a free tier you can trial with. Vendor pricing moves, so re-check both before deciding.

Two things run the other way, and you should weigh them:

- **Entri ships more upstream one-click templates than we do.** In the public [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates) repository, counted on 2026-08-19 across 696 provider domains, goentri.com contributes 77 templates and customdomain.ai contributes 18 (second place). Templates upstream are the public record of provider-hosted one-click coverage, so this is a real gap rather than a framing choice.
- **Nobody has broad one-click coverage, including us.** 38 of the 63 providers in the census have no automated write rail at all, for any vendor, and Domain Connect's own site lists nine live DNS provider implementations. A vendor implying near-universal one-click coverage is describing something the standards do not currently support. The useful evaluation question is not "how many providers" but "how good is the manual path", because the manual path is the majority case for everyone.

The team's own vendor-neutral evaluation checklist, written to be worth reading even if you pick someone else, is at [choosing a custom-domain solution](https://docs.customdomain.ai/docs/guides/choosing-a-custom-domain-solution).

## FAQ

**Do we need our clients' registrar logins?**
No, and you should actively avoid them. A registrar login grants far more than you need (transfers, billing, contact changes, every other domain in the account) and creates liability you cannot audit. The authorization model that replaces the password spreadsheet is the subject of [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md).

**Will our clients ever see the CustomDomain brand?**
Not in a flow you open yourself. When you call the widget's `open()` directly, the branding is your own client-side configuration and the control plane never sees it, so any theme you pass is applied, including hiding the powered-by footer. One case is entitlement-gated: if a connect is forwarded as a share link and a teammate resumes it, branding on that resumed session is stripped back to default styling unless the workspace is on Enterprise. How the rest works in the widget and the API is covered in [docs/02-white-label-domain-connection.md](docs/02-white-label-domain-connection.md) and on the [agencies and white label page](https://customdomain.ai/for/agencies-white-label).

**Who owns the client's domain, and what happens when we part ways?**
The client should always own the registration. You manage the connection through revocable authorization, which makes offboarding a clean checklist instead of a hostage negotiation. The full ownership and offboarding playbook is [docs/01-managing-client-domains.md](docs/01-managing-client-domains.md).

**What happens when a client, or one of their other vendors, changes DNS later?**
Records drift. Someone follows another product's setup guide, an IT cleanup removes a record nobody recognizes, a domain silently expires. Connections are enrolled in drift monitoring by default, and an hourly sweep re-checks each connection's declared records against live public DNS, emitting `domain.record_missing` when one disappears or is repointed and `domain.record_restored` when it comes back. Read the delivery caveat in [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md) before you build a runbook on those webhooks: drift alert delivery is gated per deployment, and the synchronous `POST /v1/monitor:check` is the check that always works.

**Can we sell domains to clients as part of our service?**
Yes. The API includes registrar search and purchase, so a domain can be a billed line item in your offering rather than a margin you hand to a registrar and a separate account for the client to lose track of.

**How long does a connection actually take?**
Through one-click provider authorization, about 30 seconds from paste to live HTTPS. Through guided manual, it depends on how quickly the client pastes the records; verification then confirms automatically as soon as the records resolve, rather than asking anyone to check back later. A connection that never gets its records is marked `failed` after 24 hours on an automatic rail, or 72 hours on the manual path, with an `error_code` saying which.

## About CustomDomain

This repository is maintained by [CustomDomain](https://customdomain.ai), a product of EverJust Company and a managed platform for connecting customer and client domains: automatic DNS configuration, value-checked verification, and automatic TLS issuance and renewal, across 63 DNS and registrar providers, 25 of them auto-configured and 38 guided manual. It runs as a managed control plane plus a reverse-proxy edge that terminates TLS, with tenant and application scoped isolation. Surfaces include an embeddable connect widget and the [`customdomain-js`](https://www.npmjs.com/package/customdomain-js) SDK, a REST API with a served OpenAPI 3.1 spec, signed webhooks, and a hosted MCP server registered as `ai.customdomain/mcp` ([server code on GitHub](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp)). Eighteen of its one-click templates are merged upstream into [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates). Pricing starts at $0.

- [CustomDomain for agencies and white label platforms](https://customdomain.ai/for/agencies-white-label)
- [Documentation](https://docs.customdomain.ai/docs)
- [Create a free account](https://app.customdomain.ai/signup)

Sibling guides in this organization, for readers who wear more than one hat:

- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders), if you also build the platform your clients publish from
- [connect-domain-for-ai-agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents), if agents drive part of your operations
- [connect-domain-for-email-platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms), for the sending-domain side (SPF, DKIM, DMARC)
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains), a curated list of the whole solution space including the alternatives to this one

Portions of the problem-space background in this repository draw on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base), published under the MIT license by a community of developers across multiple companies under CC0.
