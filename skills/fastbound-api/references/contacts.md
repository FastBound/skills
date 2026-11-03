# Contacts

A **contact** in FastBound is anyone or any entity on the other side of an acquisition or disposition: another FFL, a manufacturer, a distributor, an organization, an individual buyer. Every transaction in the bound book has a contact attached.

Reference article: https://fastbound.help/en/articles/3963657-contacts

## Three contact types

The same `Contact` resource is used for all three; you indicate which type by which fields you populate.

### 1. FFL / Licensee

Another federally-licensed entity — a manufacturer, distributor, dealer, gunsmith, importer, etc.

**Required:**
- `fflNumber` — their FFL license number. Hyphens are accepted; FastBound normalizes.
- `fflExpires` — license expiration date.
- `licenseName` — the legal name on the license.
- `tradeName` — the trade/DBA name.

**Optional but useful:**
- `sotein` — Special Occupational Tax EIN (for NFA dealers/manufacturers).
- `sotClass` — `1`, `2`, or `3` SOT class string.
- `businessType` — see swagger for valid strings.
- `lookupFFL` (boolean) — when `true`, FastBound will attempt to look up the FFL via FFLeZ (its licensee directory) and pre-fill license name, trade name, and address. If the lookup succeeds, the API auto-populates fields you didn't supply. Use this to reduce data-entry errors when the operator only has the FFL number.

### 2. Organization

A non-licensed business contact (e.g., a corporate buyer, a law-enforcement agency, an ammo wholesaler that doesn't ship firearms).

**Required:**
- `organizationName`

### 3. Individual

A natural person — the typical walk-in customer scenario.

> **This is PII.** An individual contact is a named human being with a residence address, and once a disposition attaches, the record says which firearm they took home. Don't copy it into your own database beyond what you need, don't log it, and don't put it in front of analytics or an AI assistant. See "This API carries other people's PII" in `references/common-patterns.md`.

**Required:**
- `firstName`
- `lastName`

**Optional:**
- `middleName`, `suffix`

## Address fields (required for all three types)

Every contact, regardless of type, requires a complete premise address:

- `premiseAddress1` (required)
- `premiseAddress2` (optional)
- `premiseCity` (required)
- `premiseCounty` (optional)
- `premiseState` (required)
- `premiseZipCode` (required)
- `premiseCountry` (required)

Plus optional contact info: `phoneNumber`, `fax`, `emailAddress`.

> **Premise vs mailing**: FastBound stores the *premise* address — the actual business location for FFLs, the residence/billing address for individuals. Don't confuse this with shipping addresses; that's separate from the contact record.

## API operations

| Operation | Endpoint |
|---|---|
| List contacts | `GET /{accountNum}/api/Contacts` (with filters: `licenseName`, `tradeName`, `fflNumber`, `organizationName`, `firstName`, `middleName`, `lastName`, `suffix`, etc.) |
| Get one by ID | `GET /{accountNum}/api/Contacts/{id}` |
| Get one by external ID | `GET /{accountNum}/api/Contacts/GetByExternalId/{externalId}` |
| Create | `POST /{accountNum}/api/Contacts` |
| Update | `PUT /{accountNum}/api/Contacts/{id}` |
| Delete | `DELETE /{accountNum}/api/Contacts/{id}` |

You can also create a contact inline as part of `Acquisitions/CreateAndCommit` or `Dispositions/CreateAndCommit` by embedding the contact object instead of providing a `contactId` — useful for one-shot transactions where you don't need to keep the contact around.

## Duplicate detection — the `X-FastBound-ExistingContactId` header

When you POST a contact and FastBound detects it as a likely duplicate of an existing one (matched on FFL number for licensees, name+address for individuals/organizations), the response includes:

- `X-FastBound-ExistingContactId` — the GUID of the existing contact
- `X-FastBound-ExistingContactUrl` — UI URL for the existing contact

**Do this:** use the returned existing contact ID instead of creating a duplicate. Surface the URL to the operator if they need to confirm. Don't paper over the warning by ignoring the headers — duplicates clutter the contact list and split history.

See `common-patterns.md` for the full table of `X-FastBound-*` response headers.

## `null` vs empty string — be careful

A subtle but important distinction in the FastBound API:

- **Setting a field to `null` erases its value.**
- **Setting a field to an empty string (`""`) does not erase its value** in the same way; it sets it to empty.

For PUT updates, this matters because PUT replaces the entire object: any field you *omit* from the body becomes `null`. If you GET a contact, drop a field you didn't intend to touch, and PUT it back, you've erased that field. Always GET → modify → PUT the full object back, including unchanged fields.

```python
# WRONG — omits emailAddress, which becomes null
existing = get_contact(contact_id)
put_contact(contact_id, {"phoneNumber": "555-1234"})

# RIGHT — modify the full object and PUT it back
existing = get_contact(contact_id)
existing["phoneNumber"] = "555-1234"
put_contact(contact_id, existing)
```

## Identifier strategy: FastBound IDs vs External IDs

**Store FastBound's Contact ID** (and Item, Acquisition, and Disposition IDs) in your own database. Extending your application to hold them is the recommended approach, and `externalId` should be reserved for cases where that's absolutely not possible:

- "Storing FastBound Identifiers is more efficient because you can infer from the presence of the ID whether the record exists in FastBound."
- Contact and item IDs are RFC 9562 v4 GUIDs.
- Lookup-by-external-ID adds an API call and complexity at every reference point — negligible for one FFL, expensive for a high-volume or multi-tenant SaaS integration.
- Troubleshooting is far easier when the ID in your database is the same ID in FastBound's UI, logs, and support tickets — and backfills become a join on a stable key.

`externalId` is still useful as a *secondary* index — for example, to handle reconciliation when your local DB is rebuilt or to make creates idempotent. But the primary key for cross-system references should be FastBound's ID. See `common-patterns.md` for the full identifier strategy.

## FFLeZ lookup

For FFL contacts, set `lookupFFL: true` and provide just the `fflNumber` to have FastBound auto-populate license name, trade name, expiration, and address from its FFLeZ directory:

```json
{
  "fflNumber": "1-99-XXX-XX-XX-99999",
  "fflExpires": "2027-01-01",
  "lookupFFL": true,
  "premiseAddress1": "123 Main St",
  "premiseCity": "Springfield",
  "premiseState": "IL",
  "premiseZipCode": "62701",
  "premiseCountry": "US"
}
```

Some fields are still required (you must provide the address even if FFLeZ has it; the lookup may fill or correct fields you supply). Check the response to confirm what was populated.

## Worked example — find-or-create a distributor

Two requests: look up by FFL number, and create only if nothing came back.

**1. Search first.** If the `contacts` array is non-empty, take the first ID and you're done — no write needed.

```http
GET /12345/api/Contacts?fflNumber=1-99-XXX-XX-XX-99999 HTTP/1.1
Host: cloud.fastbound.com
Authorization: Bearer <access_token>
```

**2. Not found — create it.** `lookupFFL: true` has FastBound fill licenseName, tradeName, expiration, and address from FFLeZ. Supply the premise address anyway; the API may still require those fields up front, and FFLeZ may correct rather than replace what you send.

```http
POST /12345/api/Contacts HTTP/1.1
Host: cloud.fastbound.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "fflNumber": "1-99-XXX-XX-XX-99999",
  "fflExpires": "2027-01-01",
  "lookupFFL": true,
  "premiseAddress1": "123 Main St",
  "premiseCity": "Springfield",
  "premiseState": "IL",
  "premiseZipCode": "62701",
  "premiseCountry": "US"
}
```

**3. Check the response headers before trusting the body.** FastBound runs its own duplicate detection on create, and when it matches an existing contact it tells you in the headers rather than failing the request:

```http
X-FastBound-ExistingContactId: 8f14e45f-ceea-467a-9c1a-1b2c3d4e5f60
X-FastBound-ExistingContactUrl: https://cloud.fastbound.com/...
```

When those are present, **use that ID** and treat the operation as a find, not a create — and surface the URL if an operator needs to confirm the match. Ignoring these headers is how duplicate distributors accumulate in an account over months. See the duplicate-detection section above, and `common-patterns.md` for the full set of `X-FastBound-*` response headers.

(Under legacy Basic Auth, add `X-AuditUser` to the POST. The GET is a read and never needs it.)

## Common pitfalls

- **Forgetting required address fields** even when supplying just an FFL number — premise address is required for all contact types.
- **Treating empty-string and null as equivalent** — they're not. PUT with omitted field → null → erased.
- **Ignoring `X-FastBound-ExistingContactId` on POST** — silently creates duplicates over time.
- **Not using FFLeZ (`lookupFFL: true`)** when only the FFL number is known — leads to typos in licenseName / tradeName / address.
- **Storing only `externalId` and re-querying constantly** — costs API calls and round trips, and makes troubleshooting harder. Store FastBound's Contact ID; treat `externalId` as a fallback for schemas you can't extend.
- **Using a GUID-shaped value as the contact `externalId`** — collides with FastBound's GUID routing on `GetByExternalId` and similar paths. Prefix, suffix, or use a non-hex encoding. See `common-patterns.md`.
- **Searching for an individual by partial name without enough qualifiers** — common names produce noisy results; use `firstName` + `lastName` + city/state at minimum.
