# Items

An **item** is a specific firearm that has entered the FFL's inventory via an acquisition. Items have a lifecycle (in inventory → disposed → optionally undisposed → optionally deleted), a rich set of identification attributes, and a permanent audit trail.

Reference article: https://fastbound.help/en/articles/3963659-items

## What an item is (and isn't)

An item is a *single physical firearm*, identified by manufacturer + model + serial. It is **not** a SKU or product definition — those are upstream catalog concepts in your POS or ERP. Two firearms of the same model with different serials are two items in FastBound.

This distinction matters for manufacturers and high-volume dealers: keep your **product catalog** (SKU → manufacturer/model/UPC) separate from your **serialized inventory records** (the actual items in FastBound).

## Lifecycle

```
[Acquisition committed]   → in inventory
[Disposition committed]   → disposed (out of inventory)
[PUT /Items/{id}/Undispose] → back in inventory (corrects an erroneous disposition)
[DELETE /Items/{id}]      → deleted (very narrow allowance — see below)
```

### Deletion is narrow

Items are part of the regulatory bound book. They can be deleted only for two narrow reasons:

- **`Duplicate`** — the same firearm was acquired twice by mistake.
- **`Error`** — the firearm was acquired in error (e.g., wrong serial entered).

Both require a permanent, unmodifiable note explaining the deletion. There is no "I want to clean up the data" pathway — wrong data is corrected (PUT) or undisposed/redisposed, not deleted.

Don't surface a generic "Delete Item" button in your UI without the operator explicitly choosing one of those two reasons.

## API operations

| Operation | Endpoint |
|---|---|
| List/search | `GET /{accountNum}/api/Items` (with rich filters) |
| Get single | `GET /{accountNum}/api/Items/{id}` |
| Update | `PUT /{accountNum}/api/Items/{id}` (full replacement) |
| Delete | `DELETE /{accountNum}/api/Items/{id}` (with reason and note) |
| Undispose | `PUT /{accountNum}/api/Items/{id}/Undispose` |
| Set external ID only | `PUT /{accountNum}/api/Items/{id}/SetExternalId` |

Use `SetExternalId` rather than a full PUT when you only need to update the external ID. It's a targeted endpoint that avoids the GET-modify-PUT dance and the risk of nulling other fields.

## Listing and search

`GET /{accountNum}/api/Items` accepts:

- **Free-text search**: `search` — when set, other parameters are ignored.
- **Targeted filters** (ignored when `search` is set): `itemNumber`, `serial`, `manufacturer`, `importer`, `model`, `type` (array), `caliber`.
- **Pagination**: `take`, `skip`.

Pick one approach per call — either free-text or targeted filters, not both. For large inventories, prefer targeted filters because free-text is broader and slower.

## Item attributes

The full attribute set, populated at acquisition time and editable on a pending acquisition or via PUT (subject to compliance constraints on committed items):

| Field | Notes |
|---|---|
| `externalId` | Your per-firearm ID. **Must NOT be GUID-shaped** (32-char hex or hyphenated GUID form) — those collide with FastBound's GUID routing. Use a prefix, suffix, or non-hex encoding. See `common-patterns.md`. |
| `manufacturer` | **Physical marking on the firearm.** Not the marketing name. |
| `importer` | Required when imported. **Physical marking.** |
| `countryOfManufacture` | Required when the location's settings demand it. **Physical marking.** |
| `model` | **Physical marking.** |
| `serial` | **Physical marking.** |
| `caliber` | **Physical marking.** |
| `type` | Pistol, Revolver, Rifle, Shotgun, Frame/Receiver, Other Firearm, etc. (see swagger for valid strings) |
| `barrelLength` | Numeric (inches). |
| `totalLength` | Numeric (inches). |
| `condition` | New, Used, etc. (free-text). |
| `cost`, `price` | Strings (currency). |
| `mpn`, `upc`, `sku` | Inventory identifiers. `sku` is a string, **max 50 characters** (added August 2022). Distinct from `upc` (the GS1 barcode) and `mpn` (manufacturer part number) — `sku` is your stock-keeping unit, freely chosen. |
| `itemNumber` | Account-controlled; ignored unless manual entry is enabled. |
| `location` | Multi-location FFLs only. |
| `note` | Free-text. |
| `lightspeedItemID`, `lightspeedSerialID`, `lightspeedSaleID` | Lightspeed POS integration IDs. |

> **Match the firearm, not the catalog.** The five "physical marking" fields (manufacturer, importer, country of manufacture, model, serial, caliber) must reflect what's stamped on the firearm — not what's on the box, the website, or the spec sheet. ATF inspectors compare bound book entries to the firearms physically on hand. Marketing-derived fields produce findings.

## Updating: GET → modify → PUT

PUT replaces the entire item. **Any field omitted from the request body is set to null.** Always:

1. GET the item.
2. Modify the fields you intend to change.
3. PUT the full object back.

For external-ID-only updates, use `PUT /{id}/SetExternalId` instead — it's a targeted operation that doesn't have the null-erasure risk.

## Bulk operations

### Updating item external IDs in bulk

For one-time backfill from a legacy system, or any time you need to set/update the `externalId` on many items at once, FastBound provides an **official PowerShell script**: `FastBound-UpdateExternals.ps1`. Source: https://github.com/FastBound/Support/blob/main/scripts/FastBound-UpdateExternals.ps1. Reference article: https://fastbound.help/en/articles/8351562-updating-item-external-ids-in-bulk

Key facts:

- **Hard batch limit: 1000 items per API request.** The script handles chunking automatically; if you write your own bulk-update tool, respect this.
- Requires PowerShell 5.1+.
- CSV-driven: default columns are `Id` (FastBound item GUID) and `ExternalId`, but configurable via `-FastBoundIdColumnName` / `-ExternalIdColumnName`.
- Modes:
  - `-GenerateExternals` — generate sequential external IDs (default starting at 100) and write back to CSV. Useful if you don't already have IDs in your legacy system.
  - `-UpdateItems` — push the CSV's external IDs to FastBound.
  - Both flags together — generate then push in one run.

Example usage (generate + push, custom audit user):

```powershell
.\FastBound-UpdateExternals.ps1 `
  -CsvFile "items.csv" `
  -GenerateExternals `
  -UpdateItems `
  -AccountNumber <YourAccountNumber> `
  -ApiKey "<YourApiKey>" `
  -AuditUser user@example.com
```

> The script uses Basic Auth (API Key) and so passes `X-AuditUser`; if you build an OAuth-based equivalent, drop the audit-user header (the access token identifies the user).

### Automated bound-book downloads

For ATF Ruling 2016-1 compliance — see `references/bound-book-downloads.md`.

## Inventory endpoints

The "items currently in inventory" view is exposed via:

- `GET /{accountNum}/api/Inventory` — the live inventory list (items currently in stock).

This is distinct from `GET /{accountNum}/api/Items`, which can return disposed items as well. Use `Inventory` when you want "what's actually on hand right now."

## Webhook events for items

| Event | Fires when |
|---|---|
| `item.imported` | Item brought in via bulk import |
| `item.edited` | Item attributes edited |
| `item.deleted` | Item deleted (Duplicate/Error reasons only) |
| `item.undeleted` | Previously deleted item restored |
| `item.undisposed` | Disposed item undisposed (back in inventory) |
| `inventory.location.moved` | Item moved between locations (multi-location FFLs) |
| `inventory.location.verified` | Periodic inventory verification |
| `inventory.notfound` | Inventory verification: item couldn't be located |

`inventory.notfound` is a flag to investigate, not necessarily a sign of theft — items go missing in the database before they go missing on the shelf. Surface it to the operator promptly.

## Common pitfalls

- **Marketing copy in `manufacturer` / `model` / `caliber`** — must match the physical markings.
- **PUT-erasing fields** — omitted = null. Always GET-modify-PUT.
- **Generic "Delete" button** — show only with `Duplicate` or `Error` reason and a required note.
- **Confusing item with SKU** — items are individual firearms; SKUs/products are catalog concepts. Don't try to dedupe items by SKU.
- **Querying disposed items via `Inventory`** — `Inventory` is current stock. Use `Items` with appropriate filters to include disposed items.
- **Mismatched serial → ID after batch acquisition** — `Items/Multiple` may not return items in input order. Match by serial, not by index.
