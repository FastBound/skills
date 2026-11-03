# Acquisitions

An **acquisition** is the "receive" side of the bound book — every firearm that enters an FFL's inventory does so through an acquisition record. Common cases: receiving stock from a distributor or manufacturer, receiving consignment items, receiving repair returns, accepting a trade-in, recording a manufactured firearm (for licensed manufacturers).

Reference article: https://fastbound.help/en/articles/3963654-acquiring-items

## Lifecycle: Pending → Committed

Every acquisition starts as **pending**. While pending, the items it references are *not yet in inventory* and don't appear on the official A&D record. **Committing** the acquisition writes it to the bound book and brings the items into inventory.

```
[POST /Acquisitions]      → pending
[POST /Acquisitions/{id}/Commit]  → committed (in inventory)
```

Why two states? Real-world receiving often has multiple steps (count the box, verify serials, attach the invoice). Pending state lets integrators stage data and recover from partial failures without polluting the A&D record. **Only commit when you're confident the data is right.** Once committed, edits and deletions are tightly constrained for compliance reasons.

**Time-to-record is regulated.** Per ATF, firearms must be acquired (recorded) within 1, 7, or 15 business days of receipt depending on the FFL type. This is the licensee's compliance obligation; your integration should help them meet it, not delay it.

## The standard four-step workflow

For a fresh acquisition with a new contact and items:

1. **Create the contact** (or look up existing). See `contacts.md`.
2. **Create the pending acquisition** referencing the contact:
   - `POST /{accountNum}/api/Acquisitions`
3. **Add items** to the acquisition:
   - Single: `POST /{accountNum}/api/Acquisitions/{id}/Items`
   - Multiple in one call: `POST /{accountNum}/api/Acquisitions/{id}/Items/Multiple`
4. **Commit** the acquisition:
   - `POST /{accountNum}/api/Acquisitions/{id}/Commit`

If the contact attachment is separate (e.g., the operator decides which supplier later):

- `PUT /{accountNum}/api/Acquisitions/{id}/AttachContact/{contactId}`

## Recommended: `CreateAndCommit` (atomic, one round trip)

For a fresh acquisition where you have everything in hand — contact, items, and the intent to commit immediately — use the consolidated endpoint:

- `POST /{accountNum}/api/Acquisitions/CreateAndCommit`

This combines all four steps in one request: it creates the acquisition, attaches the contact (or accepts a nested contact object to create on the fly), adds the items, and commits — atomically. If anything fails validation, nothing is persisted. This is the right default for POS, e-commerce receiving, and most ERP-driven workflows.

Use the multi-step flow when:

- The receiving process is genuinely staged (items arrive over time, the operator must inspect each one).
- You're recording an existing pending acquisition that already has some items and a contact.
- You need to attach an item by `externalId` after it's already physically inspected.

## Worked example — receive a shipment from a distributor

One request, on the wire. `contactId` is the distributor, created or looked up beforehand.

```http
POST /12345/api/Acquisitions/CreateAndCommit HTTP/1.1
Host: cloud.fastbound.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "externalId": "PO-2026-04812",
  "date": "2026-05-03",
  "type": "Purchase",
  "purchaseOrderNumber": "PO-2026-04812",
  "invoiceNumber": "INV-9921",
  "shipmentTrackingNumber": "1Z9...",
  "isManufacturingAcquisition": false,
  "contactId": "8f14e45f-ceea-467a-9c1a-1b2c3d4e5f60",
  "items": [
    {
      "externalId": "SN:ABC123",
      "manufacturer": "Glock",
      "importer": "",
      "countryOfManufacture": "Austria",
      "model": "19 Gen5",
      "serial": "ABC123",
      "caliber": "9mm",
      "type": "Pistol",
      "barrelLength": 4.02,
      "totalLength": 7.36,
      "condition": "New",
      "cost": "475.00",
      "price": "599.99",
      "mpn": "PA195S203",
      "upc": "764503047022",
      "sku": "GLK-19-G5"
    }
  ]
}
```

Field notes: `date` is the business date of receipt; `externalId` on the acquisition is your purchase-order ID and on each item your per-firearm ID; `importer` is required when the firearm is imported; `type` (both on the acquisition and on each item) takes specific strings — check the swagger for the valid set.

**No `X-AuditUser` header** — this is OAuth, so the access token carries the audit identity. Under legacy Basic Auth you would add `X-AuditUser: receiving@example.com`, and the rest of the request is byte-for-byte the same.

Success is **200 OK** (not 201), and the body pairs every ID FastBound assigned with the `externalId` you sent — which is what makes the write reconcilable against your own records:

```json
{
  "acquisitionId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "acquisitionExternalId": "PO-2026-04812",
  "contactId": "8f14e45f-ceea-467a-9c1a-1b2c3d4e5f60",
  "contactExternalId": "DIST-ACME",
  "items": [
    {
      "id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
      "externalId": "SN:ABC123",
      "itemNumber": "1042",
      "serial": "ABC123",
      "manufacturer": "Glock",
      "model": "19 Gen5"
    }
  ]
}
```

**Mind the naming asymmetry**: the acquisition's own identifiers are `acquisitionId` / `acquisitionExternalId`, while each item uses plain `id` / `externalId`. A parser that expects `id` at the top level finds nothing there.

The endpoint also declares a **204 No Content**. Don't write code that assumes a body is always present on success — check for one before parsing.

A few things this example does deliberately:

- **Captures FastBound's IDs from the response**: the 200 body carries `acquisitionId`, `contactId`, and each item's `id`, each alongside the `externalId` you supplied. **Store them.** That's the recommended identifier strategy — persisting FastBound's Acquisition ID, Item ID, Contact ID, and Disposition ID in your own database makes the integration more reliable and dependable, easier to troubleshoot, and easier to backfill or reconcile. See `common-patterns.md`.
- **`externalId` as a secondary index, not the primary key**: the PO number on the acquisition and a serial-based ID on each item let you make creates idempotent and recover if your local DB is rebuilt. Don't lean on it as the *only* link between the two systems unless you genuinely can't extend your schema to hold FastBound's IDs. **Critical rule when you do use it**: never use a 32-character hex string (or hyphenated GUID form) as an `externalId` — FastBound's routing treats GUID-shaped values as GUID lookups, not external-ID lookups. Use a prefix (`PO-...`, `SN:...`), suffix, or non-hex encoding. See `common-patterns.md` for details.
- **OAuth: no `X-AuditUser`** — the token's user is the audit identity. Under Basic Auth (legacy), include `X-AuditUser`.
- **No retry on validation failure**: a 400 here means the data is wrong, not that the network hiccupped. Parse the `errors` array, surface `field` and `message` to whoever can fix the record, and stop. Don't loop and retry — see "Don't queue requests" in `common-patterns.md`.

## Acquisition fields

Top-level (on `POST /Acquisitions` or `CreateAndCommit`):

| Field | Type | Notes |
|---|---|---|
| `externalId` | string | Your system's ID. Unique per account per resource type. Secondary index — store FastBound's Acquisition ID as the primary link. See `common-patterns.md`. |
| `isManufacturingAcquisition` | boolean | `true` for licensed manufacturers recording self-manufactured firearms. Default false. |
| `date` | date | Business date of acquisition. **Cannot be in the future.** |
| `type` | string | Acquisition type — see swagger for valid values (Purchase, Consignment, etc.). |
| `note` | string | Free-text. |
| `purchaseOrderNumber` | string | Optional, searchable. |
| `invoiceNumber` | string | Optional, searchable. |
| `shipmentTrackingNumber` | string | Optional, searchable. |
| `contactId` (or `contactExternalId`, or nested `contact` object) | varies | The party you're acquiring from. |
| `items` | array | When using `CreateAndCommit` or `Items/Multiple`. |

Per item (see `CreateAcquisitionItem.Command` in the swagger for the full list):

| Field | Notes |
|---|---|
| `externalId` | Your per-firearm ID. Prefer storing FastBound's Item ID from the response; see `common-patterns.md`. |
| `manufacturer`, `model`, `serial`, `type`, `caliber` | Core identification. |
| `importer`, `countryOfManufacture` | Required when imported / when the location's settings demand it. |
| `barrelLength`, `totalLength` | Numeric (inches). |
| `itemNumber` | Ignored unless the account has manual item-number entry enabled. |
| `condition` | Free-text: New, Used, etc. |
| `cost`, `price` | Strings (currency). |
| `mpn`, `upc`, `sku` | Inventory identifiers. `sku` is a string, **max 50 characters**. |
| `location` | Multi-location FFLs only. |
| `lightspeedItemID`, `lightspeedSerialID`, `lightspeedSaleID` | **Do not use.** Reserved for FastBound's Lightspeed R-Series integration; may be removed. Store your own identifiers on your side — see `common-patterns.md`. |

> **Never supply a marking value you were not given.** If manufacturer, importer, country of
> manufacture, model, caliber, type or serial is missing from what you were handed, leave it
> unset and say what is missing — do not infer it, and do not fill it from what a firearm of
> that make usually carries. `importer` is where this goes wrong most: an imported firearm's
> importer marking is a fact about the individual firearm, and guessing it from the
> manufacturer's name writes a plausible, wrong entry into a federal record that nobody will
> notice. A blank field is a question someone answers; an invented one is a finding.
>
> **Match the firearm, not the marketing copy.** A&D records must reflect what is *physically marked* on the firearm — manufacturer, importer, country of manufacture, model, caliber, serial — not what's on the box, the website, or the spec sheet. Garbage in, regulatory finding out.

## Editing and deleting

- `PUT /{accountNum}/api/Acquisitions/{id}` — edit a *pending* acquisition. **PUT replaces the entire object: any field you omit becomes null.** Always GET first, modify, PUT.
- `DELETE /{accountNum}/api/Acquisitions/{id}` — delete a *pending* acquisition. There is no committed acquisition to delete: committing consumes the pending record, and a later call against that id returns 404. The items live on in inventory. See "Committing consumes the pending record" in `common-patterns.md` before writing any retry logic around Commit.
- For items: `PUT /{accountNum}/api/Acquisitions/{id}/Items/{acquisitionItemId}`, `DELETE /{accountNum}/api/Acquisitions/{id}/Items/{acquisitionItemId}`.

## Acquisitions you didn't create (inbound transfers)

Not every acquisition in an account comes from your integration. When a supplier pushes a transfer through the Partner API, or another FastBound FFL commits a disposition naming the account as the transferee, FastBound writes a **pending** acquisition straight into the receiving account — transferor contact and items already populated, no call from you. Distributor-to-dealer is the common shape, and the sender can be any FFL in FastBound's network — the largest in the industry — as well as connected distributors. Which distributors are live changes as more are brought online, so treat any list as current-as-of-today and ask **support@fastbound.com** rather than hardcoding one. See `partner-api.md` for the push side and the `X-FastBound-Acquisition*` headers in `common-patterns.md` for the FastBound-to-FastBound side.

Three things follow, and they surprise integrators in this order.

**Nothing commits it automatically.** The acquisition sits pending until a person reviews it and commits — in the UI, or `POST /Acquisitions/{id}/Commit`. FastBound never auto-commits a transfer, and no transfer path commits on the receiver's behalf: committing is the receiving licensee's regulatory act, and the review is where a missing or mistyped serial gets caught *before* it's in the bound book. Don't build a rule that commits inbound transfers on arrival.

**Your system finds out by webhook, not by polling.** Subscribe to `acquisition.transfer` — receive side only, and always paired with `acquisition.created` for the same record, which is sent first. Two events, one acquisition: dedupe on the acquisition ID or you'll ingest it twice. Don't wait on `acquisition.committed` for this; it doesn't fire until someone reviews and commits, and until then your system is blind to firearms already on the dock. Full event semantics in `webhooks.md`.

**Don't delete it and write your own.** There's a pattern where the transfer-created acquisition gets treated as a duplicate of what's already in the other system: check it against your own receiving records, delete it from FastBound, then re-create the acquisition from your side. **FastBound does not favor this approach.** It isn't a duplicate — it's the record, already correct and already carrying the transferor's data — so deleting and rewriting buys nothing and adds a failure mode between the two writes. It also throws away anything that arrived with the transfer: **attachments on transfers are coming** (partners have asked to send documents along with the firearms), a delete takes them with it, and FastBound may block deletion of acquisitions that have attachments once that ships. An integration built on delete-and-rewrite is built on something expected to stop working.

Reconcile by matching instead. Take FastBound's acquisition ID, contact ID, and item IDs from the `acquisition.transfer` payload, upsert them against your own purchase order or receiving record, and let the licensee commit the one acquisition that was already there. That's the identifier strategy in `common-patterns.md` doing exactly what it's for.

## Search and lookup

- `GET /{accountNum}/api/Acquisitions` — list, with `take`/`skip`, filters: `id`, `externalId`, `type`, `purchaseOrderNumber`, `invoiceNumber`, `shipmentTrackingNumber`.
- `GET /{accountNum}/api/Acquisitions/{id}` — single by FastBound ID.
- `GET /{accountNum}/api/Acquisitions/GetByExternalId/{acquisitionExternalId}` — by your ID.
- Items by ID: `GET /{accountNum}/api/Acquisitions/{id}/Items/{acquisitionItemId}` and external-ID variants.

## Manufacturing acquisitions (Type 07/10 FFLs)

Set `isManufacturingAcquisition: true` to record a firearm you've manufactured (assembled from components, not received from another FFL). The flag is what distinguishes a manufacturer's bound book entry from a dealer's.

For the more involved scenario — assembling a firearm from a serialized lower receiver that's already in inventory — you don't create a manufacturing acquisition directly. You create a **manufacturing disposition** that consumes the lower and FastBound auto-creates a pending acquisition for the completed firearm. See `dispositions.md`.

Reference article: https://fastbound.help/en/articles/11421255-fastbound-integration-guide-for-firearms-manufacturers

## Webhook events for acquisitions

When subscribing to webhooks (`webhooks.md`), the relevant event names are:

- `acquisition.created` — pending acquisition created.
- `acquisition.committed` — committed to the bound book; this is the "in inventory" signal.
- `acquisition.deleted` — pending acquisition deleted.
- `acquisition.transfer` — an acquisition arrived by transfer from another FFL. Receive side only, always paired with `acquisition.created`. This is the signal for stock you didn't record yourself — see "Acquisitions you didn't create (inbound transfers)" above.

### How events map to `CreateAndCommit` (easy to miss — by design)

This is an intentional implementation detail worth understanding up front: **`Acquisitions/CreateAndCommit` emits `acquisition.committed` only — not `acquisition.created`.** That isn't a bug; it's the natural consequence of the atomic endpoint skipping the pending state entirely. There's no pending-state event to fire because there's no pending state. The same logic applies to items added via `CreateAndCommit`: no separate `item.imported` event — those items appear inside the parent `acquisition.committed` payload.

So when picking which events to subscribe to, match them to your write path:

| Your write path | Events emitted |
|---|---|
| `POST /Acquisitions` (pending) → later `POST /Acquisitions/{id}/Commit` | `acquisition.created` on the POST, then `acquisition.committed` on the commit |
| `POST /Acquisitions/CreateAndCommit` (atomic) | `acquisition.committed` only |

**If you want to be notified of every new acquisition regardless of how it was recorded**, subscribe to `acquisition.committed` (it fires in both flows). Subscribe to `acquisition.created` only if you specifically care about pending-state notifications *and* you control the write path to use the multi-step flow. This isn't unique to acquisitions — `disposition.committed` works the same way relative to its (`Dispositions/CreateAndCommit`) atomic counterpart.

Most "I'm not getting webhooks for new acquisitions" reports trace back to subscribing to `acquisition.created` while the integrator's writes go through `CreateAndCommit`. The fix is one event-name change in the subscription.

## Common pitfalls

- **Forgetting `X-AuditUser`** — every write fails without it.
- **`date` in the future** — rejected as a validation error.
- **Committing before items are physically received** — same regulatory risk as committing dispositions before the firearm leaves: an inspector finds an item marked acquired that isn't on the premises.
- **Using PUT to update one field** — PUT replaces the object; omitted fields become null. Always GET-modify-PUT.
- **Marketing copy in `manufacturer` / `model`** — must match the physical markings.
- **Deleting a transfer-created acquisition to clear a "duplicate"** — it isn't a duplicate, it's the record, and the delete discards data that came with the transfer. Match it into your system on FastBound's IDs and let the licensee commit it.
- **Auto-committing an inbound transfer on arrival** — the review step is the point. Commit is the receiving licensee's act, not your integration's.
- **Mismatched serial → item ID after batch creation** — when you POST `/Items/Multiple`, FastBound may not return items in your input order. Match by serial or external ID, not by index.
