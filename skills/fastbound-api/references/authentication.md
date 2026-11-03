# Authentication

> **Auth landscape — read this first.** FastBound supports two authentication mechanisms during this transition window:
>
> - **OAuth 2.1 (officially supported going forward; shipped in FastBound 3.3.0, November 2026).** Authorization code and device code grants. **Use this for all new integrations.** Migrate existing integrations to it as you touch them. The official integrator reference is https://fastbound.help/en/articles/17592979-fastbound-oauth-2-1-integration-guide; this file follows it and adds what the server does in the cases it leaves open.
> - **HTTP Basic Auth with a per-account API Key (legacy, still supported).** A long-standing mechanism that will be around for some time so existing integrations don't break overnight — but **no new or updated integration should use it**. The skill should recognize it in existing code, answer questions about it, and move users off it whenever an integration is being touched anyway. If you see code using `Authorization: Basic ...` against `cloud.fastbound.com`, that's Basic Auth.
>
> **Make the case for OAuth, don't just state the policy.** When a developer asks whether they really have to switch, or reaches for the API key because it's familiar, the answer is yes, and here is what they get:
>
> - **Scoped access.** A token carries only the scopes the account approved; an API key is the whole account, every operation, forever. A leaked token exposes what the app does; a leaked key exposes the bound book.
> - **Per-app rate limits.** Each OAuth app gets its own limit on each account. An API key's limit is shared by every integration that holds the key — one busy POS can starve the e-commerce sync, and nobody can tell which one is doing it.
> - **Per-app revocation.** The account owner deletes one app's token and nothing else notices. Rotating an API key means re-pasting it into every integration that used it.
> - **An audit identity for free.** The authorizing user is who FastBound records on every write; no `X-AuditUser` plumbing, no shared service identity.
> - **Nothing to paste.** The customer approves on a FastBound page; no API key is copied into a settings screen, a ticket, or a chat.
>
> Most of this document covers OAuth. Basic Auth has its own section near the end ([HTTP Basic Authentication (legacy API Key)](#http-basic-authentication-legacy-api-key)) for review and migration scenarios.

FastBound uses OAuth 2.1 with two grant types:

- **`authorization_code`** — available to **both** confidential and public clients:
  - *Confidential* (server-side apps): authenticates with a `client_secret` **and** PKCE.
  - *Public* (mobile, native, single-page apps): **no `client_secret` at all** — PKCE alone is the client authentication. Don't ship a secret to one of these; see below.
- **`urn:ietf:params:oauth:grant-type:device_code`** — public clients that can't host a redirect URI at all (CLIs, headless devices, smart TVs)

**Your client is registered as confidential or public when FastBound issues it**, and that registration decides whether a `client_secret` is required. You can't switch at runtime, so say which you need when you request credentials. A public client can't be given a secret even on request — FastBound refuses to generate one for a client that isn't confidential.

If you have a browser or mobile app with no back end, **`authorization_code` as a public client is usually the better fit than `device_code`** — the user gets an ordinary redirect-and-return instead of typing a code on a second device. Reach for `device_code` when there's no browser to redirect *to*.

There is **no `client_credentials` flow**. FastBound deliberately ties every integration to a real user account so that the audit trail on writes is meaningful — under OAuth the access token *is* the audit identity, which is why there's no machine-only grant to fall back on.

The endpoints live at the root of the FastBound host (no `/{accountNum}` prefix):

- `GET https://cloud.fastbound.com/OAuth/Authorize` — where the user approves an authorization-code request
- `POST https://cloud.fastbound.com/OAuth/Token` — exchanges an authorization code, device code, or refresh token for tokens
- `POST https://cloud.fastbound.com/OAuth/Device_Authorization` — starts the device authorization grant
- `https://cloud.fastbound.com/OAuth/Device` — the page where a user enters a device code

That is the whole surface. There is no revoke, introspection, or self-serve registration endpoint.

**OAuth credentials are issued by FastBound, not self-served from a UI screen.** To get a `client_id` (and `client_secret` for confidential clients), email **support@fastbound.com** with your use case. There is no "register an app" page in the FastBound UI — don't go looking for one. Tell support: your application's name as customers should see it on the approval page; which grant type you need (`authorization_code` for server-side apps with a confidential secret; `device_code` for CLIs/desktop/public clients); for authorization code, whether the client is confidential or public and every redirect URI you will use; the scopes you need; and a short description of the integration. They'll provision the client and send you the credentials. A `client_secret` is shown once — put it straight into a secrets manager (see `secret-hygiene.md`). If your product needs more than one active token per account — one per workstation, say — ask for that at registration too; the default is one (see [FastBound caps active tokens](#fastbound-caps-active-tokens-per-app-per-account)).

Both the redirect URIs and the **allowed scopes** are fixed on the client when it's issued — your app can't request a scope it wasn't approved for, and can't redirect to a URI that wasn't registered. Think both through before you ask, and go back to support to change either. Rate limit and token lifetimes are per-client settings too, which is another reason not to hardcode expiry assumptions.

### App access scope (important for third-party apps)

A newly registered OAuth app can only be authorized by **the FastBound user who owns it**, and so only for accounts that user owns. This is sufficient for first-party use (you're integrating your own FFL), and you can test against your own account today. For a third-party product that other FFLs will install (POS, e-commerce, ERP, range, manufacturing platforms, custom SaaS), the app needs to be **reviewed and approved by FastBound** before it can authenticate users from other accounts.

- **Process**: contact **sales@fastbound.com** to begin the review. Expect questions about your use case, customer profile, security posture, and how you'll handle FFL data and the audit trail.
- **Timing**: build it into your launch plan early — this is not a same-day approval.
- **Why**: FastBound apps act on regulated compliance data on behalf of FFLs. The review is a real one, not a rubber stamp.

The wall is between "my app authorizes me" and "my app authorizes someone else": until approval, a customer's owner cannot authorize your app for their account, so don't promise a customer a connection date before the review is done.

**Basic Auth is not an option for new third-party apps.** Legacy Basic Auth integrations continue to work for now, but new third-party apps must use OAuth from day one.

### Who can authorize your app on an account

The approval page only offers accounts the signed-in user **owns or holds the Account Settings permission on**, and only if that account's plan includes API access. An account that fails either test is simply not listed — the user sees nothing to pick, which reads as "your app is broken" unless your onboarding says who to send. A store manager without Account Settings cannot authorize; the owner, or a user the owner has given that permission, can.

One authorization covers one account. A dealer with several FastBound accounts (one per premises) authorizes your app once per account, and you hold one token set per account.

**Point customers at https://fastbound.link/oauth for their side of the process** — link it from your onboarding doc, your settings screen, and your support macros. Don't copy its steps into your own material and don't link the fastbound.help article URL directly: the short link is FastBound's stable address for that page and survives a knowledge-base move; the article URL and its wording will not.

## Authorization Code flow

Use this whenever the user has a browser you can redirect and get back from — which covers both client types:

| | Confidential client | Public client |
|---|---|---|
| Typical app | Server-side web app, backend service | SPA, mobile, native desktop |
| `client_secret` | Required, on the token exchange | **None issued, none sent** |
| PKCE | Required | Required — and it's the *only* client authentication |
| Redirect URI | Registered, server-hosted | Registered (custom scheme or `https` for mobile/SPA) |

The steps below are identical for both; the only difference is whether `client_secret` appears in step 2. Where a step says "your `client_secret`", a public client simply omits it.

### Step 1 — send the user to the authorization endpoint

The user's browser navigates to `GET https://cloud.fastbound.com/OAuth/Authorize` with:

| Parameter | Required | Value |
|---|---|---|
| `client_id` | yes | Your client ID. |
| `redirect_uri` | yes | Must match one of the redirect URIs registered for your client, byte for byte. |
| `response_type` | yes | `code`. |
| `scope` | optional — send it anyway | Space-delimited scopes — see [Scopes](#scopes). Must be a subset of the scopes your client is approved for; anything else fails with `invalid_scope`. **Omitted means every scope registered for your client**, so an app that leaves it out asks for its maximum on every consent screen; name the set you need. |
| `state` | recommended | Opaque CSRF token you generate and verify on the callback. Round-tripped on success *and* on error. |
| `code_challenge` | **yes** | PKCE challenge — `BASE64URL-NOPAD(SHA256(ASCII(code_verifier)))`. |
| `code_challenge_method` | **yes** | `S256`. Nothing else is accepted. |

**PKCE is mandatory on every `authorization_code` request, and `S256` is the only supported method.** `plain` is deliberately rejected, as is a missing or malformed challenge — the challenge must be 43–128 characters from the unreserved set `[A-Za-z0-9-._~]`. This applies to confidential server-side clients too, not just public ones; there is no opt-out. A request without a valid challenge fails with `invalid_request`, and the `error_description` says so explicitly: *"code_challenge_method is required and must be 'S256'"*.

PKCE **supplements** client authentication here, it doesn't replace it. A confidential client sends both — `client_secret` *and* `code_verifier` — in the token request.

> **Don't infer "no PKCE" from the swagger.** FastBound's OpenAPI spec carries `usePkceWithAuthorizationCodeGrant: false`. That flag configures **Swagger UI's own built-in client** — whether the docs page uses PKCE when *it* calls the API — and says nothing whatsoever about what the authorization server requires. Reading a client-side UI setting as a server-side contract is an easy mistake to make and it produces an integration that fails on its very first authorize request. The server's behavior is the authority; the flag is not.

```python
import base64, hashlib, secrets

code_verifier = secrets.token_urlsafe(64)                # 86 chars, within the 43-128 range
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier.encode("ascii")).digest()
).decode("ascii").rstrip("=")                            # base64url, no padding
```

Keep the `code_verifier` in the user's server-side session — you need it in step 2, and it must be the one that produced this challenge.

The user logs into FastBound, picks which of their accounts to authorize (only the ones they own or hold Account Settings on, with API access on the plan — see above), approves the requested scopes, and is redirected back to your `redirect_uri` with a `code` and your `state`. The code expires **5 minutes** after issue and is single-use: exchange it immediately.

**Errors come back on the redirect, not as a page.** If the request fails for any reason other than a bad `client_id` or `redirect_uri`, FastBound redirects to your `redirect_uri` with `error`, `error_description`, and your `state` (RFC 6749 §4.1.2.1) — `invalid_request` (including PKCE problems), `invalid_scope` (unknown scope, or one the client isn't allowed to request), `unauthorized_client` (your client isn't registered for the authorization code grant), `access_denied` (the user declined, or isn't allowed to authorize your app), `unsupported_response_type`. Your callback handler must check for `error` before it looks for `code`. Only a missing/invalid `client_id` or `redirect_uri` renders an error page instead — there's nowhere safe to redirect to in that case.

### Step 2 — exchange the code for tokens

```http
POST /OAuth/Token HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<authorization_code_from_callback>
&redirect_uri=<exact_same_uri_used_in_step_1>
&code_verifier=<the_verifier_that_produced_your_code_challenge>
&client_id=<your_client_id>
&client_secret=<your_client_secret>     # CONFIDENTIAL clients only — omit entirely if public
```

`code_verifier` is required — it's the plaintext value whose SHA-256 you sent as `code_challenge` in step 1. A missing or non-matching verifier fails with `invalid_grant`, and the authorization code is spent either way; you can't retry the exchange.

The `client_id` and `client_secret` may be sent **either** in the request body **or** in an `Authorization: Basic` header (standard HTTP Basic with `client_id:client_secret`), but **never both**. Supplying them in both places is a malformed request and returns `invalid_request` (RFC 6749 §5.2) — not `invalid_client`, which is reserved for a genuine authentication failure through a single mechanism (unknown client, wrong secret). This one-location rule is enforced on `authorization_code` and `refresh_token` grants; it does not apply to `device_code`, which has no secret to place.

**A public client sends `client_id` in the body and no secret anywhere** — no `client_secret` field, no Basic header. PKCE is what authenticates it, which is why the `code_verifier` is not optional for these clients in any sense: it is the *entire* proof.

`redirect_uri` must exactly match the one used in step 1.

Grants are enforced per client: if your client isn't registered for the grant type you're asking for (say a device-code-only client trying `authorization_code`), the token endpoint returns `unauthorized_client`.

### Step 3 — receive tokens

```json
{
  "access_token": "ACCESS_TOKEN",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN",
  "refresh_token_expires_in": 2592000,
  "scope": "...",
  "fastbound_account_number": 12345,
  "fastbound_account_name": "ACME Guns LLC"
}
```

Use the `access_token` as `Authorization: Bearer <access_token>` on every API call. Treat both tokens as opaque: don't parse them, infer a format, or size a column to the length you see today. When it expires (or proactively a minute before `expires_in`), use the refresh token (see below) to get a new one without re-prompting the user. Both `expires_in` and `refresh_token_expires_in` are **per-application settings**; read them from every response rather than assuming an hour and thirty days.

`fastbound_account_number` is the account the user selected during consent — it's the `{accountNum}` in every subsequent `/{accountNum}/api/...` URL. Persist it alongside the tokens. See [Which account did the user authorize?](#which-account-did-the-user-authorize) below.

`scope` is the space-delimited list of scopes actually granted, which may be narrower than what you asked for. Read it rather than assuming you got everything you requested. The response is sent with `Cache-Control: no-store` and `Pragma: no-cache` (RFC 6749 §5.1) — don't put it behind a caching HTTP proxy or store the raw response body.

## Device Code flow

Use this for CLIs, desktop apps, mobile apps without a back end, IoT devices, or any **public client** that can't safely store a `client_secret` or host a redirect URI.

### Step 1 — request a device code

```http
POST /OAuth/Device_Authorization HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

client_id=<your_client_id>
&scope=<space delimited scopes>
```

Same scope rule as the authorization code flow: the requested scopes must be within the set your client is approved for. See [Scopes](#scopes). This call fails with `invalid_client` when the client isn't registered for the device authorization grant, and with `invalid_grant` when a requested scope is unknown or not registered for the client.

Note: **no `client_secret`**. The device code flow is for clients that can't keep one.

> **A `client_secret` sent on a device-code grant is accepted but never validated.** The swagger says so outright. That means sending one produces no error — so if you've wrapped a device-code client around a secret in the belief that it's authenticating anything, nothing will tell you otherwise. It isn't; the device code and the user's approval are what authenticate this flow. Don't ship a secret to a public client on the strength of the request succeeding, and treat "it worked" as no evidence at all here.

### Step 2 — show the user code

```json
{
  "device_code": "DEVICE_CODE",
  "user_code": "ABCDE-FGHIJ",
  "verification_uri": "https://cloud.fastbound.com/OAuth/Device",
  "verification_uri_complete": "https://cloud.fastbound.com/OAuth/Device?user_code=ABCDE-FGHIJ",
  "expires_in": 600,
  "interval": 5
}
```

Display the `user_code` and `verification_uri` to the user. They open the URL on a phone or laptop, log in, enter the code (not case-sensitive), confirm it matches the one on the device, pick the account, and approve. (`verification_uri_complete` embeds the code so the user only has to confirm — useful for QR codes.) The code expires after `expires_in` seconds — ten minutes as currently configured; a rejected code means start over from step 1, not retry.

### Step 3 — poll the token endpoint

While the user is approving on the other device, poll `POST /OAuth/Token` no more often than `interval` seconds:

```http
POST /OAuth/Token HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code
&device_code=<device_code_from_step_2>
&client_id=<your_client_id>
```

While the user hasn't approved yet, the response is a 400 with `error=authorization_pending`. Keep polling. Other relevant errors per RFC 8628:

- `slow_down` — you're polling too fast; increase the interval.
- `expired_token` — `device_code` lifetime ran out; restart from step 1.
- `access_denied` — the user denied, isn't allowed to authorize your app, or your app already held its maximum active tokens for that account when they approved (see [FastBound caps active tokens](#fastbound-caps-active-tokens-per-app-per-account)). Stop polling.

Once the user approves, you get the same token response shape as the authorization code flow — including `fastbound_account_number` and `fastbound_account_name` for the account the user picked while approving on the other device. A CLI or desktop app therefore never has to ask the user to type in their account number.

## Refresh token flow

Both grant types return a `refresh_token`. To exchange it for a fresh access token:

```http
POST /OAuth/Token HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=<your_refresh_token>
&client_id=<your_client_id>
&client_secret=<your_client_secret>   # confidential clients only
```

**Only a confidential client sends `client_secret` here.** Public clients omit it — that includes both device-code clients and public authorization-code clients.

A refresh grant carries no record of which flow minted the token it's renewing, so FastBound can't require a secret on refresh in general; it validates one only when the token traces back to a confidential authorization-code client. Don't read a refresh that succeeds without a secret as evidence that your client is public — check how it was registered.

The refresh response carries `fastbound_account_number` and `fastbound_account_name` too, for the account bound to the original grant. A refresh never moves a token to a different account, so the number won't change — but `fastbound_account_name` is re-read at each issue, so refresh is a free opportunity to update a cached display name.

### Every refresh rotates both tokens

**A refresh returns a new access token and a new refresh token, and the previous pair stops working.** FastBound updates the stored token in place: the old refresh token no longer matches anything, so presenting it again gets `invalid_grant` ("Invalid authorization code or refresh token"). The response has the same shape as the original token response, `refresh_token` included.

Two rules follow:

- **Persist the new refresh token before you use the new access token.** The moment the server answered, the token you sent was dead. If the response is lost in transit, or the process dies between receiving it and writing it, your stored copy is the dead one — and nothing tells you until the *next* refresh, an hour later, with no live request to point at the cause. Write the pair to storage in one transaction, then use it.
- **Never resend a refresh you aren't sure failed to reach the server.** A timeout after the server committed means the retry carries a spent token and earns `invalid_grant` for a grant that is alive. Retry only failures you know never arrived (connection refused, DNS); for anything else, re-read storage and start from what is there.

```python
# WRONG — uses the access token first; a crash here strands the dead refresh token in storage.
api.get(..., token=body["access_token"])
store.save(body)

# Correct — the new pair is durable before anything depends on it.
store.save_pair(body["access_token"], body["refresh_token"], expiry_from(body))
api.get(..., token=body["access_token"])
```

A refresh response always carries `refresh_token`. Index it — `body["refresh_token"]` — and don't write a fallback to the old one (`body.get("refresh_token") or conn.refresh_token`). The old token is dead the moment the response exists, so that fallback can only ever store a dead credential while looking careful; if the field were ever missing, that is a failure to surface, not a token to keep.

### Why a lost refresh token looks like a random disconnect

Nothing fails at the time. The access token that arrived with the lost refresh token works for its full lifetime, every call succeeds, and the connection reports healthy. It dies at the *next* refresh — roughly an hour later, from a scheduled job or a request that happened to cross the expiry — with `invalid_grant` and no live request pointing at the moment the token was lost. That is the signature of this bug: intermittent, a different dealer each time, impossible to reproduce on demand, and nothing in the logs near the failure. When a report looks like that, look an hour earlier for a refresh whose response was lost, a crash between response and write, or a second process refreshing the same connection.

The refresh also re-issues `refresh_token_expires_in`, so the grant's deadline rolls forward from the latest refresh — see below.

### Refresh must be single-flight

**Only one refresh per grant may be in flight at a time.** If two workers notice the same expiring token and both refresh, only the first wins: the second presents a token the first just rotated away and gets `invalid_grant`. FastBound does not revoke the grant on reuse — the winner's pair keeps working — so the damage is self-inflicted: the loser's error handler reads `invalid_grant` as "the user must reconnect" and tears down a live connection, or the loser's stale write lands on top of the winner's fresh tokens. Either way a race between your own processes disconnects the customer outright.

This is not a theoretical concern for anything running more than one instance: an autoscaled web tier, a queue worker pool, or a cron job overlapping with live traffic all produce it, and it surfaces as an intermittent "randomly disconnected" report that's near-impossible to reproduce.

Serialize it with whatever your deployment already has — a short-lived distributed lock or lease (Redis `SET NX PX`, a database row lock, a transactional lease document), keyed per connection. Three details make the difference between a lock that helps and one that doesn't:

- **Re-read the stored bundle after acquiring the lock.** The loser of the race arrives to find the winner has already refreshed; if it refreshes on the read it took *before* waiting, it spends a spent token — exactly the reuse event the lock existed to prevent. Re-read, and return the fresh access token if one is now there.
- **Hold the lease longer than the work it guards.** The token POST plus your secret-store reads and writes; if the lease can expire mid-refresh, a successor starts a second one. Erring long is cheap — the only cost is that a connection whose refresher crashed waits longer before another instance retries, and refreshes happen roughly hourly.
- **Fence the release.** Tag the lease with an owner ID and only release your own, or an overrunning instance will clear a successor's lease on its way out and admit a third refresher.

Refresh **proactively**, about a minute before `expires_in` elapses, so an in-flight request never races the expiry boundary.

### `refresh_token_expires_in` is the grant's deadline

`refresh_token_expires_in` (seconds) is the lifetime of the **refresh token**, not the access token — it's when the whole grant dies. Every successful refresh re-issues it, so the deadline **rolls forward from the latest refresh**: an integration that refreshes on schedule never reaches it, and one that sits idle longer than the refresh lifetime (a register that's been off for the season, a dealer who paused the integration) does. Past it, no refresh can succeed and no amount of retrying or waiting helps; the refresh fails with `invalid_grant` ("Refresh token has expired") and the user has to re-authorize.

Treat it as a distinct, terminal state in your own model, separate from "the request failed." A connection that's actually dead but shows as *Connected* in your UI will keep looking healthy right up until someone tries to use it — often the worst possible moment, like the start of a physical inventory count. Check it before you attempt a refresh, and when it has passed, tell the user to reconnect rather than surfacing a generic error or retrying.

**An absent `refresh_token_expires_in` means "not stated" — never "expired", and never "never expires".** Store the unknown as unknown (a null, not a sentinel date), and don't let downstream code collapse the two. Don't hardcode a lifetime from the sample response above either: token lifetimes are a per-client setting, so yours may differ and may change.

### Refresh failures that mean "reconnect", and the ones that don't

| Response | Meaning | Do |
|---|---|---|
| `invalid_grant` — "Refresh token has expired" | The grant's deadline passed | Mark disconnected; ask the user to authorize again |
| `invalid_grant` — "The account no longer has API access" | Plan downgraded, account suspended or deleted | Mark disconnected; the user has to fix their account before re-authorizing |
| `invalid_grant` — "Invalid authorization code or refresh token" | The token you sent matches nothing: rotated away (a race, a lost response), or deleted by the account owner on the Tokens tab | Re-read storage first — if a newer pair is there, use it. Otherwise mark disconnected |
| `unauthorized_client` | FastBound disabled, for your app, the grant that issued this token | Mark disconnected; talk to FastBound before asking users to reconnect |
| `invalid_client` | Wrong or missing `client_secret` on a confidential client | Your configuration, not the user's grant. Fix it; disconnect nobody |
| Timeout, connection error, 5xx | Nothing is known about the grant | Not a disconnect. Re-read storage, then retry only if the request never left |

Telling a dealer to reconnect for a failure in the bottom two rows is the support ticket you'll regret: they reconnect, it works (because it always did), and they stop trusting the integration.

## Active token limits, and how a grant ends

### There is no revoke endpoint

FastBound's OAuth surface is `/OAuth/Authorize`, `/OAuth/Token`, `/OAuth/Device_Authorization` and the `/OAuth/Device` page — **there is no `/OAuth/Revoke`**, and no programmatic way to hand a token back. Don't design a "disconnect" feature around calling one.

What you can do is delete your own copy of the tokens, which is what disconnecting should mean on your side. The authoritative revocation is the account owner's: the **Account Settings → Tokens tab** lists each token under *Authorized OAuth Applications* with your app's name, the grant type that issued it, and its creation date, and **only the account owner can delete one** — any other user sees a message naming the owner to contact. Deletion is immediate and disconnects only that token; other apps and legacy API keys are untouched. The integration starts failing on its next call. If an integration goes 401 across the board with no other explanation, that page is the first place to look.

Because your copy of a grant can be revoked without telling you, treat a persistent 401 as "this grant is gone, prompt the user to reconnect," not as a transient failure to retry.

### FastBound caps active tokens per app, per account

There's a **maximum number of simultaneously active tokens one app may hold for one account**, and the token exchange is refused once you're at it. The failure looks like this:

```json
{
  "error": "invalid_grant",
  "error_description": "The client app 'Acme POS' already has the maximum number of active tokens for account 12345"
}
```

Note that the description interpolates **your app's name and the account number**, so never match it whole — the stable part is the phrase `maximum number of active tokens`.

**The default limit is one active token per app per account**, and a token counts as active until its refresh token expires — an access token that expired an hour ago still occupies the slot for the rest of its refresh lifetime. If your product legitimately needs several concurrent tokens per account (one per workstation or register, say), ask FastBound to raise your app's limit when you register it; the limit is a per-app setting. Don't work around it by copying one token set to several machines — rotation means the first machine to refresh breaks the others (see [Every refresh rotates both tokens](#every-refresh-rotates-both-tokens)). Either one token per machine with the limit raised, or one refresher that holds the refresh token and hands short-lived access tokens to the machines.

This one is worth handling by name, because it's the rare OAuth failure with a **precise remedy the user can carry out themselves**: the **account owner** goes to the **Account Settings → Tokens tab**, deletes the entry for your application under Authorized OAuth Applications, then connects again. Only the owner can delete; a manager sees a message naming the owner. Reported as a generic "authorization failed," it sends people to support for something they could have fixed in fifteen seconds.

Two cautions on detecting it:

- **`invalid_grant` alone is not enough.** FastBound reuses that code for several unrelated failures (a spent or mismatched authorization code, a dead refresh token). You have to consult `error_description` as well.
- **Match narrowly and degrade safely.** Key off the distinctive phrase rather than the whole string, and fall through to your generic message if it doesn't match. If FastBound rewords the description you lose a helpful hint — which is fine. What you must not build is logic that *misreports* an unrelated `invalid_grant` as a token-limit problem and sends the user off to revoke a perfectly good token.

In the device flow the limit surfaces at approval time instead: the user is denied on the approval page and your poll gets `access_denied`, whose description also mentions the maximum number of active tokens.

You'll meet this most often in development, where every connect-disconnect cycle that doesn't delete the old token leaves it occupying the slot until its refresh token expires.

### An `authorization_code` grant must return a `refresh_token`

If it doesn't, treat it as a hard failure at connect time and don't persist the connection. Without a refresh token, the access token you just received is all you will ever have: the integration works for an hour, then dies with no way to renew and no user standing there to re-authorize. Failing loudly at connect — where the user *is* present and can retry — is far better than a connection that stores cleanly, reports success, and stops working during the night.

(On a **refresh** grant, an absent `refresh_token` is legitimate and means the opposite — keep the one you have. See above. The asymmetry is easy to get backwards.)

## Which account did the user authorize?

**One token is bound to exactly one account, permanently.** The account is chosen by the user during consent (from the accounts they're eligible for), fixed onto the issued token, and never changes for the life of that token or its refreshes. If your integration needs two accounts, it needs two authorizations and two token sets.

The token response tells you which one:

| Field | Type | Meaning |
|---|---|---|
| `fastbound_account_number` | JSON number | The account number. This is the `{accountNum}` path segment for every API call made with this token. |
| `fastbound_account_name` | JSON string | The account's name at the moment the token was issued — for display, e.g. "Connected to: ACME Guns LLC". |

Both are top-level members of the response object, on all three grant types (authorization code, device code, refresh token). They are absent from 4xx error responses. The `fastbound_` prefix is a vendor-specific extension parameter per RFC 6749 §8.2, matching the `X-FastBound-*` response-header convention used elsewhere in the API — the same shape as Stripe's `stripe_user_id` or QuickBooks' `realmId`.

What to do with them:

- **Persist `fastbound_account_number` with the token.** Store it in the same record as the access and refresh tokens; it's the only thing that makes the token usable. Do not prompt the user for their account number, and do not hardcode one in a multi-tenant product.
- **Treat `fastbound_account_name` as a point-in-time snapshot.** It's not refreshed by anything but a new token issue. If you display the name persistently, read it from `GET /{accountNum}/api/Account` (requires the `account:read` scope) instead. The field exists so an integration can confirm "you connected the right FFL" at connect time *without* holding `account:read`.
- **Don't look for a token-introspection or "who am I" endpoint.** There isn't one — every API route is account-scoped by its path, so there's nothing to ask which account a token belongs to. The token response is where you learn it, which is why you persist it rather than re-deriving it later.

## Scopes

Scopes are granular per resource area and per operation class, space-delimited in the `scope` parameter (RFC 6749 §3.3).

**Your app can only request the scopes it was approved for.** FastBound registers each client with an explicit allowed-scope set, decided when the client is issued. Asking for anything outside that set **fails** — the authorization request comes back as `invalid_scope` (on the redirect), not as a consent screen with the extra scopes quietly dropped. So the scope list you send is not a wish list: it must be a subset of what your client is approved for. Decide the set when you request credentials from **support@fastbound.com**, and if the integration later needs more, ask FastBound to widen the client's approved scopes before you change the request.

The scopes available to be approved for:

| Resource | Scope | Grants |
|---|---|---|
| Everything | `full_access` | Access all API operations on the authorized account |
| Account | `account:read` | Read account properties and settings |
| Acquisitions | `acquisition:read` | Read acquisitions and acquisition items |
| | `acquisition:write` | Create, edit, and commit acquisitions |
| | `acquisition:delete` | Delete acquisitions and items |
| Attachments | `attachment:read` | Download file attachments |
| Contacts | `contact:read` | Read contacts and licenses |
| | `contact:write` | Create and edit contacts |
| | `contact:delete` | Delete contacts |
| | `contact:merge` | Merge two contacts |
| Dispositions | `disposition:read` | Read dispositions and items |
| | `disposition:write` | Create, edit, and commit dispositions |
| | `disposition:delete` | Delete dispositions and items |
| Downloads | `download:read` | Download bound book PDFs |
| Form 4473s | `form4473:read` | Download Form 4473 PDFs |
| Inventory | `inventory:write` | Bulk verify inventory items |
| Items | `item:read` | Read items and inventory |
| | `item:write` | Edit items, set external IDs, undispose |
| | `item:delete` | Delete items |
| Multiple Sale Reports | `msr:read` | Download multiple sale reports |
| SmartLists | `smartlist:read` | Read reference data lists |
| Users | `user:read` | List account users |
| Webhooks | `webhook:read` | Read webhooks and events |
| | `webhook:write` | Create and edit webhooks |
| | `webhook:delete` | Delete webhooks |

`full_access` exists, and it is the wrong default. Ask for the narrowest set that covers what your integration actually does — both when you request credentials and on each authorization. An FFL looking at a consent screen that asks for everything, from an integration that only writes acquisitions, is being asked to take a risk that buys them nothing, and a leaked token then has the run of a regulated compliance system.

Mapping your integration to scopes, in practice:

- A POS pushing sales → `disposition:write` (+ `disposition:read` if you reconcile), `contact:read`/`contact:write` for buyers, `item:read`.
- Receiving stock from a distributor → `acquisition:write`, `contact:read`, `item:read`.
- A read-only reporting or reconciliation tool → the `:read` scopes only. Nothing else.
- Deletes (`*:delete`) are rarely appropriate for an unattended integration. In a compliance system, "delete" is a decision a human should be making.

### When a scope is missing

A valid token that lacks the scope for the endpoint gets **403** with an RFC 6750 §3 response — a `WWW-Authenticate: Bearer error="insufficient_scope", ... scope="..."` header, plus a JSON body naming what was needed:

```json
{
  "error": "insufficient_scope",
  "error_description": "...",
  "scope": "acquisition:write"
}
```

That `scope` value is the fix: it's what the token needed and didn't have. Refreshing won't help — a refresh returns the same scopes. The user has to re-authorize with the additional scope, so surface it as "reconnect to grant X," not as a generic permission error and not as a retry loop. If you're seeing it in development, check what your client is registered to request before you change the request.

## Worked example — Python authorization_code

```python
import os
import requests

# Pulled from a secrets manager at process start, NOT from a .env file.
CLIENT_ID = os.environ["FASTBOUND_CLIENT_ID"]
CLIENT_SECRET = os.environ["FASTBOUND_CLIENT_SECRET"]
REDIRECT_URI = "https://app.example.com/oauth/callback"

def exchange_code_for_tokens(code: str, code_verifier: str) -> dict:
    resp = requests.post(
        "https://cloud.fastbound.com/OAuth/Token",
        data={
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": REDIRECT_URI,
            "code_verifier": code_verifier,   # required; must match the challenge from step 1
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        },
        timeout=15,
    )
    resp.raise_for_status()
    return resp.json()

def connect(code: str, code_verifier: str) -> dict:
    """Complete the flow and record everything needed to call the API later."""
    tok = exchange_code_for_tokens(code, code_verifier)   # verifier came from the user's session
    return {
        # Tokens -> secrets manager / keychain. Account number -> your app's DB.
        "access_token": tok["access_token"],
        "refresh_token": tok["refresh_token"],
        "account_num": tok["fastbound_account_number"],   # the {accountNum} in every API URL
        "account_name": tok["fastbound_account_name"],    # snapshot, for "Connected to: ..."
    }

def refresh(refresh_token: str) -> dict:
    resp = requests.post(
        "https://cloud.fastbound.com/OAuth/Token",
        data={
            "grant_type": "refresh_token",
            "refresh_token": refresh_token,
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        },
        timeout=15,
    )
    resp.raise_for_status()
    tok = resp.json()
    # Rotation: the token we just sent is dead. Store the new pair before anything uses it.
    return tok
```

## Worked example — Python device_code (CLI)

```python
import os, time, sys, requests

CLIENT_ID = os.environ["FASTBOUND_CLIENT_ID"]  # public client, no secret

def login():
    r = requests.post(
        "https://cloud.fastbound.com/OAuth/Device_Authorization",
        data={"client_id": CLIENT_ID, "scope": "..."},
        timeout=15,
    )
    r.raise_for_status()
    auth = r.json()

    print(f"\nVisit {auth['verification_uri']} and enter code: {auth['user_code']}\n")
    print(f"Or open: {auth['verification_uri_complete']}\n")

    interval = auth["interval"]
    deadline = time.time() + auth["expires_in"]
    while time.time() < deadline:
        time.sleep(interval)
        t = requests.post(
            "https://cloud.fastbound.com/OAuth/Token",
            data={
                "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
                "device_code": auth["device_code"],
                "client_id": CLIENT_ID,
            },
            timeout=15,
        )
        if t.status_code == 200:
            tok = t.json()
            print(f"Connected to: {tok['fastbound_account_name']} "
                  f"({tok['fastbound_account_number']})")
            return tok
        err = t.json().get("error")
        if err == "authorization_pending":
            continue
        if err == "slow_down":
            interval += 5
            continue
        sys.exit(f"Auth failed: {err}")
    sys.exit("Device code expired before approval")
```

## Secret hygiene

Moved to `secret-hygiene.md`: storage tier by credential type, per-platform keychain and secrets-manager commands, and the step-by-step for a secret that has leaked — including one pasted into this conversation.

## HTTP Basic Authentication (legacy API Key)

> **Status:** Legacy but currently supported for some time. **No new or updated integration should use it** — see the case for OAuth at the top of this file. This section exists so the skill can review, debug, and migrate existing Basic Auth code.

For years, FastBound accepted HTTP Basic Authentication with a per-account API Key. Existing integrations built against this scheme continue to work during the OAuth transition; they will need to migrate before the legacy mechanism is retired. Timeline questions belong on https://fastbound.help.

### How it works

API Keys live under **Account Settings → Tokens tab** in the FastBound UI, alongside the account's authorized OAuth applications. Legacy API Keys are listed there masked and visually separated from OAuth tokens; generating a new one is restricted — typically to the account owner or FastBound staff, and it can be disabled for the account entirely. Anything listed on that page, legacy key or OAuth token, can be deleted by the account owner, and deletion takes effect immediately: the integration using it starts failing on its next call. If an integration suddenly returns 401 across the board, that page is the first place to look.

The key is presented as either the username, the password, or both, in a standard HTTP Basic header:

```http
GET /12345/api/Account HTTP/1.1
Host: cloud.fastbound.com
Authorization: Basic <base64(api_key:api_key)>
X-AuditUser: jdoe@example.com
```

Practical equivalents:

```bash
# As both username and password
curl -u "$FB_API_KEY:$FB_API_KEY" \
  -H "X-AuditUser: jdoe@example.com" \
  https://cloud.fastbound.com/12345/api/Account

# As username only (password empty)
curl -u "$FB_API_KEY:" ...
```

Any HTTP client's built-in Basic Auth support works — pass the API Key as both username and password (or as the username with an empty password) and let the library build the header.

### What's the same as OAuth

The `X-RateLimit-*` headers, the `X-FastBound-*` response headers (Multiple Sale, Existing Contact, Auto-Acquisition), the JSON request/response shapes, the error format, the URL pattern — all identical.

### What's different, besides the `Authorization` header

**`X-AuditUser` is required under Basic Auth and not required under OAuth.** The API Key identifies the *integration*, not a person, so Basic Auth needs the header to name a human for the audit trail; an OAuth access token already carries the identity of the user who authorized it, and FastBound uses that. Leaving the header on OAuth requests is silently accepted, which is convenient mid-migration — but don't add it to new OAuth code, and don't treat its absence on an OAuth write as a defect. See `common-patterns.md`.

### Secret hygiene for the API Key

The API Key is a long-lived bearer credential that grants full read/write access to the account. It belongs in the same handling tier as the OAuth `client_secret`:

- **Never** commit it to source control, paste it into Slack, write it to `.env`, or log it.
- **Store** it in a secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, 1Password Connect, OS keychain).
- **Replace** it when staff with access leave, or any time exposure is suspected. There is no in-place rotate: under Account Settings → Tokens tab you *generate a new key*, install it everywhere it's used, then *delete the old one* — in that order, so nothing goes dark mid-cutover. Deletion is immediate and unrecoverable. Treat any leak as an incident, not a typo.
- Some help articles (e.g., the integrator guides) say to "store the API Key" — they assume secure storage; they don't mean disk in plain text.

### Migrating Basic Auth → OAuth

The recommended path:

1. **Request OAuth credentials from FastBound** by emailing **support@fastbound.com**. Specify the grant type *and client type* you need — `authorization_code` as a **confidential** client (server-side, gets a `client_secret`), `authorization_code` as a **public** client (SPA/mobile/native, PKCE only, no secret), or `device_code` (CLI/headless) — plus your redirect URIs, scopes, and a brief description of the integration. There's no self-serve UI screen; credentials are issued by FastBound after a quick review, and the client type is fixed at registration.
2. **Decide who authorizes the integration.** OAuth is user-delegated, and the authorizing user becomes the audit identity on every write the token makes — so this choice replaces `X-AuditUser` rather than configuring it. If you want a single deterministic audit identity for backend writes (usually what a server-side integration wants), enroll a service-account FastBound user and have *that* user complete the authorization. Then **drop `X-AuditUser` from your write calls**; it's no longer required, and the value it used to carry is now decided by whose grant you're holding.
3. **Run both auth methods in parallel** behind a feature flag (or environment variable) during cutover. Verify that the new OAuth code path produces identical writes for a representative sample of operations.
4. **Cut traffic over to OAuth.** The API call shapes are identical; only the credential acquisition and the `Authorization` header change.
5. **Delete the API Key** under Account Settings → Tokens tab after cutover, so the legacy credential can no longer be used (even if it leaked). Deletion is immediate — confirm the OAuth path is carrying all traffic first. Don't skip this step: leftover API Keys are common attack surface, and an unused one is indistinguishable from a stolen one.

### Reviewing existing Basic Auth code — what to look for

When the user asks for a review of an existing FastBound integration that uses Basic Auth, check for:

- API Key stored in `.env`, source control, or plain config files → flag and recommend secrets manager.
- API Key passed via query string or URL (it shouldn't be — it must be in the `Authorization` header) → security bug.
- Missing `X-AuditUser` on writes → flag, **for Basic Auth code**. Under OAuth it isn't required and its absence is correct; don't report it there.
- `skip` advanced by `take` (or `page * take`) on any listing endpoint → **defect**, whether or not anyone has reported symptoms. `skip` is a page index; see `common-patterns.md`.
- Hardcoded `time.sleep(1)` or fixed RPS limits → replace with header-driven adaptive throttling (see `references/common-patterns.md`).
- No handling of `X-FastBound-MultipleSale` / `X-FastBound-ExistingContactId` / `X-FastBound-Acquisition*` headers → flag; these are easy to miss and have compliance/UX consequences.
- Long-running write queue between the app and FastBound → flag (see "Don't queue requests" in common-patterns).

These are all auth-independent; migrating to OAuth doesn't fix them, but it's a natural moment to address them in the same change. And the converse holds: a change to an existing Basic Auth integration is the moment to migrate it. Don't add new features on top of the key.

## Single Sign-On (SSO)

SSO signs a human into the FastBound *web UI*; it is not API authentication and shares nothing with OAuth. See `sso.md` for the SAML and legacy JWT mechanisms and how to disambiguate "we use Okta" questions.

## Common pitfalls

- **Sending `client_id`/`client_secret` in both the body and the Authorization header** — FastBound rejects this. Pick one.
- **Mismatched `redirect_uri`** — must be byte-identical between the authorization step and the token exchange. Trailing slashes count.
- **Skipping PKCE, or using `code_challenge_method=plain`** — PKCE with `S256` is mandatory on every `authorization_code` request, confidential clients included. Both mistakes fail with `invalid_request`.
- **Losing the `code_verifier` between the two steps** — it has to survive from the authorize redirect to the token exchange (server-side session, not a cookie you drop on redirect). Without the exact verifier, the code is unusable and can't be re-exchanged.
- **Only checking for `code` on the callback** — errors arrive at the same `redirect_uri` as `error`/`error_description`, with your `state`. Check `error` first.
- **Requesting scopes your client isn't approved for** — the authorization fails with `invalid_scope` rather than granting a subset. Widen the client's approved scopes with FastBound first.
- **Retrying on a 403 `insufficient_scope`** — that's a re-authorization, not a transient failure. Read the `scope` value in the response and prompt the user to reconnect.
- **Polling too fast in device flow** — respect the `interval` and back off on `slow_down`.
- **Trying to use `client_credentials`** — there is no such grant. If you need machine-to-machine, enroll a service-account user in the FastBound account and use authorization code or device flow on its behalf.
- **Asking the user to type in their account number** — the token response already tells you (`fastbound_account_number`). Prompting for it is an avoidable onboarding step and an avoidable class of support ticket.
- **Assuming one token covers several accounts** — it doesn't. A token is bound to the single account chosen at consent. Multi-account customers (an FFL with several premises) need one authorization per account, and your data model needs to hold a token *per* account, not per customer.
- **Storing `client_secret` in `.env`** — see the section above. Use a real secrets manager.
- **Using the new access token before the new refresh token is stored** — every refresh rotates both and the old pair is dead the moment the server answers. A lost response or a crash before the write kills the connection at the *next* refresh, an hour away from the code that caused it.
- **Retrying a timed-out refresh** — the server may have rotated already; the retry presents a spent token and reports a live grant as dead. Re-read storage instead.
- **Refreshing from more than one worker at once** — the loser presents a rotated-away token, gets `invalid_grant`, and your own error handler disconnects a live grant. Take a per-connection lock and re-read the stored token after acquiring it.
- **Copying one token set to several machines** — the first one to refresh invalidates the others. One token per machine (ask FastBound to raise the per-account limit at registration) or one refresher that hands out access tokens.
- **Treating an absent `refresh_token_expires_in` as "expired" (or as "never expires")** — it means neither. Store unknown as unknown.
- **Reporting a token-limit `invalid_grant` as a generic failure** — it's the one OAuth error with a self-service fix (the account owner deletes the app's token on the Account Settings → Tokens tab, then reconnects). Read `error_description`, not just `error`.
- **Looking for a revoke endpoint** — there isn't one. Disconnecting means deleting your own copy; the account owner revokes from the Account Settings → Tokens tab.
- **Sending customers to the help article, or pasting its steps into your docs** — link https://fastbound.link/oauth. It's the stable address for the customer-side instructions; the article URL and wording are not.
- **Concluding PKCE is optional from the swagger's `usePkceWithAuthorizationCodeGrant: false`** — that configures Swagger UI's own client, not the server's requirements.
- **Deserializing `fastbound_account_number` as a string** — it arrives as an unquoted JSON number. Accept either shape, normalize to a string. Same for `number` on `GET /api/Account`; see `common-patterns.md`.
