# Webhooks

Webhooks are FastBound's outbound notification mechanism. When something happens in an account (an acquisition is committed, a 4473 is completed, a multiple-sale report is transmitted), FastBound POSTs a JSON event to a URL you've registered, signed with HMAC-SHA256.

Reference articles:
- https://fastbound.help/en/articles/4782279-subscribing-to-events
- Payload schemas: bundled in `swagger/webhooks-swagger.json`

## Subscribing — POST `/Webhooks`

Subscriptions are managed via the API:

| Operation | Endpoint |
|---|---|
| Create subscription | `POST /{accountNum}/api/Webhooks` |
| Get a subscription by name | `GET /{accountNum}/api/Webhooks/{name}` |
| Update a subscription by name | `PUT /{accountNum}/api/Webhooks/{name}` |
| Delete a subscription by name | `DELETE /{accountNum}/api/Webhooks/{name}` |
| List available event names | `GET /{accountNum}/api/Webhooks/Events` |

A subscription is identified by a **name** that's unique within the account. You also pick a destination URL (HTTPS, unique within the account), an optional description, and the list of events you want.

### Create payload

```json
POST /{accountNum}/api/Webhooks
Content-Type: application/json
# OAuth: no X-AuditUser needed. Basic Auth: include X-AuditUser: <email>.

{
  "name": "pos-sync",
  "url": "https://app.example.com/webhooks/fastbound",
  "description": "Sync FB events to POS",
  "events": [
    "disposition.committed",
    "disposition.items.added",
    "disposition.items.edited",
    "disposition.items.removed",
    "form4473.completed",
    "acquisition.committed"
  ]
}
```

Constraints:
- `name`: max 50 chars; `[a-zA-Z0-9_-]`; unique within account.
- `url`: max 255 chars; HTTPS; unique within account.
- `description`: max 100 chars.
- `events`: at least one; must be valid event names (see `GET /Webhooks/Events`).

### The signing secret — capture it on creation

The create response includes:

```json
{
  "signingSecret": "0123abcd... (random 256-bit hex)",
  "name": "pos-sync",
  "url": "...",
  "description": "...",
  "disabled": false,
  "subscriptionSource": "...",
  "events": [...],
  "createUtc": "...",
  "updateUtc": "..."
}
```

> **The `signingSecret` is returned only on creation. There is no API to retrieve it later.** If you don't capture it from the POST response and store it securely, you cannot verify webhook signatures and you'll have to delete the subscription and create a new one to get a fresh secret.

Store the `signingSecret` with the same hygiene as your OAuth `client_secret` (secrets manager, OS keychain — never `.env` or source control). See `authentication.md` for storage tier guidance.

## Verifying webhook signatures

Every delivery FastBound makes carries the `X-FastBound-Signature` header, formatted as:

```
X-FastBound-Signature: t=1714752000,v1=<64 hex chars>
```

Where:
- `t` — Unix timestamp (seconds) when FastBound created the signature.
- `v1` — HMAC-SHA256 hex digest (256-bit, hex-encoded = 64 characters).

### The verification algorithm

1. **Read the raw request body bytes** as received. Do *not* parse the JSON, re-serialize it, or even decode it to a string with charset assumptions. The signature is over the bytes on the wire — any reformatting changes the hash.
2. Extract `t` and `v1` from the `X-FastBound-Signature` header.
3. Form the **signed payload**: the timestamp string, an ASCII period (`.`, 0x2E), then the raw body bytes.
4. Compute `HMAC-SHA256(signing_secret, signed_payload)` and hex-encode it.
5. Compare in constant time to `v1`. Reject on mismatch.
6. (Recommended) Reject deliveries whose timestamp is too old (e.g., > 5 minutes). Replay protection.

### Worked example — Python (Flask)

```python
import hmac, hashlib, time, os
from flask import Flask, request, abort

app = Flask(__name__)
SIGNING_SECRET = os.environ["FB_WEBHOOK_SECRET"].encode()  # bytes
TOLERANCE_SECONDS = 300

def verify(body_bytes: bytes, signature_header: str) -> bool:
    try:
        parts = dict(p.split("=", 1) for p in signature_header.split(","))
        t = parts["t"]
        v1 = parts["v1"]
    except Exception:
        return False

    # Reject stale deliveries (replay protection)
    if abs(time.time() - int(t)) > TOLERANCE_SECONDS:
        return False

    signed_payload = t.encode() + b"." + body_bytes
    expected = hmac.new(SIGNING_SECRET, signed_payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, v1)

@app.post("/webhooks/fastbound")
def receive():
    sig = request.headers.get("X-FastBound-Signature", "")
    raw = request.get_data()  # raw bytes, NOT request.json
    if not verify(raw, sig):
        abort(401)

    event = request.get_json()
    handle_event(event)
    return "", 200
```

### Worked example — Node.js (Express)

```javascript
const crypto = require("crypto");
const express = require("express");
const app = express();
const SIGNING_SECRET = process.env.FB_WEBHOOK_SECRET;
const TOLERANCE_SECONDS = 300;

function verify(rawBody, signatureHeader) {
  const parts = Object.fromEntries(
    signatureHeader.split(",").map((p) => p.split("=", 2))
  );
  const t = parts.t;
  const v1 = parts.v1;
  if (!t || !v1) return false;

  if (Math.abs(Date.now() / 1000 - parseInt(t, 10)) > TOLERANCE_SECONDS) return false;

  const signedPayload = Buffer.concat([Buffer.from(t), Buffer.from("."), rawBody]);
  const expected = crypto
    .createHmac("sha256", SIGNING_SECRET)
    .update(signedPayload)
    .digest("hex");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(v1));
}

// Capture the raw body — do NOT use express.json() before this.
app.post(
  "/webhooks/fastbound",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const sig = req.headers["x-fastbound-signature"];
    if (!verify(req.body, sig)) return res.sendStatus(401);
    const event = JSON.parse(req.body.toString("utf8"));
    handle(event);
    res.sendStatus(200);
  }
);
```

### The single most common mistake

**Verifying the signature on parsed-then-re-serialized JSON.** Any framework that parses the body into an object and hands it to your handler has *destroyed the original byte stream* — key order, whitespace, number formatting may all differ. Re-serializing produces a different hash. Always capture the raw body bytes at the framework boundary and verify against those.

In Express: use `express.raw()` for the webhook route, not `express.json()`.
In Flask: `request.get_data()`, not `request.json`.
In ASP.NET Core: `EnableBuffering()` + read the request body stream.
In Go (net/http): read `r.Body` once into a `[]byte`.

## Delivery semantics

- **Initial delivery** typically within seconds of the event.
- **Identifying FastBound's requests**: outbound deliveries send a `User-Agent` beginning with `FastBound-Webhook/`, with the originating account and a link to FastBound's webhook docs in the comment section. If your endpoint sits behind Cloudflare, a WAF, or any bot-management layer, allowlist that User-Agent — a bot filter silently dropping deliveries looks exactly like a broken integration, and eventually disables the webhook (below). Verify the exact string against a real delivery; don't pattern-match on it more tightly than the `FastBound-Webhook/` prefix, and never treat it as authentication — the signature is what proves a delivery came from FastBound.
- **Success** = HTTP 2xx response within 100 seconds. Anything else (timeout, 4xx, 5xx) is a failure and triggers retry.
- **Retry schedule**: up to **23 attempts over 3 days**:
  - Hour 1: 5 attempts (1m, 5m, 14m, 30m, 55m)
  - Hours 2–24: 10 attempts
  - Hours 24–48: 5 attempts
  - Hours 48–72: 3 attempts
- **Webhook disabled**: after a run of **consecutive** failed delivery attempts, the subscription is automatically disabled and the account owner is emailed. Until someone re-enables it, no events are delivered at all. The counter tracks *consecutive* failures, so a delivery that succeeds resets it — an endpoint that fails intermittently won't accumulate its way to a shutdown, but one that's down long enough will.

### Implications for handler design

- **Be idempotent.** Use the event payload's natural ID + event type as a dedupe key. The same event may be delivered more than once on flaky connections.
- **Respond fast.** If you do heavy work, accept the webhook (200), enqueue the work, process asynchronously. FastBound times out at 100 seconds; don't sit on a connection while processing.
- **Don't return 4xx for "I'll handle this later."** A 4xx is a signal that the delivery failed. Use 200 + internal queue.
- **Watch for the disabled state.** A long enough run of consecutive failures disables the subscription, and you'll see a silent gap in events — re-enabling is a human action in the FastBound UI, not something your code can do. Monitor for the gap (heartbeat events, periodic reconciliation); don't assume the stream is healthy just because nothing errored on your side.
- **Payloads carry customer PII.** Disposition and 4473 events name real people. Your endpoint is an inbound PII pipe: TLS only, don't log raw bodies (a webhook receiver dumping payloads to stdout is the most common way this data ends up in a log aggregator), and don't forward them to third-party services. See "This API carries other people's PII" in `references/common-patterns.md`.

## Available event names

Event names use **dot notation**, all lowercase. Call `GET /Webhooks/Events` for the authoritative current set — that endpoint, not this page, is what your account actually accepts.

**Read the "fires when" column before you subscribe.** Most webhook disappointments aren't bugs — they're an event that fires on a narrower trigger than the name suggests. Several fire only from UI actions and never from the API, and a few things you'd expect to have events don't have them at all.

### Events that don't exist

Worth knowing up front, because integrations get designed around them:

- **No `disposition.created`.** A new pending disposition with no items produces *no webhook at all*. The first signal is `disposition.items.added`, once items land on it.
- **No `item.created`.** New items reach your inventory via `acquisition.created`, `acquisition.committed`, `acquisition.transfer`, or `item.imported`.
- **No "committed acquisition deleted."** `acquisition.deleted` fires only for a *pending* acquisition. Committing consumes the pending record rather than deleting it in the sense this event means — the firearms live on as inventory items, and the id stops resolving.

### Acquisitions

| Event | Fires when |
|---|---|
| `acquisition.created` | An acquisition enters the **pending** state. **Skipped entirely by `CreateAndCommit`**, which never has a pending state. |
| `acquisition.committed` | An acquisition is committed — **either** write path, pending-then-committed or `CreateAndCommit`. For `CreateAndCommit` users this is the only acquisition event for that record. |
| `acquisition.deleted` | A **pending** acquisition is deleted, with its contact and items. Not fired by committing one, and not fired by deleting individual items off a pending acquisition. |
| `acquisition.transfer` | An acquisition arrives in your account by transfer from another FFL. **Receive side only**, and always paired with `acquisition.created` for the same record — the `created` event is sent first. Only for Partner API transfers and dispositions between two FastBound accounts — an ordinary disposition to a non-FastBound contact produces no transfer event anywhere. Payload carries **both** sides — `transferorContact` and `transfereeContact` — plus `items`, `purchaseOrderNumber`, `invoiceNumber` and `shipmentTrackingNumber`, so a receiver generally needs no follow-up fetch. |

### Dispositions

| Event | Fires when |
|---|---|
| `disposition.committed` | A disposition is committed, either write path. Theft/loss/destroyed dispositions commit immediately on creation and fire this. |
| `disposition.deleted` | A **pending** disposition is deleted (committed ones can't be). Also fires as a side effect of certain Form 4473 actions — cancellation, denial, termination, delayed cleanup, detaching or swapping a 4473 disposition. **Not** fired by "Undispose Items" on a theft/loss/destroyed disposition; that fires `item.undisposed` per item and leaves the disposition record in the book. |
| `disposition.edited` | The disposition changes — and unusually, this covers **both pending and committed** dispositions. Also fires when an attached contact's details change, when an attached Form 4473 moves through its lifecycle (proceed, denied, delayed, terminated, attach/detach), and when the disposition is locked or unlocked. |
| `disposition.items.added` | Items are added to an existing **pending** disposition — by serial, by external ID, or through the UI. **The `CreateAndCommit` disposition endpoint does not fire this**; only `disposition.committed`. |
| `disposition.items.edited` | Items on a **pending** disposition change (price, pawn-redemption flag during 4473 create/edit). Committed disposition items are immutable, so this never fires for them. |
| `disposition.items.removed` | Items are removed from a **pending** disposition and return to available inventory. Not fired by "Undispose Items" — that's `item.undisposed`. |

### Contacts

| Event | Fires when |
|---|---|
| `contact.created` | A contact is created in the UI or via the Contacts API, **or** created inline as part of a single-call acquisition or disposition. **Does not fire** for contacts created as a silent side effect of a CSV item import or an inbound Partner API transfer — those contacts appear with no event. |
| `contact.edited` | **Every save** of the contact form, including saves that changed nothing. Archiving and unarchiving arrive here (with `IsArchived` updated), *not* as a delete. On a merge, the **surviving** contact fires this. |
| `contact.deleted` | A true deletion. Archiving is an edit, not a delete. On a merge, the **discarded** contact fires this. |

### Items and inventory

| Event | Fires when |
|---|---|
| `item.imported` | A CSV import completes — **once per import file, with all items in one payload**, not once per item. Items created any other way don't fire it. |
| `item.edited` | An item changes (manufacturer, serial, location, pricing…). Once per item, including inside bulk-change operations, and including changes to the item's acquisition contact. **Fires whether or not a field actually changed.** |
| `item.deleted` | A **direct, single-item** deletion. Items removed as a cascade of a pending-acquisition deletion do **not** fire this — only `acquisition.deleted` does. |
| `item.undeleted` | A deleted item is restored. |
| `item.undisposed` | A disposed item returns to available, from the explicit Undispose action (UI, API, or bulk change) — once per item, including reversals of theft/loss/destroyed dispositions. **Does not fire** as a side effect of editing or deleting a disposition. |
| `inventory.location.moved` | Items' physical location changes: once per bulk move with all selected items in one payload, and on API `BulkVerify` when location updates are enabled. **Does not fire** when you change location by editing the item — that's `item.edited`. Sent even if the new location is the same as the old. |
| `inventory.location.verified` | Locations are verified during a cycle count — one payload with all verified items. Fires from the UI "Verify Location" action and from API `BulkVerify` for **exactly matching** items. With `RollbackPartial=true`, if any serial fails to match, nothing is saved and no webhook fires. |
| `inventory.notfound` | Items are marked not-found — **UI Cycle Count "Not in this location" only.** See the warning below. |

### 4473 and Multiple Sale

| Event | Fires when |
|---|---|
| `form4473.completed` | A Form 4473 reaches a final seller-completed status — the seller submits completion of an **e-signed** 4473 (Proceed, Denied, Cancelled…), or terminates/voids one. **Does not fire** on buyer-side actions (Section B certification, recertifications), on identification verification, on terminations of **wet-signed/paper** 4473s, or for the retired 2016 (R1) revision. Payload carries buyer info, response codes, and `apiDownloadUrl` — see the note below on what that URL is. |
| `multiplesalereport.transmitted` | An MSR is transmitted — **exactly once per report**, when the seller clicks Transmit and it's successfully queued for the ATF. There's no automated transmission path, and CLEO/CC-list deliveries aren't separate events. If the ATF email fails to queue, the transmission aborts and **no webhook fires**. Payload carries the report, its items, and `apiDownloadUrl` — see the note below. |

> ### ⚠️ Several events are UI-only — the API won't trigger them
>
> If your integration writes exclusively through the API, these will never arrive no matter what you subscribe to:
>
> - **`inventory.notfound`** — only the UI Cycle Count action fires it. `BulkVerify` returns non-matching serials **in the response body** with no webhook at all. If you need not-found signals from an API-driven count, read them off the `BulkVerify` response; don't wait for an event.
> - **`contact.created`** — silent for contacts created by CSV import or inbound Partner API transfer.
> - **`item.imported`** — CSV import only; API-created items fire acquisition events instead.
>
> The inverse trap is just as common: subscribing to `acquisition.created` or `disposition.items.added` while writing through `CreateAndCommit`, which skips the pending state those events describe.

The full payload schema for each event lives in `swagger/webhooks-swagger.json`. Each event's payload includes the resource (acquisition/disposition/contact/etc.) at the time of the event — you generally don't have to fetch additional data unless you need state that wasn't in the snapshot.

### `apiDownloadUrl` is an authenticated endpoint, not a pre-signed link

`form4473.completed` and `multiplesalereport.transmitted` each carry an `apiDownloadUrl`. It is a URL to the regular Download API route for that record — **it is not a signed or tokenized storage link, and it carries no access token of its own.** Fetching it means an authenticated API call, exactly like any other: `Authorization: Bearer <access_token>` on OAuth (with `form4473:read` or `msr:read`), or your Basic Auth credentials plus `X-AuditUser` on legacy. A plain unauthenticated GET gets you a 401, not a PDF. The endpoint streams the file back in the response body.

The practical consequences, in order of how often they bite:

- **Don't hand the URL to a browser, an email, or anything else that can't authenticate.** It won't work, and building a customer-facing "download your 4473" link out of it will fail in front of the licensee.
- **Don't treat it as short-lived.** There's no embedded expiry to race, so a queued handler can fetch it later without a countdown. Retry on your own schedule.
- **Do treat what comes back as toxic.** A 4473 PDF is the most sensitive artifact in the system and an MSR isn't far behind. Stream them to whoever asked; don't spool them to disk or a bucket "for caching." See "This API carries other people's PII" in `common-patterns.md`.

### Payloads identify records by FastBound's `id`, not your `externalId`

Plan your handler around this before you write it. **The parent record's `externalId` is not in most payloads.** Every acquisition event, every disposition event, `form4473.completed`, `multiplesalereport.transmitted`, the three inventory events, and `item.imported` arrive with the record's FastBound `id` and no `externalId`. Only `contact.created`/`edited`/`deleted` and `item.deleted`/`edited`/`undeleted`/`undisposed` carry one — there the payload *is* the entity. Contacts and items nested inside an acquisition or disposition payload do carry theirs; the parent does not.

The consequence is concrete. A handler that keys its own records by `externalId` receives `acquisition.committed`, finds no `externalId`, and has to call back to FastBound to identify the record FastBound just handed it — one extra round trip per event, forever, against the same rate limits. A handler that stored FastBound's `id` at write time matches on the payload in hand and calls nothing.

This is the strongest practical argument in the identifier guidance in `common-patterns.md`: store FastBound's IDs. `externalId` is a secondary index for the cases where you genuinely can't, and webhook consumption is not one of those cases.

## What to subscribe to (recipes)

**POS / e-commerce sales**: `disposition.items.added`, `disposition.items.edited`, `disposition.items.removed`, `form4473.completed`, `disposition.committed`, `multiplesalereport.transmitted`.

**Inventory sync**: `acquisition.committed`, `disposition.committed`, `item.edited`, `item.undisposed`. **Match the event to the write path you actually use.** The atomic `Acquisitions/CreateAndCommit` endpoint skips the pending state by design, so it emits `acquisition.committed` only — never `acquisition.created`. If your integration uses `CreateAndCommit` (or could in the future) and you want to be notified of every new acquisition regardless of write path, subscribe to `acquisition.committed`. `acquisition.created` is the right pick only when you specifically want pending-state notifications and your writes go through the multi-step flow. Same logic for items added via `CreateAndCommit` (no separate `item.imported`); they show up only inside the parent acquisition event payload.

**Inbound transfers you didn't create**: `acquisition.transfer` (+ `acquisition.created` for the pairing, if you want the pending-state detail).

Firearms can land in a FastBound account with no write from you — a distributor pushing through the Partner API, or another FastBound FFL committing a disposition to that account. The pending acquisition appears with its transferor contact and items already populated, and your system never saw it happen. Subscribe to `acquisition.transfer` and upsert on FastBound's acquisition and item IDs; because the two events always fire together for the same record, dedupe on that acquisition ID or you'll ingest it twice.

**Don't try to cover this case with `acquisition.committed` alone.** That doesn't fire until someone reviews and commits, which can be days after the box arrives — a window in which your system shows nothing while the firearms are physically on the dock. And note the companion gap: the transferor contact created by an inbound Partner API transfer fires **no** `contact.created`, so a handler that builds its contact table purely from contact events will have an acquisition pointing at a contact it's never heard of. Read the contact off the acquisition payload.

Leave the acquisition pending and don't delete it — commit is the receiving licensee's regulatory act, and deleting it to reconcile against your own records is a pattern FastBound specifically doesn't favor. See "Acquisitions you didn't create" in `acquisitions.md`.

**Compliance dashboard**: `multiplesalereport.transmitted`, `form4473.completed`, `acquisition.committed`, `disposition.committed` — plus `inventory.notfound` **only if the FFL runs cycle counts in the FastBound UI**. If counts are driven through your integration's `BulkVerify` calls, that event never fires; take the not-found serials from the `BulkVerify` response body instead.

**Manufacturing pipeline**: `acquisition.committed` (for receiving components and finished firearms), `disposition.committed` (for shipments and manufacturing dispositions consuming components).

**Don't blanket-subscribe to everything.** It's tempting; resist. Each event you subscribe to is one your handler must process correctly and idempotently. Subscribe narrowly, expand as needs grow.

## Common pitfalls

- **Losing the `signingSecret`** — captured only on POST response, never retrievable. Store immediately.
- **Verifying signature on parsed JSON** — always verify on raw body bytes.
- **Not enforcing timestamp tolerance** — leaves you open to replay.
- **Returning 4xx for application-level reasons** — looks like a failure to FastBound; triggers retries; eventually disables the webhook.
- **Slow handlers** — 100s timeout. Accept fast, process async.
- **Not monitoring for disabled webhooks** — silent gap in event flow.
- **Subscribing too broadly** — every event is code you have to maintain.
- **Confusing event names with schema names** — the swagger schemas are PascalCase (`AcquisitionCommittedEvent`); the actual event-name strings used in subscriptions and deliveries are dot-case (`acquisition.committed`).
- **Subscribing to `acquisition.created` while writing via `CreateAndCommit`** — easy to miss, but it's by design, not a defect. The atomic endpoint skips the pending state, so `acquisition.created` has nothing to fire on; only `acquisition.committed` does. If you want notifications for every new acquisition regardless of write path, subscribe to `acquisition.committed`. (Symptom of the mismatch: "we never get notifications for new acquisitions even though they're showing up in FastBound.") Correct the one subscription — don't add the neighbouring events "to be safe" as well. Each one is a handler you then have to process correctly and idempotently, forever. The same applies to `disposition.items.added` against the `CreateAndCommit` disposition endpoint.
- **GETting `apiDownloadUrl` without credentials** — it's the Download API route, not a signed link. Authenticate the call the way you authenticate every other one.
- **Expecting `externalId` in the payload** — acquisition and disposition events don't carry the parent record's `externalId`, only its FastBound `id`. A handler keyed on `externalId` pays a lookup round trip on every single event. Store FastBound's IDs at write time instead.
- **Syncing inbound transfers off `acquisition.committed`** — a transfer-created acquisition arrives pending and stays pending until the FFL commits it, so your first signal is `acquisition.transfer`, not the commit. Symptom: stock visible in FastBound that your system doesn't know exists.
- **Waiting on a UI-only event from an API-driven workflow** — `inventory.notfound` in particular. `BulkVerify` reports non-matching serials in its response body and fires nothing. See the warning in the event catalogue.
- **Treating `contact.edited` as "a field changed"** — it fires on every save, including no-op saves, and it's how archiving arrives. Diff against your own copy if you need real change detection.
- **Expecting `item.deleted` when a pending acquisition is deleted** — the item removals cascade silently; only `acquisition.deleted` fires.
- **Expecting an event when a pending disposition is created** — there isn't one. Nothing fires until items are added.
- **Assuming `form4473.completed` covers paper 4473s** — it's e-signed only, seller-side only.
