# Webhooks

Webhooks are FastBound's outbound notification mechanism. When something happens in an account (an acquisition is committed, a 4473 is completed, a multiple-sale report is transmitted), FastBound POSTs a JSON event to a URL you've registered, signed with HMAC-SHA256.

Reference articles:
- https://fastbound.help/en/articles/4782279-subscribing-to-events
- Payload schemas: bundled in `swagger/webhooks-swagger.json` (64 schemas across 23 event types)

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
- **Success** = HTTP 2xx response within 100 seconds. Anything else (timeout, 4xx, 5xx) is a failure and triggers retry.
- **Retry schedule**: up to **23 attempts over 3 days**:
  - Hour 1: 5 attempts (1m, 5m, 14m, 30m, 55m)
  - Hours 2–24: 10 attempts
  - Hours 24–48: 5 attempts
  - Hours 48–72: 3 attempts
- **Webhook disabled**: if a webhook fails 45 times (cumulative across deliveries), it's automatically disabled and the account owner is emailed. Until you re-enable it, no events will be delivered.

### Implications for handler design

- **Be idempotent.** Use the event payload's natural ID + event type as a dedupe key. The same event may be delivered more than once on flaky connections.
- **Respond fast.** If you do heavy work, accept the webhook (200), enqueue the work, process asynchronously. FastBound times out at 100 seconds; don't sit on a connection while processing.
- **Don't return 4xx for "I'll handle this later."** A 4xx is a signal that the delivery failed. Use 200 + internal queue.
- **Watch for the disabled state.** If 45 consecutive failures disable your webhook, you'll see a gap in events. Monitor for that gap (e.g., heartbeat events, periodic reconciliation), don't just assume the stream is healthy.

## Available event names

Event names use **dot notation**, all lowercase. The current list (call `GET /Webhooks/Events` for the authoritative current set):

**Acquisitions**
- `acquisition.created`
- `acquisition.committed`
- `acquisition.deleted`
- `acquisition.transfer`

**Dispositions**
- `disposition.committed`
- `disposition.deleted`
- `disposition.edited`
- `disposition.items.added`
- `disposition.items.edited`
- `disposition.items.removed`

**Contacts**
- `contact.created`
- `contact.edited`
- `contact.deleted`

**Items / Inventory**
- `item.imported`
- `item.edited`
- `item.deleted`
- `item.undeleted`
- `item.undisposed`
- `inventory.location.moved`
- `inventory.location.verified`
- `inventory.notfound`

**4473 / Multiple Sale**
- `form4473.completed`
- `multiplesalereport.transmitted`

The full payload schema for each event lives in `swagger/webhooks-swagger.json`. Each event's payload includes the resource (acquisition/disposition/contact/etc.) at the time of the event — you generally don't have to fetch additional data unless you need state that wasn't in the snapshot.

## What to subscribe to (recipes)

**POS / e-commerce sales**: `disposition.items.added`, `disposition.items.edited`, `disposition.items.removed`, `form4473.completed`, `disposition.committed`, `multiplesalereport.transmitted`.

**Inventory sync**: `acquisition.committed`, `disposition.committed`, `item.edited`, `item.undisposed`. **Match the event to the write path you actually use.** The atomic `Acquisitions/CreateAndCommit` endpoint skips the pending state by design, so it emits `acquisition.committed` only — never `acquisition.created`. If your integration uses `CreateAndCommit` (or could in the future) and you want to be notified of every new acquisition regardless of write path, subscribe to `acquisition.committed`. `acquisition.created` is the right pick only when you specifically want pending-state notifications and your writes go through the multi-step flow. Same logic for items added via `CreateAndCommit` (no separate `item.imported`); they show up only inside the parent acquisition event payload.

**Compliance dashboard**: `multiplesalereport.transmitted`, `form4473.completed`, `inventory.notfound`, `acquisition.committed`, `disposition.committed`.

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
- **Subscribing to `acquisition.created` while writing via `CreateAndCommit`** — easy to miss, but it's by design, not a defect. The atomic endpoint skips the pending state, so `acquisition.created` has nothing to fire on; only `acquisition.committed` does. If you want notifications for every new acquisition regardless of write path, subscribe to `acquisition.committed`. (Symptom of the mismatch: "we never get notifications for new acquisitions even though they're showing up in FastBound.")
