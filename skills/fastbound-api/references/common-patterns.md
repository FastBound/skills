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

`accountNum` is assigned by FastBound to the licensee. Treat it as configuration, not as something to discover at runtime.

## The audit user header — depends on auth mechanism

Whether you need to send `X-AuditUser` depends on which auth mechanism you're using:

### OAuth — `X-AuditUser` is NOT required

The OAuth access token already encodes the identity of the user who authorized the integration. FastBound uses that user as the audit identity automatically. Don't send `X-AuditUser` on OAuth writes; if you do (for example during a Basic Auth → OAuth migration where you haven't removed the header yet), it's silently accepted.

For server-to-server OAuth integrations, enroll a service-account FastBound user (e.g., `integration@example.com`) in the customer's account and authorize the OAuth flow as *that* user. Everything the integration writes is then attributed to that service account in the audit log.

### Basic Auth (legacy) — `X-AuditUser` IS required on every write

```
X-AuditUser: jdoe@example.com
```

The API Key identifies the *integration*, not the human. To preserve the audit trail, every write must include the email of a FastBound user. The email must:

1. Belong to an existing FastBound user.
2. Have access to the account specified in `accountNum`.
3. Have a permission level appropriate for the operation.

If any of these fail, the request fails. Treat `X-AuditUser` as a required field on every POST/PUT/DELETE under Basic Auth.

## Idempotency via `externalId`

Most resources accept an `externalId` field — your system's ID for the same record. FastBound stores it and you can look up the record by it, e.g.:

- `GET /{accountNum}/api/Acquisitions/GetByExternalId/{acquisitionExternalId}`
- `GET /{accountNum}/api/Dispositions/GetByExternalId/{externalId}`
- Disposition item operations support `AddByExternalId` and `RemoveByExternalId`

Use this to make creates idempotent: before POSTing, check whether a record with that `externalId` already exists. This protects against duplicate writes from network retries, partial outages, or replays.

Resources that accept `externalId`: `Acquisitions`, `AcquisitionItems`, `Dispositions`, `DispositionItems`, `Contacts`, `Items`. The `externalId` doesn't have to be globally unique — only unique within its resource type within the account.

### Don't make your externalId look like a GUID

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

(Note: the Partner API's `idempotency_key` is a different mechanism — it's not part of a URL path, so the GUID-collision concern doesn't apply there. But "use a deterministic, non-random key" is still good hygiene.)

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
| 403 | Forbidden — `X-AuditUser` lacks permission, or wrong account | Check the audit user and the `accountNum` in the path |
| 404 | Not Found — resource doesn't exist or you can't see it | Don't retry; verify the ID |
| 409 | Conflict — typically a duplicate key (e.g., `externalId` already used) | Look up the existing record |
| 422 | Unprocessable Entity — business-rule violation | Same as 400 — read the errors |
| 429 | Too Many Requests — rate limited | Honor `Retry-After`, see rate-limit section |
| 5xx | Server error | Retry with exponential backoff |

## Pagination

Listing endpoints (`GET /Acquisitions`, `GET /Dispositions`, `GET /Contacts`, `GET /Items`) accept `take` and `skip` query parameters:

- `take` — page size (number of records to return).
- `skip` — number of *records* (or *pages*, depending on endpoint — verify in the swagger for the specific resource you're paging) to skip from the start.

The list responses typically include the total count alongside the records. Page through by incrementing `skip` until you've consumed the total. For very large data sets, prefer narrow filters (date range, externalId, etc.) over deep pagination.

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
- Duplicate handling on top of `externalId` checks.
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
