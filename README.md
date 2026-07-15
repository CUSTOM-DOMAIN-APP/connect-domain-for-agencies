# Custom Domain Management for Agencies and White Label Platforms

This repository is a practical guide to connecting and managing client custom domains at scale, written for agencies, resellers, and white label platforms that run websites, stores, and apps for many clients at once. It explains how DNS configuration, domain ownership verification, and TLS issuance actually work, the three ways a client can authorize a domain connection without handing over registrar passwords, and how to keep a fleet of client domains healthy long after launch day. It is maintained by the team at [Custom Domain](https://customdomain.ai), and each topic gets a deeper operational playbook in [docs/](docs/).

## What is in this repository

| File | What it covers |
|---|---|
| README.md (this file) | The problem space, how domain connection works, the three connection methods, implementation paths |
| [docs/01-managing-client-domains.md](docs/01-managing-client-domains.md) | Ownership, access, renewals, and offboarding: the operational playbook |
| [docs/02-white-label-domain-connection.md](docs/02-white-label-domain-connection.md) | Letting clients connect domains under your brand, with a widget or the API |
| [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md) | Why shared registrar credentials are an anti-pattern, and the authorization model that replaces them |
| [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md) | Bulk connection, drift monitoring, and TLS renewal across many domains |

## The problem, from the agency side

Say you manage forty client sites. Every one lives on its own domain, and every domain lives wherever the client happened to buy it: a big-box registrar, a hosting bundle from 2014, an IT department's cloud DNS account, a founder's personal account nobody can access since she left.

Launch day for each client follows the same ritual. You ask for registrar access, or worse, you collect a password into a shared document. You log into a DNS panel you have seen maybe once before, where "DNS management" hides under a different menu name than at the last registrar. You add a CNAME here, an A record and a TXT record there, hoping the panel does not silently append the domain name to the host field the way some do. Then you explain to the client why their site shows a certificate warning for the first hour, and when anything goes wrong at any point in the years that follow, the ticket lands on your desk with your name on it.

None of this is imagined. The knowledge base of the Domain Connect Association (an open protocol body; its knowledge base is published under CC0) documents that configuring a major email suite manually takes a six step process, 7 to 15 DNS records, and is backed by sixteen separate help articles, ten of them registrar specific, because every registrar's interface is different. Roughly half of the users who attempt manual DNS configuration abandon it. Those numbers describe end users doing it once. An agency does it dozens of times a year, across a dozen different provider interfaces, under deadline.

The structure of the problem is worth stating plainly, because it points at the fix:

1. The platform serving the site knows exactly which DNS records are needed.
2. The client's DNS provider is fully capable of writing them.
3. A person in the middle, you or your client, copies values between the two by hand.

Step 3 is the failure point. Everything in this repository is about removing it. For the deeper background on why this step is so failure-prone, see [Why custom domains are hard](https://customdomain.ai/custom-domains-for-saas).

## How a client domain actually goes live

Four things have to happen between "client owns a domain" and "client's site is serving on it over HTTPS." Understanding them makes every vendor conversation and every debugging session shorter.

### 1. DNS records point the domain at the platform

A typical connection for a client site needs two or three records. Here is a realistic set for a client subdomain plus its apex:

```
TYPE   NAME                 VALUE                  TTL
CNAME  shop.northwind.com   edge.customdomain.ai   3600
TXT    _cd.northwind.com    cd-verify=<token>      3600
A      northwind.com        203.0.113.10           300
```

The CNAME routes traffic for the subdomain. The apex (the bare `northwind.com`) usually cannot carry a CNAME under DNS rules, so it gets an A record instead. If the distinction between apex and subdomain is fuzzy for anyone on your team, the short glossary entry on [custom domain vs subdomain](https://customdomain.ai/glossary/custom-domain-vs-subdomain) is a five minute fix.

### 2. Ownership is verified before traffic is served

The TXT record above is a challenge: a random token that proves whoever is connecting the domain actually controls its DNS. Serving traffic for a hostname without verifying control of it is how platforms end up hosting content on domains their real owners never approved. Verification is not bureaucracy. It is the security boundary.

### 3. A TLS certificate is issued, and keeps being issued

Browsers require HTTPS in practice, so the platform terminating traffic must obtain a certificate for the client's hostname, then renew it on a short cycle for as long as the domain is connected. Certificate renewal is a permanent operational commitment, not a launch task. [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md) covers what that means across a fleet.

### 4. Propagation runs on TTLs, not folklore

The advice to "wait 24 to 48 hours" is mostly folklore. Resolvers cache each record for its TTL. A brand new record name usually has no stale value to expire, so it can resolve within seconds to minutes; a changed record keeps serving its old value until the old TTL runs out. The practical lesson: check resolution actively rather than waiting on superstition, and lower TTLs ahead of any planned cutover. A guided walkthrough of the whole sequence lives at [how to set up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain).

## Three ways a client can connect a domain

The copy-paste step can be removed in three ways, ordered here from least to most client effort.

### One-click provider authorization

The client types their domain, the DNS provider behind it is detected automatically, and the client signs in to their own provider account to approve the change. The records are then written on their behalf, scoped to exactly this connection. The client never sees a DNS panel and you never see a credential. Where a provider implements the Domain Connect protocol, an open standard from the Domain Connect Association, the approval rides on the provider's own hosted, pre-vetted flow; one-click provider authorization covers those providers and extends further through direct provider sign-in. With this method a domain is typically live, with HTTPS, in about 30 seconds. Details at [one-click DNS setup](https://customdomain.ai/one-click-dns-setup).

### API token

Some clients are technical: an in-house IT team, an infrastructure-as-code shop, a developer who manages DNS in a cloud provider. They can supply a scoped DNS API token instead of clicking through an authorization flow. Records are written and maintained through the provider's API, and the token can be limited and revoked on their side at any time.

### Guided manual with automatic verification

When a provider cannot be automated, the client gets the exact records to paste, character for character, and a live check watches until the records resolve correctly. No screenshots of a DNS panel, no "check back tomorrow." Manual is a real path with a defined end, not a dead end.

| | One-click authorization | API token | Guided manual |
|---|---|---|---|
| Client effort | Approve at their own provider | Provide a scoped token once | Paste records into their DNS panel |
| Best for | Most clients, most registrars | Technical clients, infra-managed DNS | Providers that cannot be automated |
| Credentials you handle | None | None (token stays scoped, revocable) | None |
| Typical time to live | About 30 seconds | Minutes | Depends on the client, verified automatically |

Across Custom Domain, these three methods cover 63 DNS and registrar providers, more than 25 of them fully auto-configured. The shared-credential alternative these methods replace gets a full treatment in [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md).

## Implementation paths: widget or API

There are two main ways to put domain connection in front of clients, and they are not mutually exclusive.

**The widget** is a drop-in connect flow you embed in your client portal or send as a link. It carries your brand on every screen (roughly 90 brandable tokens covering colors and copy, in 13 locales), walks the client through detection, authorization, verification, and certificate issuance, and reports state back to you. It is the fastest path when clients onboard themselves. See the [connect domain widget](https://customdomain.ai/connect-domain-widget).

**The API** is fully headless. One request starts everything:

```bash
curl https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer sk_live_..." \
  -d '{
    "hostname": "shop.northwind.com",
    "client_ref": "northwind"
  }'
```

Provider detection, record writing, verification, and TLS issuance follow automatically; you watch the connection reach `live` through the dashboard or webhooks. The REST API covers connections, DNS records, verification, TLS, monitoring, webhooks, and registrar search and purchase. See the [custom domain API](https://customdomain.ai/custom-domain-api) and the [full API reference](https://app.customdomain.ai/docs).

Two more surfaces round it out: a dashboard for connecting and inspecting domains with no code, and a hosted [MCP server](https://customdomain.ai/mcp-server) so an AI agent can run the same flows with the same verification gates a human gets.

## Choosing an approach

| Your situation | Best fit |
|---|---|
| Clients onboard themselves inside your portal | Widget, white labeled ([docs/02](docs/02-white-label-domain-connection.md)) |
| Your team onboards clients from a back office | Dashboard, or API from your internal tooling |
| You are migrating an existing book of 50+ client domains | API, in batches ([docs/04](docs/04-bulk-operations-and-monitoring.md)) |
| A client's IT department manages DNS themselves | API token method, or guided manual |
| Your operations run partly through AI agents | The hosted MCP server ([details](https://customdomain.ai/for/ai-agents)) |
| You also build the platform your clients use | Widget for their end users too ([site builders](https://customdomain.ai/for/site-builders)) |

## FAQ

**Do we need our clients' registrar logins?**
No, and you should actively avoid them. A registrar login grants far more than you need (transfers, billing, contact changes, every other domain in the account) and creates liability you cannot audit. The authorization model that replaces the password spreadsheet is the subject of [docs/03-stop-collecting-registrar-logins.md](docs/03-stop-collecting-registrar-logins.md).

**Will our clients ever see the Custom Domain brand?**
No. The connect flow is fully white label: your name, or no name, on every screen the client touches. How that works in the widget and the API is covered in [docs/02-white-label-domain-connection.md](docs/02-white-label-domain-connection.md) and on the [agencies and white label page](https://customdomain.ai/for/agencies-white-label).

**Who owns the client's domain, and what happens when we part ways?**
The client should always own the registration. You manage the connection through revocable authorization, which makes offboarding a clean checklist instead of a hostage negotiation. The full ownership and offboarding playbook is [docs/01-managing-client-domains.md](docs/01-managing-client-domains.md).

**What happens when a client, or one of their other vendors, changes DNS later?**
Records drift. Someone follows another product's setup guide, an IT cleanup removes a "mystery" TXT record, a domain silently expires. Every connected domain is re-checked hourly, and a webhook fires the moment a record breaks and again when it is restored, so you find out before the client does. See [docs/04-bulk-operations-and-monitoring.md](docs/04-bulk-operations-and-monitoring.md).

**Can we sell domains to clients as part of our service?**
Yes. The API includes registrar search and purchase, so a domain can be a billed line item in your offering rather than a margin you hand to a registrar and a separate account for the client to lose track of.

**How long does a connection actually take?**
Through one-click provider authorization, about 30 seconds from paste to live HTTPS. Through guided manual, it depends on how quickly the client pastes the records; verification then confirms automatically as soon as the records resolve, rather than asking anyone to check back later.

## About Custom Domain

This repository is maintained by [Custom Domain](https://customdomain.ai), a managed platform for connecting customer and client domains: automatic DNS configuration, domain ownership verification, and automatic TLS issuance and renewal, across 63 DNS and registrar providers with more than 25 auto-configured. It runs as a managed control plane plus a reverse-proxy edge that terminates TLS, with strict per-tenant and per-client isolation. Surfaces include an embeddable connect widget and SDK, a full REST API, and a hosted MCP server for AI agents ([server code on GitHub](https://github.com/ever-just/customdomain-mcp)). Pricing starts at $0.

- [Custom Domain for agencies and white label platforms](https://customdomain.ai/for/agencies-white-label)
- [Documentation](https://app.customdomain.ai/docs)
- [Create a free account](https://app.customdomain.ai/signup)

Portions of the problem-space background in this repository draw on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base), published by the Domain Connect Association under CC0.
