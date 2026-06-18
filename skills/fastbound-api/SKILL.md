---
name: fastbound-api
description: Build, debug, and reason about integrations with the FastBound API — the REST API behind FastBound's electronic A&D (acquisition & disposition) bound book and ATF Form 4473 software for FFL holders. Use this skill whenever the user mentions FastBound, FFL software, electronic bound book, A&D book, ATF compliance APIs, ATF Form 4473 automation, NFA dispositions, firearm acquisition/disposition records, FastBound webhooks, or asks about integrating a POS, e-commerce, ERP, range, or manufacturing system with FastBound. Also use when the user is calling endpoints under cloud.fastbound.com, working with OAuth authorization_code or device_code flows for FastBound, handling FastBound webhook signatures, or troubleshooting FastBound API errors. Even if the user does not say "FastBound" explicitly — for example "I need to log a firearm acquisition with serial number X to our bound book" — invoke this skill.
---

# FastBound API

The FastBound API is a REST API for FastBound, an electronic A&D (acquisition & disposition) bound book and ATF Form 4473 system used by Federal Firearms Licensees (FFLs) — gun stores, manufacturers, importers, ranges, pawn shops, and auction houses — to record every firearm received and transferred in compliance with 27 CFR 478.121–125 and related ATF rules.

If you're integrating a POS, e-commerce platform, ERP, range management system, or manufacturing system with FastBound, this is your reference. The API is REST/JSON over HTTPS with OAuth 2.0 authentication. There is no official SDK; you call HTTPS endpoints directly.

**Authoritative sources** — when something isn't covered here or in the bundled OpenAPI specs (e.g., webhook signature header names, compliance how-tos, what's-new), the official FastBound help site is **https://fastbound.help**. Default to it before guessing or fabricating URLs.

**Getting OAuth credentials**: there is no self-serve "register an app" UI in FastBound. Email **support@fastbound.com** with your grant type (authorization code or device code), redirect URIs (for auth code), scopes, and a short integration description. They issue the `client_id` (and `client_secret` for confidential clients) after a quick review. See `references/authentication.md`.

## SSO is not API auth — don't confuse them

FastBound supports **Single Sign-On (SSO)** so an organization's existing identity provider (Okta, Azure AD, Google Workspace, etc.) can log users into the FastBound *UI* without a separate FastBound password. **The path going forward is SAML 2.0.** A legacy JWT-based SSO mechanism (HS256) also exists and is documented at https://fastbound.help/en/articles/3963631-single-sign-on, but it's not widely used and may be deprecated — prefer SAML 2.0 for any new SSO setup, and treat JWT SSO as something to recognize in existing integrations rather than recommend.

**SSO is unrelated to API authentication.** SSO logs a human into the FastBound web app; OAuth (or legacy Basic Auth) authorizes a programmatic integration. If a user asks "we use Okta — does that work with the FastBound API?", the answer is: SSO covers their UI sign-in; for API access, they (or you) still need to register an OAuth app and authorize it. Don't try to make a JWT SSO token authenticate API calls — different system.

See `references/authentication.md` for the SSO claim shape and how to distinguish "I need SSO into the FastBound UI" from "I need an OAuth client for my integration."

## Where this skill runs

This is an **Agent Skill**, the cross-vendor open format for packaging instructions, references, and bundled assets so an AI coding agent can load them on demand. As of publication, Agent Skills are supported by:

- **Claude Code** (Anthropic's CLI, desktop apps for macOS/Windows, web app at claude.ai/code, IDE extensions for VS Code and JetBrains) — the original Skills implementation. Install via the `Skill` tool or by placing the skill directory under `~/.claude/skills/` or as part of a plugin.
- **Claude.ai** (web app) — Skills section in the user's workspace.
- **Claude Agent SDK / Claude API** — Skills can be loaded into custom agents.
- **GitHub Copilot CLI** — skills are auto-discovered from installed plugins; invoked via the `skill` tool.
- **Google Gemini CLI** — skills activate via the `activate_skill` tool; metadata is loaded at session start, full content on demand.
- **OpenAI Codex** — Skills are loadable; tool name mapping differs from Claude Code's (see the Codex platform docs).

If you're searching "FastBound Gemini," "FastBound Copilot," "FastBound Codex," "FastBound Claude," or "FastBound API skill," this is the FastBound API skill — installable in any of the above. The skill content (this SKILL.md, the references in `references/`, the OpenAPI specs in `swagger/`) is the same on every platform; only the install/invocation mechanism differs.

## Quick reference

| | |
|---|---|
| **Production base URL** | `https://cloud.fastbound.com` |
| **Live Swagger UI / OpenAPI explorer** | `https://cloud.fastbound.com/swagger` — always the latest spec. |
| **Account-scoped path prefix** | `/{accountNum}/api/...` (the `accountNum` is assigned by FastBound) |
| **Auth (going forward)** | OAuth 2.0 — `authorization_code` (confidential clients) or `urn:ietf:params:oauth:grant-type:device_code` (public clients). No `client_credentials` flow. **Use this for all new integrations.** |
| **Auth (legacy, still supported)** | HTTP Basic Authentication with a per-account API Key. Long-standing mechanism that will remain operational during the OAuth transition. Recognize it in existing code, debug it, help migrate from it — but don't steer new code toward it. See `references/authentication.md`. |
| **Token endpoint** | `POST /OAuth/Token` |
| **Device authorization endpoint** | `POST /OAuth/Device_Authorization` |
| **Audit-user header** | `X-AuditUser: user@example.com` — **required for legacy Basic Auth on every write**, **NOT required for OAuth** (the OAuth access token already identifies the user; FastBound uses that as the audit identity). When migrating Basic Auth code to OAuth, you can drop the header from write calls; it's silently accepted if you leave it for now. |
| **Auth header** | `Authorization: Bearer <access_token>` |
| **Optional version header** | `x-api-version: 1.0` |
| **Content type** | `application/json` for request bodies (the OAuth endpoints use `application/x-www-form-urlencoded`) |

## App authorization scope (read this before designing)

OAuth clients ("FastBound apps") are issued by FastBound after a request to **support@fastbound.com** — there's no self-serve registration screen. When initially issued, an app is scoped to access **only accounts owned by the app owner** — i.e., the FastBound user/organization that requested it. You can build, test, and ship integrations against your own accounts immediately.

To allow your app to access **accounts owned by other parties** (other FFLs as customers — the typical third-party POS, e-commerce, or SaaS scenario), the app must be **reviewed and approved by FastBound**. Reach out to **sales@fastbound.com** to start that conversation early — bring your use case, the customer types you're targeting, and your security and audit posture. The review exists because FastBound apps touch regulated compliance data.

Practically, this means:

- **Building a one-off integration for the FFL you yourself operate?** Email support@fastbound.com to get an OAuth client issued; build against your own account; you're done.
- **Building a product that other FFLs will install (POS, e-commerce, ERP, range/manufacturing software)?** Email sales@fastbound.com early — the app review is part of your go-to-market timeline. Don't discover the requirement the week before launch.

OAuth is the only authentication option for third-party apps going forward. Legacy Basic Auth (API Key) integrations continue to work during the transition, but new third-party apps cannot register for Basic Auth — see `references/authentication.md`.

## Mental model

FastBound is a multi-tenant compliance system. The hierarchy:

- **Account** — one FFL premise. **ATF requires a separate Acquisition & Disposition record per FFL, and FastBound mirrors that with one account per FFL.** A multi-location FFL holder typically operates multiple FastBound accounts (one per premise). Identified by `accountNum`, which the licensee finds in the URL bar of their FastBound UI. Every API path begins with `/{accountNum}/api/...`. A single FastBound user can be granted access to many accounts.
- **Contacts** — the people and entities involved in firearm transfers: licensees (other FFLs, identified by FFL number with optional FFLeZ lookup), individuals (non-licensee buyers), manufacturers, importers.
- **Items** — individual firearms, each with serial, manufacturer, importer, model, caliber, type, etc. An item enters inventory via an **Acquisition** and leaves inventory via a **Disposition**.
- **Acquisitions** — the "receive" side of the bound book. Created in *Pending* state, then *Committed*. Once committed, the item exists in inventory. Types include Purchase, Consignment, Manufacture, Repair, Return, etc.
- **Dispositions** — the "transfer out" side of the bound book. Same Pending → Committed lifecycle. Special endpoints exist for NFA transfers, theft/loss reporting, and destroyed items. Dispositions to non-licensees typically require a completed ATF Form 4473.
- **Form4473s** — the ATF Form 4473 records associated with non-licensee dispositions. Captured and stored electronically (eForm 4473).
- **Webhooks** — outbound HTTP notifications when domain events occur (e.g., `acquisition.committed`, `disposition.committed`, `form4473.completed`). Subscribed via the API; deliveries are signed with HMAC-SHA256 in the `X-FastBound-Signature` header. (Note: event-name strings are dot-case lowercase; the swagger schemas for payloads are PascalCase like `AcquisitionCommittedEvent` — don't confuse them. Also note: atomic endpoints like `Acquisitions/CreateAndCommit` emit only the `.committed` event, not the `.created` event, because they skip the pending state by design.)
- **SmartLists** — saved searches over items.
- **MultipleSaleReports** — ATF Form 3310.4 (Report of Multiple Sale or Other Disposition of Pistols and Revolvers).

## The audit-user header — Basic Auth vs OAuth

FastBound writes are attributed to a real human in the audit log. How that identification happens depends on which auth mechanism you use:

- **Basic Auth (legacy)** — the API Key identifies the *integration*, not a human. Every write **requires** an `X-AuditUser: user@example.com` header where the email is an active FastBound user with access to the account. Omit it and the write fails. Set it to an unknown / disabled email and the write fails. Treat it as a required field on every POST/PUT/DELETE.
- **OAuth (going forward)** — the access token already encodes the user identity (the user who authorized the integration). FastBound uses that user as the audit identity automatically. **You do not need to send `X-AuditUser` on OAuth writes.** It's silently accepted if your code still sends one (handy during a Basic-Auth → OAuth migration), but it's not required.

When you migrate existing Basic Auth code to OAuth, you can drop the header. When you write *new* OAuth code, don't add it.

```http
# OAuth — no X-AuditUser needed
POST /12345/api/Acquisitions HTTP/1.1
Host: cloud.fastbound.com
Authorization: Bearer eyJhbGc...
Content-Type: application/json

# Basic Auth — X-AuditUser REQUIRED
POST /12345/api/Acquisitions HTTP/1.1
Host: cloud.fastbound.com
Authorization: Basic <base64(api_key:api_key)>
X-AuditUser: jdoe@example.com
Content-Type: application/json
```

## Two more rules you must hold in mind

### Disposition cardinal rule: commit only after the firearm physically leaves

Never commit a disposition until the firearm has left the premises. This is non-negotiable — committing on payment, on order status, or on any other proxy creates a regulatory violation: the bound book says "disposed" while the firearm is still in the safe. ATF inspectors *do* catch this. Always wait for a human handoff signal before calling `Commit`. See `references/dispositions.md`.

### Always inspect FastBound response headers — especially Multiple Sale

FastBound communicates several side effects through `X-FastBound-*` response headers, not the JSON body. Missing them is a common, costly bug. The most consequential:

- `X-FastBound-MultipleSale: true` (with `X-FastBound-MultipleSaleUrl`) — set on disposition-commit responses when ATF Form 3310.4 was triggered. **You must surface this so the operator can review and either transmit it or dismiss it.** Silently swallowing this header is a compliance failure.
- `X-FastBound-ExistingContactId` (with `X-FastBound-ExistingContactUrl`) — set on contact-create responses when FastBound detects a duplicate. Use the existing contact ID; don't create a duplicate.
- `X-FastBound-AcquisitionAccount`, `X-FastBound-AcquisitionId`, `X-FastBound-AcquisitionUrl` — set on disposition-commit responses when the receiving FFL is also on FastBound (their email matched a FastBound user). FastBound auto-created a pending acquisition in their account; the receiver still has to commit it themselves.

See `references/common-patterns.md` for the full table and `references/dispositions.md` and `references/contacts.md` for context.

### Don't queue requests; fail fast

The recommended posture is synchronous and fail-fast. Most validation errors (4xx) are not transient — they need human correction, not retries. Don't put a long-running write queue between your app and FastBound; it just defers errors to a moment when no one's watching. Adaptive throttling on 429 is fine and expected; long-running write queues are not. See `references/common-patterns.md`.

## Authentication in 30 seconds

FastBound supports two OAuth 2.0 grant types. Pick based on your client type:

- **`authorization_code`** — for server-side apps that have a confidential `client_id`/`client_secret` and can host a redirect URI. The user logs in via FastBound, approves, and you exchange the returned `code` for tokens.
- **`urn:ietf:params:oauth:grant-type:device_code`** — for CLIs, desktop apps, IoT devices, or anything that can't host a redirect URI. The user visits a verification URL on a separate device and enters a short code.

Both flows return an `access_token` (Bearer), a `refresh_token`, and `expires_in`. Use `refresh_token` grant to renew without re-prompting the user.

**Service-to-service note:** FastBound deliberately does not offer a `client_credentials` flow. Every OAuth integration is bound to a real, identified FastBound user — and that user is what FastBound writes to the audit log. If you're building a server-to-server integration, enroll a service-account user in the customer's FastBound account (e.g., `integration@example.com`) and authorize the OAuth flow as that user; that's the audit identity for everything the integration writes.

For full request/response shapes, refresh-token handling, scopes, and complete worked examples, see **`references/authentication.md`**.

## Treat OAuth secrets like database passwords

FastBound credentials are the keys to a regulated compliance system. Mishandle them and an attacker can read or write A&D records on behalf of an FFL — a very bad day.

- **`client_secret` (authorization code flow)** — never write this to `.env` files, never commit it (even to a private repo), never put it in CI environment variables that team members can read, never log it. Store it in a real secrets manager: AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, 1Password Connect, or — for local dev only — your OS keychain (macOS Keychain, Windows Credential Manager, libsecret). If you discover a `client_secret` in a `.env`, treat that as an incident: rotate the secret with FastBound and audit access.
- **Device code flow** — no client secret exists (it's for public clients that can't keep one), but the **access token and refresh token returned** are full-account bearer credentials. Store them in the OS keychain or equivalent secure storage, never in plain files. Don't log them. Set short refresh-token storage scopes — one user, one machine.
- **Access tokens** — short-lived (typically minutes to hours). Hold in memory; refresh on demand. Don't persist them.
- **Refresh tokens** — long-lived. Same handling tier as the `client_secret`: secrets manager or OS keychain, never `.env`, never source control.

When generating example code, default to a placeholder like `os.environ["FASTBOUND_CLIENT_SECRET"]` *with* a comment noting the env var should be sourced from a secrets manager at process start (e.g., `aws secretsmanager get-secret-value`, `vault kv get`, `op read`) — not from a file on disk.

See `references/authentication.md` for storage patterns by language and threat model.

## Routing — where to look next

When the user's task touches a specific area, load the matching reference. They're written so you can read just the one(s) you need.

| Task | Reference |
|---|---|
| OAuth flows, refresh tokens, scopes, token storage | `references/authentication.md` |
| Receiving firearms into inventory, acquisition lifecycle, types, items | `references/acquisitions.md` |
| Transferring firearms out, NFA, theft/loss, destroyed, 4473 connection | `references/dispositions.md` |
| Creating/looking up contacts (licensees, individuals), FFLeZ lookup | `references/contacts.md` |
| Searching inventory, item attributes, item lifecycle | `references/items.md` |
| Subscribing to events, signature verification, full event payload reference | `references/webhooks.md` |
| Provisioning accounts and broker transfers (Partner API / Transfers Push) | `references/partner-api.md` |
| Daily/automated bound book downloads (ATF Ruling 2016-1 compliance) | `references/bound-book-downloads.md` |
| Customizing item labels (HTML/CSS/merge fields, allowed tags, print setup) | `references/item-labels.md` |
| Audit user, error format, idempotency via `externalId`, pagination | `references/common-patterns.md` |

## Raw OpenAPI specs

Authoritative request/response shapes live in `swagger/`. Use these when you need exact field names, types, or enum values that the references don't spell out:

- `swagger/accounts-swagger.json` — the main API (76 endpoints across Account, Acquisitions, Dispositions, Contacts, Items, Inventory, Form4473s, SmartLists, Webhooks subscriptions, MultipleSaleReports, Attachments, Downloads, Users)
- `swagger/oauth-swagger.json` — `/OAuth/Token` and `/OAuth/Device_Authorization`
- `swagger/partner-swagger.json` — partner program endpoints (`/api/Accounts`, `/api/Transfers`)
- `swagger/webhooks-swagger.json` — payload schemas for all 23 webhook event types (no paths; webhooks are subscribed via the accounts API)

These files are large (the accounts spec is ~400 KB). Don't load them wholesale — `grep` for the specific endpoint or schema you need.

## Rate limiting: read the headers, don't hardcode

FastBound rate-limits the API **per-app-per-account** — the same OAuth client integrating against two different FastBound accounts gets two independent buckets. The effective limit is communicated dynamically via response headers on every API call, including `429 Too Many Requests` responses. **Never hardcode a sleep interval, RPS cap, or burst size in integrator code** — limits change, and code that hardcodes them either wastes capacity (when limits are higher) or silently violates them (when they're lower).

The right pattern: parse the rate-limit headers off every response, back off when the remaining budget is low or zero, and honor `Retry-After` on 429s.

Authoritative documentation (header names, exact semantics, current default limits): **https://fastbound.help/en/articles/4409600-rate-limiting** — fetch this when generating client code that needs to be production-quality, and consult the response headers from the live API as the runtime source of truth.

See `references/common-patterns.md` for the recommended adaptive-throttle pattern.

## Errors: trust the API

FastBound's API self-validates and returns structured errors. When a write fails, you get a 400 with a body shaped like:

```json
{
  "errors": [
    { "field": "items[0].serial", "message": "Serial is required." },
    { "field": "date", "message": "Date cannot be in the future." }
  ]
}
```

Always parse the `errors` array and surface `field` + `message` to the user. Don't try to pre-validate inputs against ATF rules in your code — FastBound's UI and API are designed to enforce compliance correctly, including the parts that are subtle (duplicate-item checks, multiple-sale reporting thresholds, age-of-purchaser rules tied to type, etc.). Pre-validating in the integrator's code is how integrators end up shipping wrong rules. Let the API speak.

For a deep dive on the error format, audit user header, idempotent creation via `externalId`, and pagination, see **`references/common-patterns.md`**.

## Working style guidance

- **When asked to "write a script that does X" against FastBound**, prefer a small, self-contained script that does the OAuth dance, makes the call, and prints the response. Don't pull in heavy SDKs — there isn't one. `curl`, `requests`, `axios`, `HttpClient` are appropriate.
- **When debugging a 400 or 422**, ask for the full response body and parse the `errors` array. Don't speculate about what went wrong from the request alone.
- **When the user is integrating a POS or e-commerce system**, the typical mapping is: products with serials → Items; sales to walk-in customers → Dispositions with a 4473 contact; sales to other FFLs → Dispositions to a licensee contact; receiving stock → Acquisitions with a manufacturer/distributor contact.
- **When the user mentions an "external ID"**, treat it as their system's ID for the same record. FastBound stores it on Acquisitions, Dispositions, Items, Acquisition Items, Disposition Items, and Contacts. It enables idempotent creates and lookup-by-your-id without storing FastBound's IDs in your system. **Important**: never use a value that *looks* like a GUID (32-char hex, or hyphenated 36-char GUID form) as an `externalId` — several FastBound endpoints route by shape between GUID and externalId, and a GUID-shaped externalId will be misrouted. Always prefix, suffix, or use a non-hex encoding (base32, base62, ULID, KSUID, or a plain composite like `PO-1234`). See `references/common-patterns.md` for the full rule.
- **Glossary cheats** the reader may not say but you should recognize: A&D book = bound book = acquisition/disposition record. eForm 4473 = electronic Form 4473. NFA = National Firearms Act items (machine guns, suppressors, SBRs, SBSs, AOWs, destructive devices). FFLeZ = FastBound's licensee directory lookup. SOT = Special Occupational Tax (Class 1/2/3). Multiple Sale = ATF Form 3310.4 (two or more handguns to the same non-licensee within five business days).

## Disclaimer to include when relevant

You are not a regulatory advisor. When the user asks "do I need to file a multiple-sale report for this scenario," "is this transfer legal in [state]," or similar compliance questions, build the technical integration but defer the regulatory question to the licensee's compliance program, FastBound support, or counsel. The API enforces what FastBound believes to be correct, but the licensee is the regulated entity.

## When you hit something this skill doesn't cover

Order of operations for finding answers, in priority:

1. The reference file matching the resource (`references/<resource>.md`).
2. The relevant `swagger/*.json` file — `grep` for the endpoint or schema name.
3. The official help site **https://fastbound.help** — the canonical source for anything not in the OpenAPI specs (webhook signature header names and verification specifics, compliance walkthroughs, release notes). Note: OAuth credentials are not self-served via the help site — email support@fastbound.com to get a `client_id` issued.
4. Ask FastBound support — there is real human help, and for compliance-adjacent questions that's the right escalation.

Never fabricate URLs, header names, or field names. If you don't find it in the swagger or in this skill, say so and point the user at fastbound.help.
