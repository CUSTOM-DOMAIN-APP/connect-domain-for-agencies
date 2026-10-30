# Custom domains for agencies

Connect client domains to the sites you host, under your brand, without collecting registrar logins.

[Custom domains for agencies on customdomain.ai](https://customdomain.ai/gh/agencies) · [Agencies in the docs](https://docs.customdomain.ai/docs/use-cases/agencies) · [Start free: first ten domains, no card](https://app.customdomain.ai/signup)

Managing forty client sites means managing forty domains that live wherever the client happened to buy them.
This repository is the operational playbook for that: how a domain actually goes live, the three ways a client
can authorize the change without handing you a registrar password, and how to keep a fleet healthy for years
after launch day. Maintained by [CustomDomain™](https://customdomain.ai); the guidance stands whether or not you
use the product.

## The problem, from the agency side

Launch day follows the same ritual every time. You ask for registrar access, or worse, you collect a password
into a shared document. You log into a DNS panel you have seen once before, where DNS management hides under a
different menu name than it did at the last registrar. You add a CNAME and an A record, hoping the panel does
not silently append the domain to the host field the way some do. Then you explain the certificate warning, and
for years every ticket about it lands on your desk with your name on it.

The numbers are not imagined. The Domain Connect knowledge base (CC0) documents a major email suite whose manual
setup runs 7 to 15 DNS records across a six-step process, backed by 16 help articles, 10 of them
registrar-specific, and reports that roughly half of the users who attempt manual DNS configuration abandon it.
That describes an end user doing it once. An agency does it dozens of times a year, across a dozen different
consoles, under deadline.

The structure points straight at the fix. The platform serving the site knows exactly which records are needed.
The client's DNS provider can write them. A person in the middle copies strings between browser tabs by hand,
and that third step is both the failure point and where the liability lives: a registrar login grants transfers,
billing, contact changes and every other domain in the account, none of which you need and none of which you can
audit. Everything below removes it.

## Quickstart

One request starts a connection. `domain` is the only required field; `end_user_ref` is your own id for the
client, set once at create and never mutated, and `batch_id` correlates every domain connected in one migration
session. Both are what make per-client views and reconciliation possible later.

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.northwind.com",
       "end_user_ref": "northwind",
       "batch_id": "migration-2026-q3"}'
```

The response carries the connection plus its authoritative desired record set, and status walks `pending` to
`propagating` to `live`. Create is idempotent per application plus domain, so a re-run mid-migration replays the
existing connection with a `200` instead of duplicating it. Unknown fields are a `400`. Sign up at
[app.customdomain.ai/signup](https://app.customdomain.ai/signup); every workspace has a default API key on its
**API & SDK** page.

## Client domain workflows without registrar logins

The questions agencies actually search for are concrete: connect a client's GoDaddy domain without their
password, get DNS access at Cloudflare without the whole account, hand the DNS step to the client's IT person.
Four workflows cover them, and none of them puts a registrar login in your hands.

1. **The client approves at their own provider.** For the 7 providers with a one-click rail (Cloudflare,
   DigitalOcean, DNSimple, Netlify, Vercel and WordPress.com by OAuth, NameSilo by Domain Connect), the client
   signs in at the provider and approves one scoped change. You never see a credential.
2. **The client creates a scoped token.** For the 66 providers on the API-token rail (GoDaddy, Amazon Route 53,
   IONOS, Porkbun and Hetzner among them), the client creates a token limited to DNS and pastes it into the
   widget, which uses it once and discards it by default. At GoDaddy that is a Personal Access Token with only
   `domains.domain:read` and `domains.dns:update`; Route 53 can instead grant an access role to CustomDomain™'s
   AWS account.
3. **Someone else finishes it.** Start the connection in the widget, then use **Share** to send a link (it
   expires after 30 days) to whoever controls the client's DNS. They complete the step in the same flow, and
   you get `connection.live` when the domain is serving.
4. **No one can automate this provider.** For the other 116 providers in the census, the widget names the
   provider, shows the exact records in that provider's own terms, and verifies them in public DNS as they
   appear. The client or their IT adds them; you watch the status.

Tag every connection with your own client id (`end_user_ref`) and every migration with a `batch_id`, as in the
Quickstart: every connection `GET /v1/connections` returns carries both, and the console shows which client
connected each domain.

If you would rather stay inside the registrars' own tools, two of them have a real answer: GoDaddy's
**Delegate Access** invites you into the client's account without their password, and Cloudflare account
members can hold roles scoped to one domain. Both work. Both still leave you editing records by hand for every
client, and GoDaddy's access levels reach well beyond DNS.

## In this guide

- **Ownership, access, renewals and offboarding**, as a checklist rather than a hostage negotiation. [docs/01](docs/01-managing-client-domains.md)
- **Connecting under your brand**, with the widget or the API, and exactly what is entitlement-gated. [docs/02](docs/02-white-label-domain-connection.md)
- **The case against shared registrar logins**, and the authorization model that replaces them. [docs/03](docs/03-stop-collecting-registrar-logins.md)
- **Bulk connection, drift monitoring and TLS renewal** across many domains at once. [docs/04](docs/04-bulk-operations-and-monitoring.md)

## How it works

Four things happen between "client owns a domain" and "client's site serves HTTPS on it": records point the name
at the platform edge, control of the zone is proven, a certificate is issued and then renewed forever, and
caches expire. The apex is the part that surprises people. A bare domain cannot hold a CNAME, because RFC 1034
§3.6.2 forbids other data alongside one and the apex must carry SOA and NS, so it takes a provider alias (CNAME
flattening, ALIAS, ANAME) or A records you accept responsibility for updating.

The copy-paste step can be removed three ways. The split below is the live census at `GET
https://api.customdomain.ai/v1/providers/census`, counted 2026-10-30.

| Rail | Providers (of 189) | Client effort | Credentials you handle |
|---|---|---|---|
| One-click provider authorization | 7 (6 provider OAuth, 1 provider-hosted Domain Connect) | Approve inside their own provider session | None |
| Scoped API token | 66 | Supply one DNS-scoped token, used once by default | None |
| Guided manual with automatic verification | 116 | Paste the exact records for their detected provider | None |

Control is proven by whichever rail wrote the records; on the manual path, proof is the records resolving to the
exact expected values, never "the name resolves to something". There is no TXT ownership token. The product docs
split one-click into provider OAuth and provider-hosted Domain Connect and call it four rails: same system,
counted two ways ([connect flow overview](https://docs.customdomain.ai/docs/connect-flow/overview)).

## White-label and the API surface

The widget carries your brand on every screen: 85 raw design tokens for colours, spacing, radii and shadows,
per-locale copy overrides, your logo, your font, and a hideable powered-by footer. When you call `open()`
yourself the theme is client-side configuration the control plane never sees, so it applies in full. One case is
gated: if a connect is forwarded as a share link and a teammate resumes it, branding on that resumed session
falls back to default unless the workspace is on Enterprise.

Connections are enrolled in drift monitoring by default: an hourly sweep re-checks every declared record against
live public DNS, because records do get deleted by an IT cleanup or another vendor's setup guide months later.
The rest of the surface (connections, records, TLS, webhooks and domain search) is in the [API
reference](https://docs.customdomain.ai/docs/api-reference). Buying a domain is live in the console only.

## Where this loses

Plan prices and allowances are in the facts block below. Read the monthly number, not the annual one. A tier is
sold per year and metered per UTC calendar month at `ceil(domains_per_year / 12)`, so the question is how many
*new* connections you create in a busy month. Keeping an existing one costs nothing. Entri is the vendor you are
most likely comparing against. Read 2026-08-19, its entry tier is $249/mo for 600 domains a year, with no free
tier. Where Entri is ahead: in [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates),
goentri.com ships 77 one-click templates and customdomain.ai ships 18 (counted 2026-09-26). That gap is real and
it is theirs.

## Limits and known gaps

- **Nobody has broad one-click coverage, including us.** 116 of the 189 known providers have no automated write rail at all, for any vendor. The useful evaluation question is how good the manual path is, not how many providers a brochure claims.
- **Drift webhooks are off by default.** The hourly sweep runs, but `domain.record_missing` and `domain.record_restored` are gated behind a deployment flag the hosted service leaves off. Build alerting on `POST /v1/monitor:check` until that changes.
- **A stale CAA record blocks issuance silently.** RFC 8659 CAA is honored by every public CA, so `CAA 0 issue "digicert.com"` from a 2019 vendor relationship fails certificate issuance while every DNS record you added is correct. Run `dig +short CAA northwind.com` before you promise a launch time.
- **SSO and SCIM are not built.** No tier grants them.

<!-- facts:start -->
## Facts and sources

Read from the live endpoints on 2026-10-30. Every number in this README traces to a row here or to a source named
where it is used. If one does not, open an issue with the file and line.

| Fact | Value | Source |
|---|---|---|
| DNS providers known | 189 | [`/v1/providers/census`](https://api.customdomain.ai/v1/providers/census) |
| Automatic path | 73: 66 by scoped API key, 6 by provider sign-in (OAuth), 1 by one-click setup (Domain Connect) | census `mode` |
| Exact records and provider steps | 116 | census `mode` |
| Going live | No TXT ownership record. A domain goes `live` when every record resolves. Statuses: `pending`, `propagating`, `live`, `failed` | [Connections](https://docs.customdomain.ai/docs/concepts/connections) |
| HTTPS on the CustomDomain™ edge | Growth and up. On Starter and Startup, domains point at your servers and you serve TLS | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Plans | Starter $10 a month, 10 domains a year, hard cap. Startup $149 a month, 600 a year. Growth $649 a month, 600 a year. Premium and Enterprise by contract, 12,000 a year. 14 day trial on Starter, Startup and Growth | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Free to start | A new workspace connects its first ten domains free. No card needed to sign up | [Plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas) |
| REST API | 163 operations on 140 paths, OpenAPI 3.1.0 | [`/v1/openapi.json`](https://api.customdomain.ai/v1/openapi.json) |
| MCP server | `customdomain-mcp` 0.6.0, 20 tools. Buying a domain through MCP is off on the hosted service | [`mcp.json`](https://mcp.customdomain.ai/.well-known/agent/mcp.json) |

Check the provider split yourself:

```sh
curl -s https://api.customdomain.ai/v1/providers/census \
  | python3 -c 'import json,sys,collections; d=json.load(sys.stdin); print(d["count"], collections.Counter(p["mode"] for p in d["providers"]))'
# 189 Counter({'manual': 116, 'api': 66, 'oauth': 6, 'dc': 1})
```
<!-- facts:end -->

## Related

- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders): the field guide for the platforms many agencies build client sites on
- [customdomain-sdk](https://github.com/CUSTOM-DOMAIN-APP/customdomain-sdk): the browser SDK `customdomain-js` and the React wrapper `@customdomain/react`, for the branded widget
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, including the alternatives to this product

## Support and license

Problem framing draws on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0).

Questions go to [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions) and corrections to
[this repository's issues](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies/issues). Account and billing:
connect@customdomain.ai. Report security issues privately to security@customdomain.ai, never in a public issue
([policy](https://app.customdomain.ai/security)).

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
