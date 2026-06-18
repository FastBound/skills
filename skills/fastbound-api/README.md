# `fastbound-api`

An [Agent Skill](https://www.anthropic.com/news/skills) that teaches AI coding assistants how to build, debug, and reason about integrations with the [FastBound API](https://cloud.fastbound.com/swagger) — the REST API behind FastBound's electronic A&D (acquisition & disposition) bound book and ATF Form 4473 software for Federal Firearms Licensees (FFLs).

If you're integrating a POS, e-commerce platform, ERP, range management system, or manufacturing system with FastBound, this skill gives your AI assistant the API's mental model, the regulatory rules that shape it, the auth options, the response headers that carry side effects, and where to look in the bundled OpenAPI specs when it needs an exact field name.

> **Install:** see the [repository README](../../README.md#install). This skill ships in the `fastbound` plugin.

## What's inside

```
SKILL.md              Top-level instructions the agent loads on every invocation
references/           Topic-specific deep dives, loaded on demand
  acquisitions.md       Receiving firearms into inventory; pending → committed lifecycle
  authentication.md     OAuth (auth code + device code), legacy Basic Auth, SSO vs API auth
  bound-book-downloads.md  ATF Ruling 2016-1 daily bound book downloads
  common-patterns.md    Audit user header, error format, externalId, pagination, rate limits
  contacts.md           Licensees, individuals, FFLeZ lookup, duplicate detection
  dispositions.md       Transfers out, NFA, theft/loss, destroyed, 4473 linkage
  item-labels.md        Custom HTML/CSS labels, merge fields, print setup
  items.md              Inventory search, item attributes and lifecycle
  partner-api.md        Account provisioning and Transfers Push (partner program)
  webhooks.md           Subscriptions, HMAC-SHA256 signature verification, event payloads
swagger/              Authoritative OpenAPI specs — the agent greps these for exact shapes
  accounts-swagger.json    Main API (~400 KB; 76 endpoints)
  oauth-swagger.json       /OAuth/Token and /OAuth/Device_Authorization
  partner-swagger.json     Partner program endpoints
  webhooks-swagger.json    Payload schemas for all 23 webhook event types
```

## Authoritative sources

When something isn't covered by `SKILL.md`, a reference file, or the bundled OpenAPI specs, the skill defers to the official FastBound help site at **https://fastbound.help** rather than guessing.

## Getting credentials

OAuth credentials are not self-served. Email **support@fastbound.com** to have a `client_id` issued for your integration. Third-party apps that need to access accounts owned by other FFLs (the typical POS / e-commerce / SaaS scenario) must be reviewed and approved — start that conversation with **sales@fastbound.com** early.

## Scope

This skill teaches the API; it does not replace regulatory advice. It defers compliance questions ("do I need to file a multiple-sale report for this scenario," "is this transfer legal in [state]") to the licensee's compliance program, FastBound support, or counsel.
