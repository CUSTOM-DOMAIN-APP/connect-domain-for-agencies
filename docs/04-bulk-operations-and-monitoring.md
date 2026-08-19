# Bulk Domain Operations and Monitoring for Client Fleets

Five client domains can be managed by hand and memory. Fifty cannot, and the failure is not dramatic: it is a slow accumulation of records nobody remembers writing, certificates nobody remembers renewing, and one Tuesday morning when three client sites are down for three unrelated reasons. This guide covers running client domains as a fleet: connecting in bulk, migrating an existing book of business, catching DNS drift before clients do, and what TLS renewal has to look like at scale.

Foundations are in the [main guide](../README.md); ownership and offboarding are in [doc 01](01-managing-client-domains.md).

## Connecting many domains through the API

Every connection starts with the same request, distinguished by `end_user_ref`:

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

For a migration, there is a batch form that saves you writing the fan-out yourself:

```bash
curl -X POST https://api.customdomain.ai/v1/connections:bulk \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "domains": ["shop.northwind.com", "www.contoso.example", "app.fabrikam.example"],
    "batch_id": "migration-2026-q3"
  }'
```

It runs the same idempotent upsert per domain, up to 200 domains per call, and each domain is independent: a bad one comes back `failed` with a message while the rest continue, and one already connected under this application comes back `already`. The result array is `{ domain, connection_id, status, message }` per entry, which is exactly the shape you want to write straight into a migration tracker.

Detection, record writing (via each client's authorization or token), verification, and certificate issuance follow automatically. Each connection then reports a status:

```
pending -> propagating -> live
```

with `failed` as the terminal case: an automatic rail whose records never appeared within 24 hours, or a manual connection whose records never appeared within 72. A failed connection carries `error_code` of `propagation_timeout` or `setup_incomplete`, and both clear automatically if the records later show up.

Three rules make bulk work sane:

1. **One `end_user_ref` per client, always.** It is the key that groups domains into a per-client view and makes billing and offboarding tractable later. It is set once at create and never mutated. It is a label, not a security boundary; the boundary is tenant plus application ([doc 02](02-white-label-domain-connection.md#isolation-the-part-clients-never-see-but-you-must-get-right)).
2. **One `batch_id` per migration wave.** Every domain connected in one session carries it, so "what did we touch on Tuesday" is a filter rather than a reconstruction.
3. **Webhooks, not polling loops.** At fleet size, "poll every connection every minute" is a self-inflicted rate problem. Register a webhook once and receive every status transition as it happens.

The [API reference](https://docs.customdomain.ai/docs/api-reference) covers connections, DNS records, verification, TLS, drift checks, webhooks, and registrar search and purchase. If your internal tooling includes AI agents, the hosted [MCP server](https://customdomain.ai/mcp-server) exposes the same operations with the same gates, and with no tool that accepts DNS records as input.

## Migrating an existing book of domains

Moving fifty live client domains is a sequencing problem. The order that works:

1. **Inventory before touching anything.** For every domain: current records, current TTLs, registrar, who can authorize changes, expiry date, and any CAA record. Half the value of a migration is discovering the two domains that expire next month and the one whose CAA record will refuse your certificate.
2. **Lower TTLs first, cut over later.** A few days ahead, drop TTLs on the records that will change (300 is typical). Old values then age out of resolver caches in minutes instead of hours on migration day. Do it far enough ahead that the old, longer TTL has already expired everywhere, or it does nothing.
3. **Batch by risk, not alphabet.** First batch: internal and low-traffic domains, under one `batch_id`. Watch them reach `live`. Then the long tail. The flagship client goes in a batch whose failure modes you have already seen.
4. **Verify before you switch anything off.** A connection reaching `live` means every declared record resolves to its exact expected value and the certificate can issue. Keep the old serving path until then; DNS gives you the luxury of running both sides briefly.
5. **Keep the inventory as the offboarding artifact.** The record export from step 1 is the same document [doc 01](01-managing-client-domains.md) asks you to hand over when an engagement someday ends.

## Drift: the fleet's chronic disease

DNS records do not stay put, because you are not the only actor in a client's zone. Real sources of drift, all mundane:

- The client signs up for an email tool and pastes its records over yours by following that vendor's guide.
- An IT cleanup deletes a record nobody on their side recognizes.
- The client migrates registrars and the new zone is reconstructed from memory.
- The domain registration quietly expires and the registrar parks it ([doc 01](01-managing-client-domains.md#renewals-two-different-clocks) covers the expiry clock).

Unmonitored, every one of these is discovered by the client, as an outage, with your name on the invoice.

Connections are enrolled in drift monitoring by default: `monitor` is the one create-time flag that defaults to `true`, so omitting it enrolls the domain and you have to send `false` to opt out. Once a connection is live, an hourly sweep re-checks its declared records against live public DNS and produces `domain.record_missing` when one disappears or is repointed, and `domain.record_restored` when it comes back.

**Read this caveat before you build a runbook on those two events.** Drift alerting is gated per deployment. Where it is off, the sweep still runs but the drift webhooks are not delivered, and the hosted service has historically shipped with that flag off. Confirm the behavior on your own account before you promise a client proactive notification. What always works, and what you can therefore build on unconditionally, is the synchronous check:

```bash
curl -X POST https://api.customdomain.ai/v1/monitor:check \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.northwind.com"}'
```

It runs the same comparison on demand and returns `{ inSync, records[] }`. Running it across the fleet on your own schedule is a cron job and a loop, and it is what we would do first.

The events themselves, when delivered, arrive as a flat signed JSON envelope with an `id`, a `type`, the `domain`, and per-event `data`. Route the missing event into your ticketing with the client attached, and the on-call runbook becomes: read which record broke, check whether the client changed vendors on purpose, either restore via re-authorization or update the engagement. The restored event closes the ticket.

Two more fleet-relevant events are worth subscribing to and are easy to miss:

- `connection.records_outdated` fires when a reconciliation pass finds a connection whose applied records were computed against a superseded edge target. The envelope carries a link to re-run setup. Across fifty client domains this is the difference between one scripted afternoon and fifty support tickets.
- `connection.reapply.failed` fires when a managed connection's re-apply push fails, including the case where the client revoked the stored grant. That is your signal to ask the client for consent again, before anything breaks.

Delivery is worth knowing in detail because at fleet volume you will hit it. A delivery succeeds on any `2xx`. Failures retry with exponential backoff (30 seconds, then 1 minute, then doubling, capped at 6 hours between attempts) for up to 12 attempts spanning about a day, then dead-letter. A non-transient `4xx` other than `429` is not retried at all, since a retry cannot fix a misconfiguration. After a `429` or three consecutive failures, deliveries to that URL are suppressed for about five minutes. Make your endpoint idempotent and deduplicate on the envelope `id`: the same event can arrive twice. Inspect what happened with `GET /v1/webhook-deliveries`.

## TLS renewal at scale

Certificate renewal is where fleet management quietly fails, because it is invisible until it is not: nothing about a working site tells you its certificate expires in nine days. Three facts shape the problem:

1. **Lifetimes are short and shrinking.** Public TLS certificates already have brief lifetimes, and the CA/Browser Forum has adopted a schedule stepping maximum validity down to 47 days by 2029. Any process with a human in the renewal loop is already broken at fleet scale; the industry is making it more broken every year.
2. **Renewal must not depend on re-touching client DNS.** A renewal design that needs a new record in the client's zone every cycle multiplies your drift surface by every renewal of every domain. With the edge terminating TLS, issuance happens after DNS points at the edge and renewals happen at the edge, without touching the client's DNS again.
3. **Failure must fail closed.** If issuance or renewal fails, the correct behavior is to keep retrying and never serve a broken certificate, not to serve traffic with a warning. Verify your provider's failure mode before you need it. In CustomDomain the edge asks the control plane whether a host is authorized to be served, and that answer is driven by connection status, so a host that is not `live` is not served. The control plane also sits outside the request path, so client traffic never waits on an API call, and quota and billing checks live on the write path only rather than on the serving path.

One honest limit on the monitoring side, since this section is about knowing before the client does: what runs is DNS record drift detection, comparing live public DNS against the baseline a connection stores. Nothing probes the client's origin for reachability, status code, or latency. It is not uptime monitoring, and if you are selling uptime as part of the engagement you still need a separate tool for it.

## The fleet dashboard: what to actually watch

| Signal | Meaning | Action |
|---|---|---|
| Connection not `live` after a day | Onboarding stalled | Nudge the client to authorize; check the guided-manual records reached them |
| Connection `failed` | Records never appeared within the window | Read `error_code`: `propagation_timeout` means written but never resolved, `setup_incomplete` means never added |
| `domain.record_missing` | DNS drift, or the registration lapsed | Runbook above; call the client if the domain itself expired |
| `connection.records_outdated` | Applied records predate a change in the edge target | Re-run setup from the link in the envelope, in batches |
| `connection.reapply.failed` | A managed re-apply failed, possibly a revoked grant | Ask the client to re-consent before it breaks |
| Registration expiry within 60 days | Client-side renewal risk | Confirm auto-renew and payment card with the client |
| Domains per `end_user_ref` changed | Someone added or removed a connection | Reconcile with the engagement's scope |

One console view of every client's domains, statuses, and certificates, with drift checked regularly, is the difference between operating a fleet and merely owning one. That console, the widget your clients see ([doc 02](02-white-label-domain-connection.md)), and the API above are the same system wearing three interfaces.

## About CustomDomain

This guide is maintained by [CustomDomain](https://customdomain.ai), a managed domain-connection platform for [agencies and white label platforms](https://customdomain.ai/for/agencies-white-label): connection methods covering 63 providers, 25 of them fully auto-configured (one-click provider authorization or a scoped API token) and 38 guided manual with automatic verification, value-checked DNS verification, TLS issued and renewed at a managed edge that serves only authorized hosts, hourly DNS drift checks with webhooks, and a [REST API](https://customdomain.ai/custom-domain-api) with a served OpenAPI spec. Docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs). Start free at [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
