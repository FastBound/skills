# Common patterns

Cross-cutting conventions that apply across the FastBound API. Read this once; everything else is shape.

## Every request

| Header | When | Value |
|---|---|---|
| `Authorization` | Always | `Bearer <access_token>` (OAuth) or `Basic <base64(api_key:api_key)>` (legacy) |
| `X-AuditUser` | **Basic Auth only** — every write (POST/PUT/DELETE). NOT required for OAuth — the access token already identifies the user. | Email of an active FastBound user with access to the account |
| `Content-Type` | Requests with a body | `application/json` (form-urlencoded for OAuth endpoints only) |
| `Accept` | Optional but recommended | `application/json` |
| `x-api-version` | Optional | `1.0` (current) |

URL pattern for the accounts API:

```
https://cloud.fastbound.com/{accountNum}/api/{Resource}/...
```

`accountNum` is assigned by FastBound to the licensee. Where you get it depends on the auth mechanism:

- **OAuth** — the token endpoint returns it as `fastbound_account_number` (with `fastbound_account_name`) on every grant type. Persist it with the tokens; it's per-connection data, not global config, and a token is bound to exactly one account. Don't ask the user to type it in. See `references/authentication.md`.
- **Basic Auth (legacy)** — nothing returns it. It's configuration, supplied by the licensee (they can read it out of the URL bar in the FastBound UI).

Either way, don't try to discover it by probing paths at runtime.

## The audit user header — depends on auth mechanism

Whether you need to send `X-AuditUser` depends on which auth mechanism you're using:

### OAuth — `X-AuditUser` is NOT required

The OAuth access token already encodes the identity of the user who authorized the integration. FastBound uses that user as the audit identity automatically. Don't send `X-AuditUser` on OAuth writes; if you do (for example during a Basic Auth → OAuth migration where you haven't removed the header yet), it's silently accepted.

**The bundled spec contradicts this, and you will hit it.** `swagger/accounts-swagger.json` declares `X-AuditUser` on 52 write operations and marks it `required: true` on every one, unconditionally — it does not model the header as depending on which auth mechanism you used. Follow the guidance above rather than the spec here: the requirement is real for Basic Auth and not for OAuth. It matters in two practical ways. A client generated from the spec will make the header mandatory, and the fix is to make it optional in your generated client, never to invent an audit-user value to satisfy it. And if you are reading the spec to settle an argument about this, it cannot settle it — confirm with FastBound support if you need certainty for a specific endpoint.

For server-to-server OAuth integrations, enroll a service-account FastBound user (e.g., `integration@example.com`) in the account and authorize the OAuth flow as *that* user — whether the account is your own FFL's or one you integrate on behalf of. Everything the integration writes is then attributed to that service account in the audit log.

### Basic Auth (legacy) — `X-AuditUser` IS required on every write

```
X-AuditUser: jdoe@example.com
```

The API Key identifies the *integration*, not the human. To preserve the audit trail, every write must include the email of a FastBound user. The email must:

1. Belong to an existing FastBound user.
2. Have access to the account specified in `accountNum`.
3. Have a permission level appropriate for the operation.

If any of these fail, the request fails. Treat `X-AuditUser` as a required field on every POST/PUT/DELETE under Basic Auth.

## Identifiers: store FastBound's IDs

> **Recommended: extend your application to store FastBound's identifiers** — Contact ID, Acquisition ID, Item ID, Disposition ID. Reach for `externalId` only when that's genuinely not possible.

Every FastBound record has a FastBound-assigned ID (an RFC 9562 v4 GUID), returned in the create response and on every read. Persisting those IDs alongside your own records makes for a **more reliable and dependable integration**:

- **Unambiguous.** A FastBound ID resolves to exactly one record, with no routing ambiguity and no uniqueness rules you have to enforce yourself.
- **Fewer round trips.** You address records directly. Working from an `externalId` often means an extra API call just to resolve it to the actual ID before you can do the thing you actually came to do.
- **Existence is knowable locally.** The presence of a stored FastBound ID tells you the record exists in FastBound — no lookup required.
- **Easier to troubleshoot.** The ID in your database is the same ID in FastBound's UI, logs, and support tickets. Correlating the two sides of an issue is a copy-paste, not an investigation.
- **Easier to backfill and reconcile.** Repairing or re-syncing data is a straightforward join on a stable key.

The cost is a column (or a few) in your schema. That's the whole trade, and it's worth paying up front — retrofitting FastBound IDs into an integration that assumed `externalId` is far more work than including them from the start.

### The account number arrives unquoted — don't type it as a string

The **account number** is a JSON number, not a JSON string:

```json
{ "number": 12345, "name": "ACME Guns LLC" }
```

Note `12345`, not `"12345"`. It's declared `integer`/`int64` in exactly two places, and they're the two you meet first:

| Field | Where |
|---|---|
| `fastbound_account_number` | the OAuth token response |
| `number` | `GET /{accountNum}/api/Account` |

This bites in strictly-typed languages, where a model declaring `string number` fails to deserialize the response outright — in .NET, `System.Text.Json` throws *"Cannot get the value of a token type 'Number' as a string"* and the call dies at parse time, after a perfectly successful HTTP round trip. Go, Java, Swift, and Kotlin have the same sharp edge; dynamically-typed clients absorb it silently and then compare `12345 == "12345"` somewhere later.

**Don't over-generalize this.** Most FastBound identifiers really are strings, and a blanket "numbers might be unquoted" rule adds converters you don't need:

- `itemNumber` — **string** everywhere it appears.
- Record IDs (Item, Contact, Acquisition, Disposition) — always quoted GUIDs.
- A contact license `number` (the FFL number) — **string**, and a different concept from the account number despite the shared field name.

Handle the account number, and let the rest be what the spec says. Normalize it to a **string** in your own model: it's an identifier, never arithmetic, and comparisons and leading zeros both behave better as text.

```csharp
// .NET: a converter that accepts either shape, applied to the account number.
public sealed class NumberOrStringConverter : JsonConverter<string?>
{
    public override string? Read(ref Utf8JsonReader reader, Type _, JsonSerializerOptions __) =>
        reader.TokenType switch
        {
            JsonTokenType.String => reader.GetString(),
            JsonTokenType.Null   => null,
            JsonTokenType.Number => reader.GetInt64().ToString(CultureInfo.InvariantCulture),
            _ => throw new JsonException($"Expected a string or a number, got {reader.TokenType}."),
        };

    public override void Write(Utf8JsonWriter writer, string? value, JsonSerializerOptions _) =>
        writer.WriteStringValue(value);
}
```

```python
# Python: normalize at the boundary, don't scatter str() calls through the codebase.
account_number = str(payload["number"])
```

### When to use `externalId`

`externalId` is your system's ID for the same record, stored by FastBound. It exists for cases where storing FastBound's IDs genuinely isn't feasible — a legacy system whose schema you can't extend, a one-way import, a migration in progress. **Use it when you must, not by default.** FastBound recommends that integrated applications remain responsible for the storage of their own identifiers.

**Webhooks don't carry it.** This is the one that breaks integrations built on `externalId`, and it's worth checking against the bundled `swagger/webhooks-swagger.json` before you design around it: **the parent record's `externalId` is absent from most event payloads.** Every acquisition event, every disposition event, `form4473.completed`, `multiplesalereport.transmitted`, the inventory events, and `item.imported` deliver the record with its FastBound `id` and no `externalId` at all. Only the three `contact.*` events and four of the `item.*` events (`deleted`, `edited`, `undeleted`, `undisposed`) carry one, because there the payload *is* the entity.

Contacts and items nested **inside** an acquisition or disposition payload do carry their own `externalId`. The parent doesn't.

So an integration that keys on `externalId` cannot resolve an inbound `acquisition.committed` or `disposition.committed` to its own record from the payload. It has to call back to FastBound to fetch the record it was just handed — a round trip per event, on every event, to recover an identifier it would already have if it had stored the `id`. An integration that stored FastBound's IDs matches on the payload it already has and makes no call at all.

**Watch the call volume.** Building on `externalId` usually means an extra round trip per operation to resolve it to the real ID anyway. For a single FFL posting a handful of records a day, that's fine — the cost is invisible. For a **high-volume integration or a multi-tenant SaaS product** serving many accounts, it's pure waste: it roughly doubles your request count against the same rate limits, adds a failure point and a latency hop to every write path, and scales linearly with your customer base. Store the IDs instead.

**Steer away from it, including for idempotent creates.** The pre-POST existence check used to
be the one case worth keeping, and it is the case to be most careful about, because it is where
an integration adopts `externalId` "just for safety" and then builds its whole identity model on
it. It is also the weakest place to rely on it: an acquisition found by `externalId` is still
*pending*, since committing consumes the record — so a hit tells you a create landed, never that
the record was committed, and an integration that reads it as "already recorded" reports success
while firearms sit outside the book.

**Reconcile after the fact instead of checking before.** When a create's response is lost, look
the record up by the attributes it already carries, which cost you no extra identifier and no
extra design commitment:

| Looking for | Filter |
|---|---|
| An acquisition or disposition | `GET /Acquisitions` or `GET /Dispositions` with `purchaseOrderNumber`, `invoiceNumber` or `shipmentTrackingNumber` |
| An item | `GET /Items?serial=` — a serial is unique in an account, which is the property that makes this work |
| A licensee contact | `GET /Contacts?fflNumber=` |

Two honest limits. There is no natural-attribute filter for an individual (non-licensee) contact,
so a duplicate contact created by a retry is found by search or not at all — one reason to create
contacts once and store their id rather than per transaction. And these filters answer "did my
write land", not "was it committed"; for that, see "Committing consumes the pending record — so a repeated Commit returns 404" below.

If a local database is rebuilt, reconcile the same way. `externalId` is not the recovery
mechanism it appears to be, and reaching for it in that moment is how it becomes permanent.

Treat it as that secondary index. The primary key for cross-system references should be FastBound's ID wherever you can manage it.

Lookup endpoints, when you need them:

- `GET /{accountNum}/api/Acquisitions/GetByExternalId/{acquisitionExternalId}`
- `GET /{accountNum}/api/Dispositions/GetByExternalId/{externalId}`
- Disposition item operations support `AddByExternalId` and `RemoveByExternalId`

Resources that accept `externalId`: `Acquisitions`, `AcquisitionItems`, `Dispositions`, `DispositionItems`, `Contacts`, `Items`. The `externalId` doesn't have to be globally unique — only unique within its resource type within the account.

### If you do use externalId, don't make it look like a GUID

> **Hard rule**: never use a **32-character hexadecimal string** (16 bytes hex-encoded — i.e., an unhyphenated GUID, an MD5 hex digest, or any other 32-char hex value) as an `externalId`. By extension, avoid the 36-char hyphenated GUID form too.

Why: several FastBound endpoints are path-overloaded between FastBound's GUID identifier and the integrator's `externalId` (e.g., `GET /{accountNum}/api/Acquisitions/{id}` and `GET /{accountNum}/api/Acquisitions/{externalId}` — the routing distinguishes between them by shape). If your `externalId` is shaped like a GUID, FastBound's router resolves it as a GUID lookup, not an external-ID lookup. Result: your "find by external ID" call either 404s, or — worst case — silently returns a *different* record that happens to have a matching GUID. Both are bad.

The fix is purely client-side: ensure your `externalId` is **not** a 32-character hex string. Easy ways to do that:

- **Prefix it.** `PO-{po_number}` instead of `{md5_of_po_number}`. `INV-9921` not `7f9c8a...`. Even a single-character prefix (`x` + GUID) is enough.
- **Suffix it.** `{guid}-acq` works.
- **Use a non-hex encoding.** Base32, base62, ULIDs (Crockford base32, 26 chars), or KSUIDs all sidestep the GUID shape entirely.
- **Use a different separator.** `2026:04812:rev1` is fine.
- **Just don't hex-encode hashes.** If you're tempted to use a hash digest as the external ID, base32 or base64 it instead.

Quick sanity test: `re.fullmatch(r"[0-9a-fA-F]{32}|[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}", external_id)` — if that matches, your external ID will collide.

This applies anywhere FastBound takes an `externalId` segment in a URL path: `Acquisitions/{externalId}`, `Items/GetByExternalId/{externalId}`, `Dispositions/{dispositionExternalId}/Items/AddByExternalId`, etc. Pick a scheme up front, apply it everywhere.

### Don't use the `lightspeed*` fields

Items and acquisition items expose `lightspeedItemID`, `lightspeedSerialID`, and `lightspeedSaleID`. These exist to support FastBound's integration with **Lightspeed's R-Series** and **may be removed in the future**. Applications should not use them — not to store their own identifiers, and not as a lookup key to depend on.

As with `externalId`, FastBound recommends that integrated applications remain responsible for the storage of their own identifiers. To correlate a FastBound record with a record in your system, store FastBound's ID on your side — or, if that isn't possible, use `externalId` subject to the guidance above.

(Note: the Partner API's `idempotency_key` is a different mechanism — it's not part of a URL path, so the GUID-collision concern doesn't apply there. But "use a deterministic, non-random key" is still good hygiene.)

## This API carries other people's PII

Everything above treats FastBound records as data. They are also **personal information about an FFL's customers**, and a firearms purchase is about as sensitive as consumer records get. Design accordingly — this section is engineering practice, not legal advice; specific obligations belong with the licensee's counsel.

### What's actually in there

| Where | What it holds |
|---|---|
| **Individual contacts** | Legal name, residence address, phone, email — a real person, not a business |
| **Dispositions + items** | The association: *this named person, at this address, took possession of this firearm, by make/model/serial, on this date* |
| **Form 4473 PDFs** (`GET /Form4473s/Download/{id}`) | The most sensitive artifact in the system — date and place of birth, government ID type and number, citizenship, race and ethnicity, sometimes SSN, and the background-check response |
| **Multiple Sale Reports** (ATF 3310.4 PDFs) | Buyer identity plus multiple handguns, already flagged as a reportable pattern |
| **Bound book downloads** | Names and addresses across the account's entire transfer history, in one file |
| **Attachments** | Whatever was uploaded — commonly scans of driver's licenses and permits |

The association is the sensitive part, more than any single field. A leaked name and address is bad; a leaked name, address, and "owns a Glock 19, serial ABC123, purchased 3 March" is a targeting list. The FFL's customers never chose to trust *your* application, and they can't opt out of being in these records.

### Rules that follow

- **Don't mirror what you don't need.** FastBound is the system of record. A POS needs to *record* a disposition; it rarely needs a second copy of the buyer's full address, and it almost never needs the 4473 PDF. Every copy is a copy someone has to defend.
- **Store the ID, fetch on demand.** The [identifier guidance above](#identifiers-store-fastbounds-ids) is also the privacy-preferable pattern: hold FastBound's ID, pull the record when you actually need it, don't keep a shadow database of customers.
- **Request narrow scopes.** `form4473:read`, `attachment:read`, and `download:read` are the high-blast-radius ones. If your integration doesn't need them, not holding them is a stronger control than any policy you write about them.
- **Never log request or response bodies** on Contacts, Dispositions, Form4473s, MultipleSaleReports, Downloads, or Attachments. Error trackers (Sentry, Rollbar, Bugsnag) capture request bodies and breadcrumbs *by default* — turn that off for these routes, or scrub it. Don't interpolate customer names into log messages either.
- **Watch the query string.** `GET /Contacts?firstName=John&lastName=Smith` puts a customer's name in your access logs, your proxy's logs, your CDN's logs, and anything shipping those to a log aggregator. Search parameters are PII too; keep retention short and scrub what you can.
- **PDFs are toxic assets.** Stream 4473s, MSRs, and bound book downloads to the requesting user; don't spool them to disk, a temp directory, or an object-storage bucket "for caching." If you must persist one, encrypt at rest, gate it behind authorization, and set a deletion date.
- **Don't hand it to third parties.** No analytics, no session replay (a replay tool recording a screen with 4473 data on it is a breach in slow motion), no LLM or AI service, no offshore support tooling, no "just paste it in the Slack channel," no emailed CSV exports.
- **Don't seed test environments with production data.** Use synthetic contacts. The staging database is always the one with the weaker access controls.
- **Deleting your copy is not a compliance question.** The *licensee* must retain these records for the periods ATF requires — that's what FastBound is for. Your application has no such obligation, so retention on your side is pure liability: keep what you need, for as long as you need it, and no longer. Never advise deleting anything inside FastBound to satisfy a privacy request.
- **Aggregating across FFLs deserves a lawyer before it deserves a schema.** A multi-tenant product that pools purchaser data across accounts starts to resemble things that federal and state law treat carefully. If someone asks you to build that, say plainly that it's a legal question first.
- **If your copy leaks, it's the FFL's customers who are exposed** — and the licensee will hear about it from them. Tell the licensee immediately; breach-notification duties generally attach to whoever held the data.

### Working with an AI assistant on this data

The same rule as secrets: **real customer records don't go into a chat window.** Not a 4473, not a contact record with a real name and address, not a bound book export "so you can see the format." Transcripts persist, and the person in that record didn't consent to it.

When you need help with a specific record, redact first — swap the name, address, and ID numbers for obvious placeholders, keep the structure and the error. If an assistant asks you for a real record, that's the wrong request; give it a synthetic one. Assistants should ask for redacted or synthetic samples and should say something when a live record shows up in the conversation.

## Errors

All 4xx errors return:

```json
{
  "errors": [
    { "field": "items[0].serial", "message": "Serial is required." },
    { "field": "date", "message": "Date cannot be in the future." }
  ]
}
```

Always parse the array — a single failed write often returns multiple errors. Surface `field` and `message` together in your UI; the field paths use dot/bracket notation that maps to your request body.

Common status codes:

| Status | Meaning | Action |
|---|---|---|
| 200 | OK (read or no-content write) | Use the body |
| 201 | Created | Capture the returned ID and/or full record |
| 400 | Bad Request — validation failed | Parse `errors`, surface to the user, don't retry as-is |
| 401 | Unauthorized — token missing/expired | Refresh the access token and retry |
| 403 | Forbidden — token lacks the required OAuth scope (`error: "insufficient_scope"`), or under Basic Auth the `X-AuditUser` lacks permission / wrong account | For `insufficient_scope`, read the `scope` field in the body (and the `WWW-Authenticate` header) — it names what was needed; the user must re-authorize, refreshing won't help. Otherwise check the audit user and the `accountNum` in the path |
| 404 | Not Found — resource doesn't exist or you can't see it. **On a `Commit` call, read this as "probably already committed", not "wrong id"** — see below | Don't retry. On a Commit, reconcile before concluding anything |
| 409 | Conflict — typically a duplicate key (e.g., `externalId` already used) | Look up the existing record |
| 422 | Unprocessable Entity — business-rule violation | Same as 400 — read the errors |
| 429 | Too Many Requests — rate limited | Honor `Retry-After`, see rate-limit section |
| 5xx | Server error | Retry with exponential backoff |

## Committing consumes the pending record — so a repeated Commit returns 404

**A successful commit deletes the pending acquisition or disposition.** There is no
committed-but-still-addressable record left behind at that id, so a second
`POST /Acquisitions/{id}/Commit` or `POST /Dispositions/{id}/Commit` returns **404**.

This is the single most important thing to know before writing retry logic, because it
inverts the obvious reading of a 404. The natural conclusion — "not found, so my id is
wrong, so the write never happened" — is backwards: on a Commit, a 404 most often means the
commit already succeeded. An integrator who retries on that assumption, or who reports
failure to an operator who then re-enters the transfer by hand, ends up with a duplicate
federal record or a firearm the book says never moved.

The swagger does not say this. It declares 200, 204, 400 and 404 on the `{id}` Commit routes
and describes only the success path, so the behaviour is not derivable from the spec — which
is exactly why integrations get it wrong and why guessing here is worse than asking.

**Recovering from a commit whose response you never saw:** do not re-issue the Commit, and do
not infer state from whether the pending record still exists. Look for the *items* instead —
they survive, carrying the id of the acquisition or disposition that moved them:

```http
GET /{accountNum}/api/Items?dispositionId={dispositionId}&take=100&skip=0
```

Items come back, the commit landed. Nothing comes back, it did not. Note the asymmetry: the
bundled spec exposes `dispositionId` as a filter on `GET /Items` but no `acquisitionId`
equivalent, so confirm the acquisition-side lookup against the live API or FastBound support
before relying on it.

**A related consequence for idempotent creates.** If `GetByExternalId` returns an acquisition,
that record is still *pending* — a committed one would be gone. Treating a hit as "already
recorded, nothing to do" is backwards, and after a failure between `CreateAsPending` and
`Commit` it makes every retry report success while the firearms never enter the book.

## Pagination

Listing endpoints (`GET /Acquisitions`, `GET /Dispositions`, `GET /Contacts`, `GET /Items`) accept `take` and `skip` query parameters:

- `take` — page size (number of records to return). The server currently returns **at most 100** per page regardless of what you ask for. That ceiling is runtime behavior rather than a documented contract, so clamp to it client-side and don't build anything that depends on it never changing.
- `skip` — **a page index, not a record offset.** Page 0, then page 1, then page 2.

### `skip` is a page index — this one silently loses data

This is documented, not folklore: the swagger describes `skip` as *"how many **pages** to skip"* — identically on `/Items`, `/Acquisitions`, `/Contacts`, `/Dispositions`, and `/Dispositions/Only4473s`. It applies to every paginating endpoint, so there's no per-endpoint check to do.

**Advance `skip` by exactly 1 per page.** This is the single most expensive mistake in the API, because it fails without failing. On the wire, paging an inventory looks like this:

```http
GET /12345/api/Items?status=1&skip=0&take=100    ← records 1–100
GET /12345/api/Items?status=1&skip=1&take=100    ← records 101–200
GET /12345/api/Items?status=1&skip=2&take=100    ← records 201–300
```

And the mistake looks like this:

```http
GET /12345/api/Items?status=1&skip=0&take=100    ← records 1–100
GET /12345/api/Items?status=1&skip=100&take=100  ← records 10,001–10,100  ✗
GET /12345/api/Items?status=1&skip=200&take=100  ← records 20,001–20,100  ✗
```

The parameter is named `skip` and takes an integer, so `skip += take` is the reflex an experienced developer brings from every other API they've paged. With `take=100`, that asks for page 100 on the second request — jumping roughly 10,000 records. The response is a **200 with well-formed records in it**. Nothing raises, nothing warns, and the pull terminates early looking complete while having returned on the order of 1% of the account.

In a compliance system, that specific failure is worse than an outage. A short inventory pull becomes phantom "missing" firearms in a reconciliation, and the person holding that report is potentially an ATF inspector. An error would have been caught in testing; this won't be, unless the account under test is larger than one page.

If you're reviewing an existing FastBound integration, check this first — a `skip += take` or `skip = page * take` against any FastBound listing endpoint is a defect regardless of whether anyone has noticed symptoms.

### Knowing when to stop

**A short page ends the sequence.** Fetch until a page comes back with fewer than `take` records; that page is the last one. An empty page counts as short. Build on this rather than on arithmetic over a count — it can't drift out of step with what the server actually returned:

```python
page = 0
while True:
    r = get(f"/{acct}/api/Items?status=1&skip={page}&take=100").json()
    items = r["items"]          # lowercase, and named per resource — see below
    yield from items
    if len(items) < 100:
        break          # short page — done
    page += 1          # NOT page += 100
```

**The collection is named after its resource, not a generic key**, and every list response pairs it with a `records` count:

| Endpoint | Response shape |
|---|---|
| `GET /Items` | `{ "items": [...], "records": n }` |
| `GET /Contacts` | `{ "contacts": [...], "records": n }` |
| `GET /Acquisitions` | `{ "acquisitions": [...], "records": n }` |
| `GET /Dispositions` | `{ "dispositions": [...], "records": n }` |

All lowercase — which matters in any case-sensitive language, and is easy to get wrong if you learned the shape from a C# client where the deserializer forgave you.

Use `records` for a progress bar or a sanity check by all means, but keep the short-page test as the thing that actually terminates the loop. Guessing optimistically about termination has the same consequence as the `skip` bug: a pull that ends early and reports success.

For very large data sets, prefer narrow filters (date range, `externalId`, location, acquired-date bounds) over deep pagination. At 100 records per page against a per-account rate limit, a 10,000-item account is ~100 sequential requests and takes minutes — see "Don't queue requests" below, and expect to run pulls of that size as background work rather than inside a request/response cycle.

## Rate limiting

**Authoritative reference:** https://fastbound.help/en/articles/4409600-rate-limiting

FastBound's rate limit is **per-app-per-account**: each `(OAuth client, accountNum)` pair has its own independent budget. The same client integrated against ten accounts has ten independent budgets.

The runtime source of truth is three response headers, present on every API response:

| Header | Format | Meaning |
|---|---|---|
| `X-RateLimit-Limit` | integer | Maximum requests permitted in the current window |
| `X-RateLimit-Remaining` | integer | Requests remaining in the current window |
| `X-RateLimit-Reset` | integer (UTC epoch seconds) | Wall-clock time the window resets |

**Never hardcode a delay, RPS cap, or burst size** — read these headers and adapt. Hardcoded values either waste capacity when the actual limit is higher than your guess, or silently violate the limit when it's lower or changes.

The cheapest rate-limit headroom is the requests you don't make. Storing FastBound's IDs (see "Identifiers: store FastBound's IDs" above) removes the resolve-the-`externalId` round trip from every operation — meaningful once you're operating at volume.

### The pattern

After every API call:

1. **Read the three rate-limit headers** off the response.
2. **If the response is 429**: pause until `X-RateLimit-Reset`, then retry the same request. (If FastBound also sends `Retry-After`, honor it; otherwise compute `Reset - now`.)
3. **If `Remaining` is low** (say, < 10% of `Limit`) and you're inside a batch: voluntarily back off until `Reset`. A short sleep is cheaper than a 429 storm.
4. **Apply jitter on retries** so concurrent workers don't synchronize and thunder.

### Sketch (Python, requests)

```python
import time, requests

def call_with_throttle(method, url, **kwargs):
    while True:
        r = requests.request(method, url, **kwargs)

        if r.status_code == 429:
            reset = int(r.headers.get("X-RateLimit-Reset", "0"))
            wait = max(1, reset - int(time.time()))
            # Retry-After (if present) wins
            if "Retry-After" in r.headers:
                wait = int(r.headers["Retry-After"])
            time.sleep(wait)
            continue

        try:
            limit = int(r.headers["X-RateLimit-Limit"])
            remaining = int(r.headers["X-RateLimit-Remaining"])
            if remaining < max(1, limit // 10):
                reset = int(r.headers.get("X-RateLimit-Reset", "0"))
                wait = max(0, reset - int(time.time()))
                time.sleep(min(wait, 5))
        except (KeyError, ValueError):
            pass  # headers missing on some responses; skip the back-off

        return r
```

### What not to do

- Don't add `time.sleep(0.1)` between every call "to be safe."
- Don't put `RATE_LIMIT_RPS = 10` in a config file. Whatever you guess will be wrong.
- Don't catch 429 and retry immediately — you'll get another 429 and burn quota.
- Don't share rate-limit state across `(client, account)` pairs in a connection pool — each pair has its own budget.
- Don't queue API requests behind a long-running internal queue (see "Don't queue requests" below). Adaptive throttling is fine; deferred submission is not.

## FastBound's `X-FastBound-*` response headers

These headers communicate side effects and signals that aren't in the JSON body. **Watch for them on every disposition and contact write** — missing them is one of the most common integration defects, and the consequences range from "we created a duplicate contact" to "we missed a regulatory multiple-sale report and the FFL never knew."

**Authoritative reference:** https://fastbound.help/en/articles/6566241-http-headers-returned-in-api-calls

### Multiple Sale (commonly overlooked!)

Set on dispositions to non-licensee individuals when the disposition triggers ATF Form 3310.4 (Report of Multiple Sale or Other Disposition of Pistols and Revolvers — two or more handguns to the same buyer within five business days):

| Header | Format | Meaning |
|---|---|---|
| `X-FastBound-MultipleSale` | `true` | A multiple-sale report was generated by this disposition |
| `X-FastBound-MultipleSaleUrl` | absolute URL | Link to the report in the FastBound UI |

**Required action**: surface this in your UI. The licensee must review the report and either transmit it to ATF or dismiss it. If your integration silently swallows this header, the licensee may miss a required filing — that's a compliance failure, not a UX nicety.

A common, costly bug: integrators commit a disposition, see HTTP 200, and move on without inspecting headers. The disposition is in FastBound, but the multiple-sale notification never reaches the operator. Don't be that integration. Always parse these headers on every disposition commit response.

### Duplicate contact detection

When a contact create/update/merge detects an existing match:

| Header | Format | Meaning |
|---|---|---|
| `X-FastBound-ExistingContactId` | GUID | The existing contact's FastBound ID |
| `X-FastBound-ExistingContactUrl` | absolute URL | Link to the existing contact in the UI |

**Required action**: use the existing contact ID instead of creating a duplicate. Surface the URL to the operator if confirmation is needed.

### Auto-created acquisition (inter-FFL transfer)

When you commit a disposition whose contact is another FFL using FastBound (matched by the contact's email belonging to a FastBound user), FastBound automatically creates a *pending* acquisition in the receiving FFL's account so they don't have to re-enter it:

| Header | Format | Meaning |
|---|---|---|
| `X-FastBound-AcquisitionAccount` | account number (5–6 digits) | The receiving FFL's `accountNum` |
| `X-FastBound-AcquisitionId` | GUID | The pending acquisition's FastBound ID |
| `X-FastBound-AcquisitionUrl` | absolute URL | Deep link the recipient FFL can open to review and commit |

**Required action**: store the receiver's account number + acquisition ID for cross-system reconciliation. If you're orchestrating the transfer (e.g., a distributor shipping to a dealer who also uses FastBound), notify the recipient that a pending acquisition awaits their review.

### Why headers and not body fields?

These signals describe *side effects* in other accounts or systems (a multiple-sale report, a contact merge, an acquisition created in someone else's account). They don't fit the create/edit response model for the resource you wrote, so FastBound communicates them out-of-band via headers. The flip side is that any HTTP client middleware that strips or normalizes headers will silently break these features. Make sure your client logs and surfaces all `X-FastBound-*` headers.

## Don't queue requests

Counterintuitive but emphatic guidance from the FastBound team: **don't put a write queue between your application and the FastBound API**. The recommended posture is synchronous and fail-fast — when you POST, you wait for the response and decide right then.

The reasoning: FastBound's writes are tied to compliance state (the bound book, the 4473, the MSR). Queueing introduces:

- Retry logic and backoff strategy you have to maintain.
- Duplicate handling on top of stored-ID (or `externalId`) checks.
- Out-of-sync states between your local DB and FastBound's view of the world.
- Deferred error surfacing — by the time the queued write fails, the user is gone and the record is wrong.

Most validation errors (400/422) are *not transient* — they require human correction. Queue-and-retry will queue forever and never succeed. Better: fail synchronously, revert the local transaction, show the error to the user, and require manual remediation. Adaptive rate-limit throttling (back off on 429, retry once when the window resets) is fine and expected; that's bounded waiting on a known-recoverable signal. Long-running write queues are not.

## Manufacturing flag

Acquisitions and dispositions have an `isManufacturingAcquisition` / `isManufacturingDisposition` boolean. This matters because licensed manufacturers (Type 07 FFLs) record items they manufacture as a special kind of acquisition (and corresponding disposition into inventory) — distinct from a Type 01 dealer who only buys and resells. If the licensee isn't a manufacturer, leave these false.

## Type fields use string enums, not integers

Several fields (acquisition `type`, disposition `type`, contact `businessType`, contact `sotClass`) are exposed as **strings** in the API even though they're enums internally. Pass the string name (e.g., `"Purchase"`, `"Sale"`, `"Licensee"`), not an integer. The exact valid values per field are in the swagger schemas — `grep` for the field name.

## Date handling

Dates on records (acquisition `date`, disposition `date`, etc.) represent real-world acquisition/disposition dates and are stored as date or date-time strings. **A date in the future is a validation error** — you can't pre-record a future acquisition. Use the local business day in the licensee's time zone.

Webhook timestamps and `createUtc` / `updateUtc` audit fields are UTC ISO-8601.
