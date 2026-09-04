# Custom Domain for Agencies

White-label custom domains for agencies — connect client domains to the platforms you build on

**Status:** Maintained guide · every number re-verified against the live API on 2026-09-04 · public

[![docs](https://img.shields.io/badge/docs-docs.customdomain.ai-1c1917?style=flat)](https://docs.customdomain.ai/docs)
[![license](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](./LICENSE)

|  |  |
|---|---|
| **What it is** | Operational guide to connecting and running client domains at scale, under your brand |
| **Who it's for** | Agencies, resellers and white-label platforms running sites for many clients |
| **Live at** | [customdomain.ai/for/agencies-white-label](https://customdomain.ai/for/agencies-white-label) · docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| **Stack** | Markdown guide · REST API, OpenAPI 3.1 · embeddable widget (`customdomain-js`) |
| **Status** | Maintained · 63 providers and 5 plans re-counted from the live API 2026-09-04 |

Managing forty client sites means managing forty domains that live wherever the client happened to
buy them. This repository is the operational playbook for that: how a domain actually goes live,
the three ways a client can authorize the change without handing you a registrar password, and how
to keep a fleet healthy for years after launch day. Maintained by
[Custom Domain](https://customdomain.ai); the guidance stands whether or not you use the product.

## The problem, from the agency side

Launch day follows the same ritual every time. You ask for registrar access, or worse, you collect
a password into a shared document. You log into a DNS panel you have seen once before, where DNS
management hides under a different menu name than it did at the last registrar. You add a CNAME and
an A record, hoping the panel does not silently append the domain to the host field the way some
do. Then you explain the certificate warning, and for years every ticket about it lands on your
desk with your name on it.

The numbers are not imagined. The Domain Connect knowledge base (CC0) documents a major email suite
whose manual setup runs 7 to 15 DNS records across a six-step process, backed by 16 help articles,
10 of them registrar-specific, and reports that roughly half of the users who attempt manual DNS
configuration abandon it. That describes an end user doing it once. An agency does it dozens of
times a year, across a dozen different consoles, under deadline.

The structure points straight at the fix. The platform serving the site knows exactly which records
are needed. The client's DNS provider can write them. A person in the middle copies strings between
browser tabs by hand, and that third step is both the failure point and where the liability lives:
a registrar login grants transfers, billing, contact changes and every other domain in the account,
none of which you need and none of which you can audit. Everything below removes it.

## Quickstart

One request starts a connection. `domain` is the only required field; `end_user_ref` is your own
id for the client, set once at create and never mutated, and `batch_id` correlates every domain
connected in one migration session. Both are what make per-client views and reconciliation possible
later.

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.northwind.com",
       "end_user_ref": "northwind",
       "batch_id": "migration-2026-q3"}'
```

The response carries the connection plus its authoritative desired record set, and status walks
`pending` to `propagating` to `live`. Create is idempotent per application plus domain, so a re-run
mid-migration replays the existing connection with a `200` instead of duplicating it. Unknown
fields are a `400`. Free-tier keys: [app.customdomain.ai/signup](https://app.customdomain.ai/signup).

## What it does

- **Ownership, access, renewals and offboarding**, as a checklist rather than a hostage negotiation. [docs/01](docs/01-managing-client-domains.md)
- **Connecting under your brand**, with the widget or the API, and exactly what is entitlement-gated. [docs/02](docs/02-white-label-domain-connection.md)
- **The case against shared registrar logins**, and the authorization model that replaces them. [docs/03](docs/03-stop-collecting-registrar-logins.md)
- **Bulk connection, drift monitoring and TLS renewal** across many domains at once. [docs/04](docs/04-bulk-operations-and-monitoring.md)
- **Publishes every number with its source**, so you can re-count before trusting it.

## How it works

Four things happen between "client owns a domain" and "client's site serves HTTPS on it": records
point the name at the platform edge, control of the zone is proven, a certificate is issued and
then renewed forever, and caches expire. The apex is the part that surprises people. A bare domain
cannot hold a CNAME, because RFC 1034 §3.6.2 forbids other data alongside one and the apex must
carry SOA and NS, so it takes a provider alias (CNAME flattening, ALIAS, ANAME) or A records you
accept responsibility for updating.

The copy-paste step can be removed three ways. The split below is the live census at
`GET https://api.customdomain.ai/v1/providers/census`, counted 2026-09-04.

| Rail | Providers (of 63) | Client effort | Credentials you handle | Time to live |
|---|---|---|---|---|
| One-click provider authorization | 8 (6 provider OAuth, 2 provider-hosted Domain Connect) | Approve inside their own provider session | None | About 30 seconds |
| Scoped API token | 17 | Supply one DNS-scoped token, used once by default | None | Minutes |
| Guided manual with automatic verification | 38 | Paste the exact records for their detected provider | None | Verified automatically |

Control is proven by whichever rail wrote the records; on the manual path, proof is the records
resolving to the exact expected values, never "the name resolves to something". There is no TXT
ownership token. The product docs split one-click into provider OAuth and provider-hosted Domain
Connect and call it four rails: same system, counted two ways
([connect flow overview](https://docs.customdomain.ai/docs/connect-flow/overview)).

## White-label and the API surface

The widget carries your brand on every screen: 85 raw design tokens for colours, spacing, radii and
shadows, per-locale copy overrides, your logo, your font, and a hideable powered-by footer. When
you call `open()` yourself the theme is client-side configuration the control plane never sees, so
it applies in full. One case is gated: if a connect is forwarded as a share link and a teammate
resumes it, branding on that resumed session falls back to default unless the workspace is on
Enterprise.

Connections are enrolled in drift monitoring by default: an hourly sweep re-checks every declared
record against live public DNS, because records do get deleted by an IT cleanup or another vendor's
setup guide months later. The rest of the surface (connections, records, TLS, webhooks, registrar
search and purchase) is in the [API reference](https://docs.customdomain.ai/docs/api-reference).

## Repository layout

```text
.
├── README.md  # the problem, the three rails, white-label, pricing
├── AGENTS.md  # machine-readable brief for coding agents working in this repo
├── docs/      # 01 client domain operations · 02 white-label connection
│              # 03 stop collecting registrar logins · 04 bulk operations and monitoring
└── LICENSE    # MIT
```

Markdown only, no build step. Surfaces referenced from here: REST at `api.customdomain.ai`
(OpenAPI 3.1: 67 paths, 79 operations, counted 2026-09-04), the widget published as
[`customdomain-js`](https://www.npmjs.com/package/customdomain-js), and a hosted MCP server at
`mcp.customdomain.ai/mcp` (registry id `ai.customdomain/mcp`) for agent-driven operations.

## Pricing, and where this loses

Read from `GET https://api.customdomain.ai/v1/plans` on 2026-09-04.

| Plan | Price | Connections included | Notes |
|---|---|---|---|
| Free | $0 | 10/yr, 1/month | The only tier refused at its quota |
| Startup | $149/mo | 600/yr, 50/month | Self-serve |
| Growth | $649/mo | 600/yr, 50/month | Adds the reverse-proxy and certificate APIs |
| Premium / Enterprise | Contact sales | 12,000/yr | Adds monitoring preview, then the white-label entitlement |

Read the monthly number, not the annual one: a tier is sold per year and metered per UTC calendar
month at `ceil(domains_per_year / 12)`, so the question is how many *new* connections you create in
a busy month. Keeping an existing one costs nothing. Entri is the vendor you are most likely
comparing against; read 2026-08-19, its entry tier is $249/mo for the same 600 a year with no free
tier. Where Entri is ahead: across 696 provider domains in
[Domain-Connect/Templates](https://github.com/Domain-Connect/Templates), goentri.com ships 77
one-click templates and customdomain.ai ships 18. That gap is real and it is theirs.

## Limits and known gaps

- **Nobody has broad one-click coverage, including us.** 38 of the 63 catalogued providers have no automated write rail at all, for any vendor. The useful evaluation question is how good the manual path is, not how many providers a brochure claims.
- **Drift webhooks are off by default.** The hourly sweep runs, but `domain.record_missing` and `domain.record_restored` are gated behind a deployment flag the hosted service leaves off. Build alerting on `POST /v1/monitor:check` until that changes.
- **A stale CAA record blocks issuance silently.** RFC 8659 CAA is honoured by every public CA, so `CAA 0 issue "digicert.com"` from a 2019 vendor relationship fails certificate issuance while every DNS record you added is correct. Run `dig +short CAA northwind.com` before you promise a launch time.
- **SSO and SCIM are not built.** No tier grants them.

## Corrections

Every number here traces to a live endpoint or a public repository, named at the point of use. If
one does not, that is a bug: open an issue with the file and line.

Sibling guides: [for AI agents](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents) ·
[for website builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders) ·
[for email platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms) ·
[awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains), which lists
the alternatives to this one. Problem framing draws on the
[Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0), an
open standard maintained by a community across multiple companies and referenced here as prior art.

## License

[MIT](./LICENSE) © CustomDomain.ai, a product of EverJust Company.
