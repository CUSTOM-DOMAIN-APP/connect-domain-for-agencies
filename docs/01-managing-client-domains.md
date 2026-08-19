# Managing Client Domains: Ownership, Access, Renewals, and Offboarding

Domains are the one piece of client infrastructure an agency touches that the client keeps forever. That makes them different from hosting, code, and content: get the ownership and access model wrong on day one, and you inherit years of quiet liability that only surfaces at the worst possible moments, renewal lapses, disputes, and offboarding. This playbook covers the four decisions that matter, in the order they come up.

Back to the [main guide](../README.md).

## Who should own what

The single most common mistake is registering client domains inside the agency's own registrar account. It feels efficient. It is a trap. Use this matrix instead:

| Asset | Should belong to | Why |
|---|---|---|
| Domain registration | The client | It is their brand and a legal asset. If it lives in your account, every dispute, invoice lapse, or acquisition turns into a transfer negotiation. |
| Registrar or DNS account | The client | Bundled with the registration in most cases. Moving DNS hosting into your account couples their infrastructure to your business continuity. |
| The DNS records pointing at the platform | Managed by you, via authorization | This is the part you are operationally accountable for, and the part that should never require a shared password. |
| TLS certificates for connected hostnames | The platform terminating TLS | Certificates must live where the traffic terminates, and their renewal must be automated there. |
| Site and app content | Whatever the contract says | Decide in writing before launch, not during offboarding. |

If a client has no domain yet, resist the shortcut of buying it under your account "for now." Either walk them through registering it themselves, or purchase it on their behalf through a flow that lands ownership with them from the start. CustomDomain's [API](https://customdomain.ai/custom-domain-api) includes registrar search and purchase for exactly this: the domain becomes a line item in your offering, and the ownership records are clean from day one.

## Access: authorization, not credentials

Everything you legitimately need to do with a client's domain amounts to writing and maintaining a small set of DNS records under specific hostnames. A registrar login grants vastly more: transfers, contact changes, billing, deletion, and every other domain in the account.

The sustainable model is delegated authorization. The client approves the connection from their own provider account, the scope of what gets written is fixed and visible, and the grant is revocable without anyone rotating a password. This is important enough that it has its own deep dive: [Stop collecting registrar logins](03-stop-collecting-registrar-logins.md).

Day to day, this means your onboarding step is one link. The client pastes their domain, approves at their own provider, and the records, verification, and certificate are done in about 30 seconds, under your brand ([how white label works](02-white-label-domain-connection.md)).

## Renewals: two different clocks

"The domain expired" and "the certificate expired" are different failures on different clocks, owned by different parties. Agencies that conflate them get burned by both.

| | Domain registration | TLS certificate |
|---|---|---|
| Cycle | Annual (or multi-year) | Months, and industry rules are shortening it further |
| Where it renews | At the client's registrar | At the edge serving the traffic |
| Paid by | The client (ideally on their card, with auto-renew on) | Included in the platform serving the domain |
| Failure mode | Site and email both go dark; redemption fees; possible loss of the name | Browser warnings; traffic effectively stops |
| Your job | Calendar it, confirm auto-renew, confirm the card on file is alive | Verify it is automated, then verify the monitoring |

Two practices prevent nearly all registration-side incidents. First, at onboarding, confirm auto-renew is on and the payment card belongs to the organization, not to an employee who might leave. Second, track expiry dates for the whole fleet in one place and alert at 60 and 30 days. An expired domain is the only failure in this document that can be genuinely unrecoverable.

Certificate renewal, by contrast, should never appear on a human's calendar. A managed edge issues the certificate when the domain connects and renews it for as long as the domain stays connected, without touching the client's DNS again. What that looks like across many domains, including the fail-closed behavior you should demand, is covered in [Bulk operations and monitoring](04-bulk-operations-and-monitoring.md).

One thing that does belong on a checklist, once per client zone: CAA. A `CAA` record naming a certificate authority you are not using will block issuance while every other record looks perfect, and the error surfaces minutes later as a TLS failure rather than as a DNS one. Run `dig +short CAA <domain>` at onboarding. An empty answer means no restriction and is fine.

## Offboarding: the checklist

Client relationships end. A clean offboarding is the strongest evidence that your ownership model was right all along. In order:

1. **Confirm ownership is already with the client.** Registration in their name, registrar account under their email, payment method theirs. If any of these are not true, fix that first, while goodwill still exists.
2. **Lower TTLs ahead of the cutover.** A day or two before migration, drop the TTL on the records that will change (to 300 or so), so the switch is fast and reversible.
3. **Hand over a record inventory.** Export the exact DNS records that exist because of you: type, name, value, TTL. `GET /v1/connections/{id}/records` returns the authoritative set per connection, so this is a script rather than an archaeology project. Their next provider will thank you, and nothing becomes a record someone deletes as a mystery in two years.
4. **Point the domain at its new home, or remove your records.** Coordinate the timing with whoever is receiving the traffic.
5. **Disconnect the domain and revoke the authorization.** `DELETE /v1/connections/{id}` stops traffic serving, stops certificate renewal, and stops monitoring; for a connection in managed mode it first reverts the records it wrote, through the stored grant, before deleting the connection. Any provider grant or remembered API token tied to the engagement gets revoked the same day. See [offboarding in the docs](https://docs.customdomain.ai/docs/connect-flow/offboarding).
6. **Confirm from the outside.** Resolve the domain from a public resolver and load it in a browser. Offboarding ends with verification, exactly like onboarding does.

Notice what is absent: no password changes, no "please remove our staff from your registrar account," no wondering what access is still floating around. That is what the authorization model buys you. If your current offboarding involves a password spreadsheet audit, start with [doc 03](03-stop-collecting-registrar-logins.md).

## Onboarding, compressed

The same discipline, run forward, fits in five lines:

1. Client owns the registration (or buys it through your flow, in their name).
2. Client connects via [one-click provider authorization](https://customdomain.ai/one-click-dns-setup), under your brand.
3. The records are verified against public DNS by exact value, the connection reaches `live`, and the certificate issues. Nothing serves before that.
4. The domain stays enrolled in drift monitoring, with the checks and their delivery caveat covered in [doc 04](04-bulk-operations-and-monitoring.md).
5. Registration expiry goes on the fleet calendar.

A general walkthrough of the connection sequence itself is at [how to set up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain).

## About CustomDomain

This guide is maintained by [CustomDomain](https://customdomain.ai), a managed platform that lets your clients connect their own domains: automatic DNS configuration, value-checked verification, and TLS issuance and renewal across 63 providers, 25 of them fully auto-configured (one-click provider authorization or a scoped API token) and 38 guided manual, fully white label for [agencies and resellers](https://customdomain.ai/for/agencies-white-label). Docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs), free tier at [signup](https://app.customdomain.ai/signup).
