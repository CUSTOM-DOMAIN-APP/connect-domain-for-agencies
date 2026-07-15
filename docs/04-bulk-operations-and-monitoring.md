# Bulk Domain Operations and Monitoring for Client Fleets

Five client domains can be managed by hand and memory. Fifty cannot, and the failure is not dramatic: it is a slow accumulation of records nobody remembers writing, certificates nobody remembers renewing, and one Tuesday morning when three client sites are down for three unrelated reasons. This guide covers running client domains as a fleet: connecting in bulk, migrating an existing book of business, catching DNS drift before clients do, and what TLS renewal has to look like at scale.

Foundations are in the [main guide](../README.md); ownership and offboarding are in [doc 01](01-managing-client-domains.md).

## Connecting many domains through the API

Every connection starts with the same request, distinguished by `client_ref`:

```bash
curl https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer sk_live_..." \
  -d '{
    "hostname": "shop.northwind.com",
    "client_ref": "northwind"
  }'
```

Detection, record writing (via each client's authorization or token), ownership verification, and certificate issuance follow automatically. Each connection then reports a state:

```
pending_ownership -> verifying -> live
```

Two rules make bulk work sane:

1. **One `client_ref` per client, always.** It is the key that groups domains into a per-client view, scopes isolation, and makes billing and offboarding tractable later.
2. **Webhooks, not polling loops.** At fleet size, "poll every connection every minute" is a self-inflicted rate problem. Register a webhook once and receive every state transition as it happens.

The [API reference](https://app.customdomain.ai/docs) covers connections, DNS records, verification, TLS, monitoring, webhooks, and registrar search and purchase. If your internal tooling includes AI agents, the hosted [MCP server](https://customdomain.ai/mcp-server) exposes the same operations with the same verification gates.

## Migrating an existing book of domains

Moving fifty live client domains is a sequencing problem. The order that works:

1. **Inventory before touching anything.** For every domain: current records, current TTLs, registrar, who can authorize changes, and expiry date. Half the value of a migration is discovering the two domains that expire next month.
2. **Lower TTLs first, cut over later.** A few days ahead, drop TTLs on the records that will change (300 is typical). Old values then age out of resolver caches in minutes instead of hours on migration day.
3. **Batch by risk, not alphabet.** First batch: internal and low-traffic domains. Watch them reach `live`. Then the long tail. The flagship client goes in a batch whose failure modes you have already seen.
4. **Verify before you switch anything off.** A connection reaching `live` means ownership proved, records resolving, certificate issued. Keep the old serving path until then; DNS gives you the luxury of running both sides briefly.
5. **Keep the inventory as the offboarding artifact.** The record export from step 1 is the same document [doc 01](01-managing-client-domains.md) asks you to hand over when an engagement someday ends.

## Drift: the fleet's chronic disease

DNS records do not stay put, because you are not the only actor in a client's zone. Real sources of drift, all mundane:

- The client signs up for an email tool and pastes its records over yours by following that vendor's guide.
- An IT cleanup deletes a "mystery" TXT record that was your ownership verification.
- The client migrates registrars and the new zone is reconstructed from memory.
- The domain registration quietly expires and the registrar parks it ([doc 01](01-managing-client-domains.md#renewals-two-different-clocks) covers the expiry clock).

Unmonitored, every one of these is discovered by the client, as an outage, with your name on the invoice. Monitored, they are tickets you open before the client notices. Every connected domain is re-checked hourly; when a record drifts, a webhook fires the moment it breaks and another fires when it is restored. Conceptually, your consumer receives something shaped like:

```
POST /your-endpoint
{
  "event": "record broken",
  "hostname": "shop.northwind.com",
  "client_ref": "northwind",
  "detail": "CNAME shop.northwind.com no longer resolves to edge"
}
```

(Exact event names and payloads are in the [API reference](https://app.customdomain.ai/docs).) Route the broken event into your ticketing with the `client_ref` attached, and the on-call runbook becomes: read the detail, check whether the client changed vendors on purpose, either restore via re-authorization or update the engagement. The restore event closes the ticket automatically.

## TLS renewal at scale

Certificate renewal is where fleet management quietly fails, because it is invisible until it is not: nothing about a working site tells you its certificate expires in nine days. Three facts shape the problem:

1. **Lifetimes are short and shrinking.** Public TLS certificates already have brief lifetimes, and the CA/Browser Forum has adopted a schedule stepping maximum validity down to 47 days by 2029. Any process with a human in the renewal loop is already broken at fleet scale; the industry is making it more broken every year.
2. **Renewal must not depend on re-touching client DNS.** A renewal design that needs a new record in the client's zone every cycle multiplies your drift surface by every renewal of every domain. With the edge terminating TLS, issuance happens before the first request and renewals happen at the edge, without touching the client's DNS again.
3. **Failure must fail closed.** If issuance or renewal fails, the correct behavior is to keep retrying and never serve a broken certificate, not to serve traffic with a warning. Verify your provider's failure mode before you need it. Custom Domain fails closed, and the control plane sits outside the request path, so client traffic never waits on an API.

## The fleet dashboard: what to actually watch

| Signal | Meaning | Action |
|---|---|---|
| Connection not `live` | Onboarding stalled or verification pending | Nudge the client to authorize; check guided-manual records |
| `connection.broken` webhook | DNS drift or expiry | Runbook above; call the client if the domain itself expired |
| Certificate state degraded | Issuance retrying | Usually self-heals; investigate if it persists |
| Registration expiry within 60 days | Client-side renewal risk | Confirm auto-renew and payment card with the client |
| Domains per `client_ref` changed | Someone added or removed a connection | Reconcile with the engagement's scope |

One console view of every client's domains, states, and certificates, with drift checked hourly, is the difference between operating a fleet and merely owning one. That console, the widget your clients see ([doc 02](02-white-label-domain-connection.md)), and the API above are the same system wearing three interfaces.

## About Custom Domain

This guide is maintained by [Custom Domain](https://customdomain.ai), a managed domain-connection platform for [agencies and white label platforms](https://customdomain.ai/for/agencies-white-label): connection methods covering 63 providers, more than 25 of them fully auto-configured for one-click provider authorization, automatic ownership verification, TLS issued and renewed at a managed edge that fails closed, hourly drift monitoring with webhooks, and a full [REST API](https://customdomain.ai/custom-domain-api). Start free at [app.customdomain.ai/signup](https://app.customdomain.ai/signup).
