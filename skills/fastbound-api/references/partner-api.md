# Partner API

The Partner API is a small, separate API surface aimed at integrators that operate outside the standard FastBound account model. It has two endpoints, each addressing a distinct integrator scenario:

- **`POST /api/Accounts`** — provision a new FastBound account (and its owner user) on behalf of an FFL customer. For partners that resell or bundle FastBound (e.g., a POS vendor that creates a FastBound account as part of customer onboarding).
- **`POST /api/Transfers`** — the **Transfers Push API**. For external systems (POS, e-commerce, distributors, dealer software) that need to push a transfer to a destination FFL using FastBound, without operating the destination account themselves.

These endpoints are not part of the per-account `/{accountNum}/api/...` surface. They have their own access model, their own request/response conventions, and their own onboarding.

Reference articles and code:
- **Transfers Push API article**: https://fastbound.help/en/articles/10732676-transfers-push-api
- **Validated sample code (multiple languages)**: https://github.com/FastBound/Support/tree/main/samples/transfers — start here when implementing the Transfers Push API. These are FastBound-maintained reference implementations covering common languages and frameworks; treat them as the authoritative shape for request bodies, retry handling, and idempotency-key patterns. If your language has a sample, port from it rather than building from scratch.
- **JSON Schema**: https://schemas.fastbound.org/transfers-push-v1.json — for IDE intellisense and request validation in your CI.
- **OpenAPI for the partner endpoints**: https://cloud.fastbound.com/swagger/index.html?urls.primaryName=v1+Partner#/PartnersTransfersV1
- Partner program enrollment / Account provisioning: https://fastbound.help

## Transfers Push API — the most common reason to be here

> **Read this carefully.** Most third-party developers who think they need the Partner API actually need the standard accounts API. Here's the rule:
>
> **If both the transferor and transferee are FastBound customers, you don't need the Transfers Push API.** Transfers between FastBound customers are handled automatically by the platform — see the auto-pending-acquisition flow in `dispositions.md` (the `X-FastBound-Acquisition*` response headers on a disposition commit).
>
> **The Transfers Push API exists for the case where your system holds inventory data but is *not* itself a FastBound customer recording it as A&D.** A distributor's order management system pushing transfers to dealer FFLs (some of whom use FastBound, some of whom may not) is the canonical example.

### Eligibility and access

- "Available at no cost to integrators building integrations that benefit FastBound customers."
- Intended for software developers building POS / e-commerce / distributor / dealer software.
- **To get credentials, email support@fastbound.com.** (For OAuth app review and approval — accessing accounts owned by other parties — that's a separate conversation with sales@fastbound.com. Different conversations, different teams.)

### Request

```
POST https://cloud.fastbound.com/api/Transfers
Content-Type: application/json
```

```json
{
  "$schema": "https://schemas.fastbound.org/transfers-push-v1.json",
  "idempotency_key": "po-2026-04812:rev1",
  "transferor": "1-99-XXX-XX-XX-99999",
  "transferee": "1-99-YYY-YY-YY-88888",
  "transferee_emails": ["receiving@example.org"],
  "tracking_number": "1Z9...",
  "po_number": "PO-2026-04812",
  "invoice_number": "INV-9921",
  "acquire_type": "Purchase",
  "note": "Standard wholesale order",
  "items": [
    {
      "manufacturer": "Glock",
      "importer": "",
      "country": "Austria",
      "model": "19 Gen5",
      "caliber": "9mm",
      "type": "Pistol",
      "serial": "ABC123",
      "cost": "475.00",
      "price": "599.99",
      "mpn": "PA195S203",
      "sku": "GLK-19-G5",
      "upc": "764503047022",
      "barrelLength": 4.02,
      "overallLength": 7.36,
      "condition": "New",
      "note": ""
    }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `$schema` | yes | Always `https://schemas.fastbound.org/transfers-push-v1.json`. |
| `idempotency_key` | **yes** | Up to 255 case-insensitive characters. Use a UUID or a hash derived from your transaction data — same key on retry returns the original result instead of creating a duplicate. |
| `transferor` | yes | Sender's FFL number. With or without hyphens. |
| `transferee` | yes | Receiver's FFL number. With or without hyphens. |
| `transferee_emails` | yes | At least one email address. **Matching uses both email AND FFL number** — at least one of the listed emails must match a FastBound user on the destination account. |
| `tracking_number`, `po_number`, `invoice_number` | optional | Searchable identifiers carried into the destination acquisition. |
| `acquire_type` | optional | The acquisition type to record on the destination side. |
| `note` | optional | Free-text. |
| `items` | yes | At minimum each item needs `manufacturer`, `model`, `caliber`, `type`, `serial`. Use the snake_case schema below — note the differences from the camelCase accounts API. |

### Response

```json
{
  "accountNumber": 12345,
  "acquisitionId": "guid",
  "acquisitionUrl": "https://cloud.fastbound.com/..."
}
```

The destination FFL receives a **pending** acquisition. They must review and commit it themselves via `POST /Acquisitions/{id}/Commit` — the push API doesn't commit on their behalf; commitment is the receiver's regulatory act.

### Match-failure case

> "If none of the email addresses included in the request match a FastBound account, the response will indicate that no matching accounts could be found."

Practical recommendation from FastBound: when you onboard a customer (a dealer FFL who will receive transfers from you), add a **designated FastBound email field** in your UI that they can update to opt in (or out) of electronic transfers. Pull that email into `transferee_emails` for the push.

### Field-naming inconsistencies, called out

The Partner API uses **snake_case** (`idempotency_key`, `tracking_number`, `po_number`, `invoice_number`, `acquire_type`) and the per-item schema uses **`country`** (not `countryOfManufacture`) and **`overallLength`** (not `totalLength`) — different from the accounts API's camelCase + `countryOfManufacture` + `totalLength`. Don't share serializers between the two surfaces; treat them as distinct schemas.

The OpenAPI surface for the partner endpoints lives at:
**https://cloud.fastbound.com/swagger/index.html?urls.primaryName=v1+Partner#/PartnersTransfersV1**

### Errors

Partner endpoint errors return RFC 7807 `application/problem+json` documents — different shape from the accounts API's `{ errors: [{ field, message }] }` format. Parse `type`, `title`, `status`, `detail`, `instance`.

### Idempotency is required

Always populate `idempotency_key`. A reasonable pattern:

```
{your_system_id}:{po_or_invoice}:{rev_or_attempt}
```

Same key on retry returns the original successful result (or processes the retry safely).

### Implementation: start from the FastBound samples

For the Transfers Push API specifically, FastBound maintains validated sample code at **https://github.com/FastBound/Support/tree/main/samples/transfers** — multiple languages, real working request bodies, the right error handling and idempotency patterns. **Default to porting from these samples rather than writing your client from scratch.** They reflect the current schema, current auth, and the patterns FastBound has seen work in production. If your language isn't covered, pick the closest sample as your reference shape.

### Decision tree: do I want the Transfers Push API?

```
Are you the transferor's FastBound integrator (POS/ERP for the sender FFL)?
├─ Yes → Use the standard accounts API: POST /Dispositions/CreateAndCommit (with the
│         receiving FFL as the contact). If the receiver is also on FastBound, the
│         X-FastBound-Acquisition* headers will surface the auto-pending acquisition
│         in their account. See dispositions.md.
└─ No → Are you a distributor/wholesaler/order management system pushing
        transfers to FFL customers WITHOUT being their FastBound integrator?
        ├─ Yes → Use the Transfers Push API. Email support@fastbound.com.
        └─ No → You probably want the standard accounts API.
```

## Account provisioning — `POST /api/Accounts`

For partners who resell or bundle FastBound — e.g., a POS vendor that creates a FastBound account as part of customer onboarding — this endpoint creates an account and its owner user atomically.

```json
POST /api/Accounts
Content-Type: application/json

{
  "verifiedEmailAddress": "owner@example.com",
  "fflNumber": "1-99-XXX-XX-XX-99999",
  "accountNickname": "Springfield Main Store",
  "planId": "...",
  "timeZoneId": "America/Chicago"
}
```

| Field | Required | Notes |
|---|---|---|
| `verifiedEmailAddress` | yes | Email of the account owner user. **Must already be verified on the partner side**; FastBound trusts the partner that the email is real. |
| `fflNumber` | yes | The FFL number of the account to create. |
| `accountNickname` | no | Defaults to the FFL's trade name (or license name) if null. |
| `planId` | yes | Billing plan ID — partners receive valid plan IDs from FastBound. |
| `timeZoneId` | yes | IANA time zone (e.g., `America/Chicago`, `America/Los_Angeles`). Used for business-day calculations and disposition timing. |

Response:

```json
{
  "id": "guid",
  "number": 12345,
  "apiKey": "..."
}
```

> **The `apiKey` field returned here is the legacy Basic Auth API Key for the new account.** That mechanism is being phased out in favor of OAuth (see `authentication.md`), but the provisioning response is currently structured around it. Treat `apiKey` as a long-lived bearer credential and store it accordingly (secrets manager, never `.env`, never source control). Expect this surface to evolve as the OAuth-only direction firms up; check the help site for current state.

Errors return `application/problem+json` (RFC 7807) — different shape from the accounts API.

## When *not* to use the Partner API

If you're doing any of these, you want the **accounts API**, not the Partner API:

- Building a POS or e-commerce app for FFLs (each customer has their own account; users authenticate via OAuth).
- Building a single-FFL custom integration (your own FFL or a single customer).
- Recording acquisitions and dispositions within an account you own or have OAuth access to.
- Pushing transfers between two FFLs both of whom are on FastBound (the platform handles it automatically — see `dispositions.md`).

## Common pitfalls

- **Reaching for the Transfers Push API when both sides are on FastBound.** Use the standard accounts API and let the platform handle the inter-FFL transfer automatically.
- **Treating Partner endpoints like the accounts API** — different schema (snake_case vs camelCase, `country`/`overallLength`), different error shape (problem+json), different access control (partner program).
- **Skipping `idempotency_key`** — required, and your retries will create duplicate transfers if you skip or randomize it across retries.
- **Wrong `transferee_emails`** — none of the emails matches a user on the destination account → transfer can't resolve the destination → request fails. Build a "designated FastBound email" field in your UI so customers can keep this current.
- **Provisioning an account with an unverified email** — the partner asserts the email has been verified on its side; passing through unverified addresses creates orphaned, unrecoverable accounts.
- **Treating the returned `apiKey` as user-displayable** — it's a credential. Surface it through a secure provisioning flow (one-time password reset link, secure portal), not via email or a CSV export.
- **Asking sales@fastbound.com for Transfers Push credentials** — that's a support@ conversation. Sales handles the OAuth app review for accessing third-party accounts; support handles Partner API onboarding. Email the right inbox.
