# Stop Collecting Registrar Logins: The Shared Credential Anti-Pattern

Nearly every agency has the spreadsheet. Client name in one column, registrar in the next, then a username, a password, and sometimes a cell helpfully labeled "2FA code, ask Dana." It exists because it solved a real problem on a deadline. It persists because nobody wants to be the one to open that conversation with forty clients. This guide is about why the spreadsheet has to go, and about the delegated authorization model that actually replaces it, without giving up the ability to manage client DNS.

Context on the broader problem space is in the [main guide](../README.md).

## Why the spreadsheet exists

The origin is innocent. A client's site cannot launch until DNS records are created at their registrar. The client does not know what a CNAME is, and asking them to follow a fifteen-step help article fails about half the time (the Domain Connect Association's knowledge base, published under CC0, documents manual DNS abandonment at roughly 50 percent). So the agency says the pragmatic thing: "just send us your registrar login and we'll handle it." The launch happens. The credential stays.

Multiply by every client and every registrar, and the spreadsheet becomes load-bearing infrastructure that no one designed and everyone depends on.

## What you need versus what a login grants

The mismatch is the core of the problem. To launch and operate a client site, you need to write and maintain a handful of DNS records under specific hostnames. A registrar login grants control of everything:

| You actually need | A shared registrar login grants |
|---|---|
| Create 2 or 3 records (CNAME, A, TXT) under agreed hostnames | Create, edit, or delete any record in any zone in the account |
| Keep those records correct over time | Transfer domains away, change ownership contacts |
| Nothing else | Access billing and stored payment methods |
| | Every other domain in the account, including ones you have no engagement for |
| | Email forwarding, WHOIS privacy, account closure |

Security people call this a total failure of least privilege. You are holding the keys to assets you were never asked to manage, and you cannot give back part of a password.

## The risks, concretely

**No audit trail.** Registrar activity logs show the account owner did everything. When a record changes unexpectedly, you cannot distinguish your intern, your former contractor, the client's nephew, or an attacker. Every incident review starts from zero.

**Staff turnover means mass rotation, which means nobody rotates.** The day anyone with spreadsheet access leaves, every credential in it is compromised in policy terms. Rotating forty registrar passwords requires forty client conversations, so in practice it does not happen, and everyone quietly knows it.

**2FA gets weakened to make sharing work.** Shared logins and two-factor authentication are structurally incompatible, so teams either disable 2FA on client accounts (making the client less safe than before they hired you) or share TOTP seeds around, which defeats the point.

**You become the breach.** If your systems are compromised, the blast radius now includes the master keys to dozens of client domains: the asset that controls their web traffic, their email routing (MX), and their email authentication (SPF, DKIM, DMARC). A hijacked domain is one of the most damaging things that can happen to a small business, and your spreadsheet is a map to all of them.

**Offboarding is fiction.** When an engagement ends, "we deleted the row" is not revocation. Only a password change is, and it is in the client's hands, not yours. Compare that with the clean offboarding checklist in [doc 01](01-managing-client-domains.md).

## The alternative: delegated authorization

The fix is a different shape of access. Instead of you logging in as the client, the client approves a specific, scoped change from inside their own provider account:

1. The client types their domain into a connect flow (yours, white labeled: see [doc 02](02-white-label-domain-connection.md)).
2. Their DNS provider is detected automatically.
3. They sign in at their own provider, on the provider's own page, with their own credentials and their own 2FA intact.
4. The provider shows what will change, and the client approves it.
5. The scoped records are written. No credential ever crosses the boundary.

This is not a proprietary trick; the industry has been standardizing it. The Domain Connect protocol, an open standard from the Domain Connect Association, formalizes exactly this pattern for participating providers: service templates are vetted by the DNS provider in advance, the consent screen is rendered by the provider rather than the requesting service, changes are limited strictly to the approved template's scope, and the user can cancel at any point. The protocol's published security model is worth reading in full in the Association's CC0 [knowledge base](https://github.com/Domain-Connect/knowledge-base): its central property is that the DNS provider never has to trust the requesting service at runtime, only the pre-approved template.

Note what the client's sign-in step became in this model. In the spreadsheet world, the provider login was the thing you extracted from clients. In the authorization world, it is the security checkpoint: proof that the person approving the change controls the zone.

## How Custom Domain applies this

Custom Domain's one-click provider authorization implements the delegated model. Where a provider supports a hosted one-click flow, the approval rides on it; elsewhere, direct provider sign-in achieves the same scoped, consented write. Two more paths complete the coverage, as described in the [main guide](../README.md#three-ways-a-client-can-connect-a-domain):

- **API token**, for technical clients who prefer to mint a scoped, revocable DNS token themselves.
- **Guided manual with automatic verification**, for providers that cannot be automated: exact records, a live check, and no credential involved at all.

In all three, ownership is verified before any traffic is served, and the certificate issues automatically. Together the three methods cover 63 DNS and registrar providers, more than 25 of them fully auto-configured, which is broader coverage than any single protocol's adoption alone. The mechanics are on the [one-click DNS setup page](https://customdomain.ai/one-click-dns-setup).

## Retiring the spreadsheet

1. Inventory it honestly: every credential, every account, every person with access.
2. For each active client, send the connect link and let them re-establish the records via authorization. It takes them about 30 seconds each.
3. As each domain verifies and goes live under the new model, ask the client to change that registrar password and turn 2FA back on. Frame it as an upgrade you are giving them, because it is.
4. Delete the spreadsheet. Actually delete it, including the copy in the old shared drive.
5. Point your monitoring at the fleet ([doc 04](04-bulk-operations-and-monitoring.md)) so drift gets caught by webhooks instead of by client phone calls.

## About Custom Domain

This guide is maintained by [Custom Domain](https://customdomain.ai), a managed platform for connecting client domains without collecting credentials: one-click provider authorization, automatic ownership verification, and automatic TLS issuance and renewal, fully white label for [agencies](https://customdomain.ai/for/agencies-white-label). Docs at [app.customdomain.ai/docs](https://app.customdomain.ai/docs), free tier at [signup](https://app.customdomain.ai/signup).
