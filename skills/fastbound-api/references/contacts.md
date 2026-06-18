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

The retailer integration guide explicitly recommends **storing FastBound's GUIDs for contacts** (and items) in your own database, rather than relying on `externalId` lookups everywhere:

- "Storing FastBound Identifiers is more efficient because you can infer from the presence of the ID whether the record exists in FastBound."
- Contact and item IDs are RFC 9562 v4 GUIDs.
- Lookup-by-external-ID adds an API call and complexity at every reference point.

`externalId` is still useful as a *secondary* index — for example, to handle reconciliation when your local DB is rebuilt or to make creates idempotent. But the primary key for cross-system references should be FastBound's ID where you can.

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

## Worked example — find-or-create a distributor (Python, OAuth)

```python
def find_or_create_distributor(ffl_number: str, ffl_expires: str) -> str:
    # Try to find by FFL number first.
    r = requests.get(
        f"{BASE}/{ACCOUNT_NUM}/api/Contacts",
        params={"fflNumber": ffl_number},
        headers={"Authorization": f"Bearer {token}"},
    )
    r.raise_for_status()
    contacts = r.json().get("contacts", [])
    if contacts:
        return contacts[0]["id"]

    # Not found — create. Set lookupFFL to let FastBound fill in the rest.
    r = requests.post(
        f"{BASE}/{ACCOUNT_NUM}/api/Contacts",
        json={
            "fflNumber": ffl_number,
            "fflExpires": ffl_expires,
            "lookupFFL": True,
            # FFLeZ may fill licenseName/tradeName/address, but the API
            # may still require the premise address fields up front:
            "premiseAddress1": "...",
            "premiseCity": "...",
            "premiseState": "...",
            "premiseZipCode": "...",
            "premiseCountry": "US",
        },
        # OAuth: no X-AuditUser needed. Basic Auth: include it.
        headers={"Authorization": f"Bearer {token}"},
    )

    # Check duplicate-detection headers — if FastBound matched an existing contact,
    # use that ID instead of treating this as a new one.
    if "X-FastBound-ExistingContactId" in r.headers:
        return r.headers["X-FastBound-ExistingContactId"]

    r.raise_for_status()
    return r.json()["id"]
```

## Common pitfalls

- **Forgetting required address fields** even when supplying just an FFL number — premise address is required for all contact types.
- **Treating empty-string and null as equivalent** — they're not. PUT with omitted field → null → erased.
- **Ignoring `X-FastBound-ExistingContactId` on POST** — silently creates duplicates over time.
- **Not using FFLeZ (`lookupFFL: true`)** when only the FFL number is known — leads to typos in licenseName / tradeName / address.
- **Storing only `externalId` and re-querying constantly** — costs API calls and round trips. Store FastBound's GUIDs for contacts you'll reference frequently.
- **Using a GUID-shaped value as the contact `externalId`** — collides with FastBound's GUID routing on `GetByExternalId` and similar paths. Prefix, suffix, or use a non-hex encoding. See `common-patterns.md`.
- **Searching for an individual by partial name without enough qualifiers** — common names produce noisy results; use `firstName` + `lastName` + city/state at minimum.
