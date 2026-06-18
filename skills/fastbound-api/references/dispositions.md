# Dispositions

A **disposition** is the "transfer out" side of the bound book — every firearm leaving an FFL's inventory does so through a disposition record. Common cases: walk-in customer sale (with ATF Form 4473), online sale shipped to a receiving FFL, NFA transfer, theft or loss reporting, destruction, manufacturing consumption.

Reference articles:
- https://fastbound.help/en/articles/3963658-disposing-items
- https://fastbound.help/en/articles/3963660-recalling-a-4473
- https://fastbound.help/en/articles/9326601-fastbound-integration-guide-for-firearms-retailers (POS workflow)

## The cardinal rule

**Never commit a disposition until the firearm has physically left the premises.** This is non-negotiable.

In states with waiting periods (e.g., California's 10-day wait), payment happens days before pickup. If you commit on payment, the firearm is marked disposed in the bound book *while it's still in the safe*. If ATF inspects during that window, every prematurely-committed disposition is a violation. Inspections are not rare.

The integration pattern: payment, 4473 submission, NICS, waiting period — all proceed as pending. Commit only happens at the **handoff** (a human action: an employee scans the firearm out, confirms with the customer, or marks "delivered"). Never auto-commit on a payment webhook, an order status change, or a calendar timer.

This applies to walk-in sales, ship-to-FFL transfers, NFA transfers, manufacturing dispositions — every type. The only exceptions are the dedicated reporting endpoints (TheftLoss, Destroyed) where the event itself implies the firearm is gone.

## Lifecycle: Pending → Committed (and the special states)

```
[POST /Dispositions]                          → pending
[POST /Dispositions/{id}/Commit]              → committed (out of inventory)
```

Same two-state model as acquisitions, with stronger commit-time discipline.

Special creation modes:

- `POST /Dispositions/CreateAndCommit` — atomic create+commit. **Only use this when the firearm is already physically gone** (e.g., recording an after-the-fact transfer, or shipping where the truck has departed and the BOL is in hand).
- `POST /Dispositions/CreateAsPending` — alias for the standard pending create; useful for explicit intent.
- `POST /Dispositions/Lock/{id}` and `LockByExternalId/{externalId}` — lock a pending disposition (prevent further edits while a 4473 or NICS check is in progress).

## Disposition types

The `type` field is a string. Valid values are enumerated in the swagger (`Sale`, `Transfer`, etc.). One important special case:

> **A `null` disposition `type` indicates the disposition is related to a 4473** (i.e., it represents a non-licensee sale captured on Form 4473). The disposition gets its substantive type when the 4473 is completed.

Three dedicated endpoints exist for special non-routine disposition reasons:

| Endpoint | When to use |
|---|---|
| `POST /Dispositions/NFA` | NFA transfers (machine guns, suppressors, SBRs, SBSs, AOWs, destructive devices) — special tax-stamp and Form 4 considerations apply. Creates a **pending shell only** — no inline contact or items; attach them afterward (see "NFA dispositions are a pending shell" below). |
| `POST /Dispositions/TheftLoss` | Reporting stolen or lost firearms (ATF Form 3310.11 federally; check state requirements too). |
| `POST /Dispositions/Destroyed` | Recording firearms that have been destroyed (e.g., damaged beyond use, court-ordered). |

These don't follow the normal "wait for handoff" pattern because the event is the disposition.

### NFA dispositions are a pending shell

`POST /Dispositions/NFA` (`CreateNfaDisposition.Command`) does **not** accept a contact or items inline. Its entire accepted payload is: `externalId`, `date`, `submissionDate`, `type` (**required**), `note`, `ttsn`, `generateTTSN`, `otsn`, `purchaseOrderNumber`, `invoiceNumber`, `shipmentTrackingNumber` — and the schema is `additionalProperties: false`, so the API rejects any attempt to smuggle `contact`/`contactId`/`items` into the create call.

So you cannot one-shot an NFA disposition the way you can with `CreateAndCommit`. The flow is:

1. `POST /Dispositions/NFA` — creates a pending shell (returns the disposition id).
2. `PUT /Dispositions/{id}/AttachContact/{contactId}` — attach the receiving party.
3. `POST /Dispositions/{id}/Items` (or the by-external-id / by-search variants) — add the firearm(s).
4. `POST /Dispositions/{id}/Commit` — commit when the firearm physically leaves.

Two differences from the standard create to keep in mind:

- **`type` is required** on the NFA create. The "`type=null` means 4473-related" convention does **not** apply here.
- It exposes a `submissionDate` field (otherwise only seen on `EditDisposition`).
- `ttsn`/`otsn`/`generateTTSN` are present on this command, but per the 4473-gating rule they still should not be set at creation — see the field table below.

## Standard four-step workflow (transfer-out)

1. **Have the contact ready.** Either look it up by `externalId`, retrieve by ID, or create new. See `contacts.md`. For non-licensee sales captured via 4473, the contact is typically created by FastBound's UI when the buyer fills out the form.
2. **Create the pending disposition:**
   - `POST /{accountNum}/api/Dispositions`
3. **Attach contact and add items:**
   - `PUT /{accountNum}/api/Dispositions/{id}/AttachContact/{contactId}`
   - `POST /{accountNum}/api/Dispositions/{id}/Items` (single) or `Items/Multiple` (batch)
4. **Commit when the firearm physically leaves:**
   - `POST /{accountNum}/api/Dispositions/{id}/Commit`

## Walk-in customer sale (4473) workflow

Walk-in sales are different because the **4473 is the source of truth for the buyer's data**. FastBound captures the 4473 directly (eForm 4473), and the disposition takes shape as the 4473 progresses. Your integration's job isn't to recreate the 4473 — it's to *react* to it via webhooks and synchronize POS state.

Subscribe to these events (`webhooks.md`):

| Event | Meaning | Action |
|---|---|---|
| `disposition.items.added` | Items added to a pending disposition (often by the 4473 flow) | Update POS line items |
| `disposition.items.edited` | Sale price or item details changed on a pending disposition | Update POS pricing |
| `disposition.items.removed` | Items removed from pending | Update POS |
| `form4473.completed` | Buyer finished the 4473; PDF available | Capture the PDF URL for record retention; gate handoff on this signal |
| `disposition.committed` | Disposition committed to A&D | Final state — sale is recorded |

The end-to-end POS pattern:

1. Customer brings firearms to the counter.
2. POS records the items pending sale (in your system).
3. Operator launches the 4473 (in FastBound or via deep link); buyer fills it out on a tablet.
4. NICS check runs (FastBound or external).
5. After waiting period (if any), customer returns to pick up.
6. **Operator clicks "deliver" in your POS, which calls** `POST /Dispositions/{id}/Commit`. This is the *only* moment commit happens.
7. Inspect the response headers — see "Multiple Sale" below.

## Recalling a 4473 (linking a pending 4473 disposition to your POS sale)

If a customer started a 4473 in FastBound and then your POS needs to "pick it up" (e.g., to ring up the sale, attach payment), use:

- `GET /{accountNum}/api/Dispositions/Only4473s` — lists pending dispositions that are 4473-related (filters out non-4473 dispositions).
- `GET /{accountNum}/api/Dispositions` with appropriate filters can also include 4473s.

Match the pending 4473 disposition to your POS transaction (typically by serial number on the items, or by an external ID you set when the 4473 was initiated). Then proceed with the standard "commit on handoff" pattern.

## Inter-FFL transfer (ship to receiving FFL)

When a sale is shipped to a destination FFL (e.g., online order to a customer who picks up at their local FFL), the disposition contact is the **receiving FFL** (a Licensee contact), not the end customer.

The seller's flow:

1. Look up or create the receiving FFL contact (`contacts.md`).
2. Create the pending disposition with the receiving FFL as the contact.
3. Add items.
4. **Ship the firearm.** Confirm departure (BOL / shipping label scanned).
5. Commit the disposition.

If the receiving FFL also uses FastBound (matched by the contact's email belonging to a FastBound user), commit triggers an **auto-pending-acquisition** in their account. You'll see these response headers:

- `X-FastBound-AcquisitionAccount` — receiver's account number
- `X-FastBound-AcquisitionId` — receiver's pending acquisition ID
- `X-FastBound-AcquisitionUrl` — deep link the receiver can open

Capture these for cross-system reconciliation. If you orchestrate the transfer end-to-end, notify the receiving FFL out-of-band that a pending acquisition awaits their review (they still have to commit it themselves; you can't commit on their behalf).

## Multiple Sale Reports — DO NOT MISS THESE

When you commit a disposition to a non-licensee individual that triggers ATF Form 3310.4 (two or more handguns to the same buyer within five business days), FastBound generates the multiple-sale report and signals it on the **commit response**:

```http
HTTP/1.1 200 OK
X-FastBound-MultipleSale: true
X-FastBound-MultipleSaleUrl: https://cloud.fastbound.com/...
```

**Required action:** surface this in your UI immediately. The licensee has a regulatory obligation to review the report and either transmit it to ATF or dismiss it. **If your integration silently ignores the header, the licensee may miss a required filing — that's a compliance failure, not a UX issue.**

A common, costly bug: integrators see HTTP 200 on commit, log "success," and move on. The disposition is in FastBound, but the multiple-sale notification never reaches the operator. Don't be that integration.

## Manufacturing dispositions

For licensed manufacturers (Type 07/10 FFLs) assembling a firearm from a serialized lower receiver/frame already in inventory:

- Create a **manufacturing disposition** that consumes the lower receiver. The disposition's contact defaults to the account contact (the manufacturer to itself).
- Set the manufacturing flag and include the `manufacturingChanges` object describing what the resulting firearm is (manufacturer, model, caliber, type — the physical markings that will appear on the assembled firearm).
- On commit, FastBound automatically creates a **pending acquisition** in the same account for the completed firearm. That pending acquisition must then be reviewed and committed via `POST /Acquisitions/{id}/Commit`.

This two-step (manufacturing disposition consumes the lower → auto-pending acquisition for the finished firearm) is the audit-trail requirement for manufacturers: every transformation of a serialized component into a finished firearm is recorded with both sides of the ledger.

Manufacturing dispositions to wholesale customers (selling completed firearms to other FFLs) follow the standard inter-FFL transfer flow, not the manufacturing disposition flow.

Reference: https://fastbound.help/en/articles/11421255-fastbound-integration-guide-for-firearms-manufacturers

## Disposition fields

Top-level (`POST /Dispositions` or `CreateAndCommit`):

| Field | Type | Notes |
|---|---|---|
| `externalId` | string | Your system's ID. |
| `date` | date | Business date of disposition. **Cannot be in the future.** |
| `type` | string | Disposition type — see swagger. **`null` means 4473-related.** |
| `note` | string | Free-text. |
| `ttsn` | string | Transferor's Transaction Serial Number. **Only populated once a 4473 is completed — do not set at disposition creation.** |
| `generateTTSN` | boolean | Have FastBound generate the TTSN. **Tied to 4473 completion — do not set at creation (see `ttsn`).** |
| `otsn` | string | Other Transaction Serial Number (e.g., the state's transaction serial number). **Only populated once a 4473 is completed — do not set at disposition creation.** |
| `purchaseOrderNumber`, `invoiceNumber`, `shipmentTrackingNumber` | string | Searchable identifiers. |
| `isManufacturingDisposition` | boolean | True for manufacturing dispositions consuming a serialized component. |
| `contactId` (or `contactExternalId`, or nested `contact`) | varies | The party receiving the firearm. |
| `items` | array | When using `CreateAndCommit`. |

Item-level fields when adding to a disposition: serial reference (existing item ID or external ID), price (sale price), and disposition-specific fields per the swagger.

## Editing items on a pending disposition

These endpoints exist specifically for the 4473/sale flow:

- `POST /{accountNum}/api/Dispositions/{id}/Items` — add items by ID
- `POST /{accountNum}/api/Dispositions/{dispositionExternalId}/Items` — add by your disposition ID
- `POST /{accountNum}/api/Dispositions/{dispositionExternalId}/Items/AddByExternalId` — add by item external ID
- `POST /{accountNum}/api/Dispositions/{dispositionExternalId}/Items/AddBySearch` — add by search criteria
- `PUT /{accountNum}/api/Dispositions/{id}/Items/EditPrice/{itemId}` — change sale price
- `PUT /{accountNum}/api/Dispositions/{id}/Items/{dispositionItemId}` — full edit
- `DELETE /{accountNum}/api/Dispositions/{id}/Items/Remove/{itemId}` — remove
- `DELETE /{accountNum}/api/Dispositions/{id}/Items/RemoveByExternalId/{itemExternalId}` — remove by external ID

## Search and lookup

- `GET /{accountNum}/api/Dispositions` — list with `take`/`skip` and filters
- `GET /{accountNum}/api/Dispositions/{id}` — by ID
- `GET /{accountNum}/api/Dispositions/{externalId}` — by external ID (path-overloaded; check swagger for the pattern)
- `GET /{accountNum}/api/Dispositions/GetByExternalId/{externalId}` — explicit external-ID lookup
- `GET /{accountNum}/api/Dispositions/Only4473s` — pending dispositions tied to a 4473

## Webhook events for dispositions

| Event | Fires when |
|---|---|
| `disposition.committed` | Disposition committed (final state) |
| `disposition.deleted` | Pending disposition deleted |
| `disposition.edited` | Pending disposition edited |
| `disposition.items.added` | Items added to a pending disposition |
| `disposition.items.edited` | Item details/price changed |
| `disposition.items.removed` | Items removed |
| `form4473.completed` | The 4473 attached to the disposition was completed by the buyer |
| `multiplesalereport.transmitted` | A multiple sale report was transmitted to ATF (closes the loop on the `X-FastBound-MultipleSale` header) |

> **Note**: there is no `disposition.created` event. Pending dispositions don't have a creation event of their own; the integration sees the lifecycle through `disposition.items.added`/`edited`/`removed` and the eventual `disposition.committed`. Don't try to subscribe to `disposition.created` — it's not in the catalog. Same logic as `Dispositions/CreateAndCommit`: only `disposition.committed` fires for that atomic path.

For a POS integration, subscribing to `disposition.items.added/edited/removed`, `form4473.completed`, and `disposition.committed` covers the lifecycle. See `webhooks.md`.

## Common pitfalls

- **Auto-committing on payment** — never. Commit only on physical departure.
- **Ignoring `X-FastBound-MultipleSale`** — compliance failure waiting to happen.
- **Ignoring `X-FastBound-Acquisition*` headers on inter-FFL commit** — the receiving FFL has work to do; tell them.
- **Forgetting `X-AuditUser` under Basic Auth** — every write fails. (Not needed under OAuth; the token identifies the user.)
- **Auto-retrying a 422** — if commit fails because the firearm hasn't been added or the contact is invalid, retrying won't help. Surface the error and require manual fix.
- **Treating `type=null` as missing data** — null means "this is a 4473-related disposition; the type will be set when the 4473 is completed."
- **Setting `ttsn`/`otsn`/`generateTTSN` at disposition creation** — these are transaction serial number fields (`ttsn` = transferor's, `otsn` = other, e.g. the state's), not NFA tax-stamp numbers. They're only populated once a 4473 is completed; leave them unset on the create call.
- **Trying to pass `contact`/`items` to `POST /Dispositions/NFA`** — it creates a pending shell only and rejects unknown properties (`additionalProperties: false`). Attach the contact and add items in follow-up calls, then commit.
- **PUT erases unspecified fields** — GET first, modify, PUT.
