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

## Worked example — receive a shipment from a distributor (Python, OAuth)

```python
import os, requests

BASE = "https://cloud.fastbound.com"
ACCOUNT_NUM = "12345"
ACCESS_TOKEN = os.environ["FB_ACCESS_TOKEN"]  # from the OAuth flow
AUDIT_USER = "receiving@example.com"

def post(path, body):
    r = requests.post(
        f"{BASE}/{ACCOUNT_NUM}/api{path}",
        json=body,
        headers={
            "Authorization": f"Bearer {ACCESS_TOKEN}",
            "X-AuditUser": AUDIT_USER,
        },
        timeout=30,
    )
    if not r.ok:
        # Parse FastBound's structured errors and surface them.
        for e in r.json().get("errors", []):
            print(f"  {e['field']}: {e['message']}")
        r.raise_for_status()
    return r

# Suppose distributor_contact_id was created/looked up earlier.
distributor_contact_id = "..."

# OAuth: no X-AuditUser needed (token identifies the user).
# Basic Auth: include X-AuditUser on the request.
resp = post("/Acquisitions/CreateAndCommit", {
    "externalId": "PO-2026-04812",            # your system's purchase order ID
    "date": "2026-05-03",                     # business date of receipt
    "type": "Purchase",                       # see swagger for valid type strings
    "purchaseOrderNumber": "PO-2026-04812",
    "invoiceNumber": "INV-9921",
    "shipmentTrackingNumber": "1Z9...",
    "isManufacturingAcquisition": False,
    "contactId": distributor_contact_id,
    "items": [
        {
            "externalId": "SN:ABC123",        # your system's per-firearm ID
            "manufacturer": "Glock",
            "importer": "",                   # required when imported
            "countryOfManufacture": "Austria",
            "model": "19 Gen5",
            "serial": "ABC123",
            "caliber": "9mm",
            "type": "Pistol",                 # see swagger for valid type strings
            "barrelLength": 4.02,
            "totalLength": 7.36,
            "condition": "New",
            "cost": "475.00",
            "price": "599.99",
            "mpn": "PA195S203",
            "upc": "764503047022",
            "sku": "GLK-19-G5",
        },
        # ... more items
    ],
})

# 201 Created — the body is the full acquisition with FastBound IDs.
acq = resp.json()
print("Acquisition ID:", acq["id"])
for item in acq["items"]:
    print(f"  Item {item['serial']} → FastBound ID {item['id']}")
```

A few things this example does deliberately:

- **`externalId` everywhere**: PO number on the acquisition, serial-based ID on each item. Now you can look up either by your own ID (`GET /Acquisitions/GetByExternalId/{externalId}`) without storing FastBound's IDs in your DB. That said, the integrator guides explicitly recommend storing FastBound's GUIDs for items and contacts because lookup-by-external-ID adds an API call and complexity. Pick one approach and be consistent. **Critical rule**: never use a 32-character hex string (or hyphenated GUID form) as an `externalId` — FastBound's routing treats GUID-shaped values as GUID lookups, not external-ID lookups. Use a prefix (`PO-...`, `SN:...`), suffix, or non-hex encoding. See `common-patterns.md` for details.
- **OAuth: no `X-AuditUser`** — the token's user is the audit identity. Under Basic Auth (legacy), include `X-AuditUser`.
- **No retry on validation failure**: `r.raise_for_status()` propagates the error so a human can fix the data. Don't loop and retry — see "Don't queue requests" in `common-patterns.md`.

## Acquisition fields

Top-level (on `POST /Acquisitions` or `CreateAndCommit`):

| Field | Type | Notes |
|---|---|---|
| `externalId` | string | Your system's ID. Unique per account per resource type. |
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
| `externalId` | Your per-firearm ID. |
| `manufacturer`, `model`, `serial`, `type`, `caliber` | Core identification. |
| `importer`, `countryOfManufacture` | Required when imported / when the location's settings demand it. |
| `barrelLength`, `totalLength` | Numeric (inches). |
| `itemNumber` | Ignored unless the account has manual item-number entry enabled. |
| `condition` | Free-text: New, Used, etc. |
| `cost`, `price` | Strings (currency). |
| `mpn`, `upc`, `sku` | Inventory identifiers. `sku` is a string, **max 50 characters**. |
| `location` | Multi-location FFLs only. |
| `lightspeedItemID`, `lightspeedSerialID`, `lightspeedSaleID` | Lightspeed POS integration IDs (FastBound ↔ Lightspeed). |

> **Match the firearm, not the marketing copy.** A&D records must reflect what is *physically marked* on the firearm — manufacturer, importer, country of manufacture, model, caliber, serial — not what's on the box, the website, or the spec sheet. Garbage in, regulatory finding out.

## Editing and deleting

- `PUT /{accountNum}/api/Acquisitions/{id}` — edit a *pending* acquisition. **PUT replaces the entire object: any field you omit becomes null.** Always GET first, modify, PUT.
- `DELETE /{accountNum}/api/Acquisitions/{id}` — delete a *pending* acquisition. Committed acquisitions can't be deleted casually.
- For items: `PUT /{accountNum}/api/Acquisitions/{id}/Items/{acquisitionItemId}`, `DELETE /{accountNum}/api/Acquisitions/{id}/Items/{acquisitionItemId}`.

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
- `acquisition.transfer` — used in inter-FFL transfer scenarios.

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
- **Mismatched serial → item ID after batch creation** — when you POST `/Items/Multiple`, FastBound may not return items in your input order. Match by serial or external ID, not by index.
