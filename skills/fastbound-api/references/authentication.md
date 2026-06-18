# Authentication

> **Auth landscape — read this first.** FastBound supports two authentication mechanisms during this transition window:
>
> - **OAuth 2.0 (officially supported going forward).** Authorization code and device code grants. **Use this for all new integrations.** Migrate existing integrations to it as you touch them.
> - **HTTP Basic Auth with a per-account API Key (legacy, still supported).** A long-standing mechanism that will be around for some time so existing integrations don't break overnight. The skill should recognize it in existing code, answer questions about it, and help users migrate from it — but should not steer *new* code toward it. If you see code using `Authorization: Basic ...` against `cloud.fastbound.com`, that's Basic Auth.
>
> Most of this document covers OAuth. Basic Auth has its own section near the end ([HTTP Basic Authentication (legacy API Key)](#http-basic-authentication-legacy-api-key)) for review and migration scenarios.

FastBound uses OAuth 2.0 with two grant types:

- **`authorization_code`** — confidential clients (server-side apps with a `client_secret`)
- **`urn:ietf:params:oauth:grant-type:device_code`** — public clients (CLIs, desktop apps, devices that can't keep a secret or host a redirect URI)

There is **no `client_credentials` flow**. FastBound deliberately ties every integration to a real user account so that the `X-AuditUser` audit trail on writes is meaningful.

The endpoints live at the root of the FastBound host (no `/{accountNum}` prefix):

- `POST https://cloud.fastbound.com/OAuth/Token`
- `POST https://cloud.fastbound.com/OAuth/Device_Authorization`

**OAuth credentials are issued by FastBound, not self-served from a UI screen.** To get a `client_id` (and `client_secret` for confidential clients), email **support@fastbound.com** with your use case. There is no "register an app" page in the FastBound UI — don't go looking for one. Tell support: which grant type you need (`authorization_code` for server-side apps with a confidential secret; `device_code` for CLIs/desktop/public clients), your redirect URI(s) for authorization-code flow, the scopes you need, and a short description of the integration. They'll provision the client and send you the credentials.

### App access scope (important for third-party apps)

A newly registered OAuth app can only access accounts **owned by the app owner** — i.e., the FastBound user/organization that registered it. This is sufficient for first-party use (you're integrating your own FFL). For a third-party product that other FFLs will install (POS, e-commerce, ERP, range, manufacturing platforms, custom SaaS), the app needs to be **reviewed and approved by FastBound** before it can authenticate users from other accounts.

- **Process**: contact **sales@fastbound.com** to begin the review. Expect questions about your use case, customer profile, security posture, and how you'll handle FFL data and the audit trail.
- **Timing**: build it into your launch plan early — this is not a same-day approval.
- **Why**: FastBound apps act on regulated compliance data on behalf of FFLs. The review is a real one, not a rubber stamp.

You can build and test against your own account during development without approval. The wall is between "my app authorizes my user" and "my app authorizes someone else's user."

**Basic Auth is not an option for new third-party apps.** Legacy Basic Auth integrations continue to work for now, but new third-party apps must use OAuth from day one.

## Authorization Code flow

Use this when you have a server-side application that can:
- Keep a `client_secret` confidential, and
- Host a redirect URI that the user's browser can be sent back to.

### Step 1 — send the user to the authorization endpoint

The user's browser navigates to FastBound's authorization page. The exact URL and query parameters (response_type, scope, state, code_challenge for PKCE if supported) are documented at https://fastbound.help — confirm them there rather than guessing.

The user logs into FastBound, approves the requested scopes, and is redirected back to your `redirect_uri` with a short-lived `code` query parameter.

### Step 2 — exchange the code for tokens

```http
POST /OAuth/Token HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<authorization_code_from_callback>
&redirect_uri=<exact_same_uri_used_in_step_1>
&client_id=<your_client_id>
&client_secret=<your_client_secret>
```

The `client_id` and `client_secret` may be sent **either** in the request body **or** in an `Authorization: Basic` header (standard HTTP Basic with `client_id:client_secret`), but **never both**. FastBound rejects requests that supply credentials in both places.

`redirect_uri` must exactly match the one used in step 1.

### Step 3 — receive tokens

```json
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "v1.M2RhO...",
  "refresh_token_expires_in": 5184000,
  "scope": "..."
}
```

Use the `access_token` as `Authorization: Bearer <access_token>` on every API call. When it expires (or proactively a minute before `expires_in`), use the refresh token (see below) to get a new one without re-prompting the user.

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

Note: **no `client_secret`**. The device code flow is for clients that can't keep one.

### Step 2 — show the user code

```json
{
  "device_code": "GmRhmh...",
  "user_code": "WDJB-MJHT",
  "verification_uri": "https://cloud.fastbound.com/...",
  "verification_uri_complete": "https://cloud.fastbound.com/...?user_code=WDJB-MJHT",
  "expires_in": 1800,
  "interval": 5
}
```

Display the `user_code` and `verification_uri` to the user. They open the URL on a phone or laptop, log in, and enter the code. (`verification_uri_complete` embeds the code so the user only has to confirm — useful for QR codes.)

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
- `access_denied` — user explicitly denied.

Once the user approves, you get the same token response shape as the authorization code flow.

## Refresh token flow

Both grant types return a `refresh_token`. To exchange it for a fresh access token:

```http
POST /OAuth/Token HTTP/1.1
Host: cloud.fastbound.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=<your_refresh_token>
&client_id=<your_client_id>
&client_secret=<your_client_secret>   # only for confidential clients (auth code flow)
```

Public clients (device code) omit `client_secret`. Some servers issue a new refresh token on each refresh; assume FastBound may, and persist whichever one comes back.

## Worked example — Python authorization_code

```python
import os
import requests

# Pulled from a secrets manager at process start, NOT from a .env file.
CLIENT_ID = os.environ["FASTBOUND_CLIENT_ID"]
CLIENT_SECRET = os.environ["FASTBOUND_CLIENT_SECRET"]
REDIRECT_URI = "https://app.example.com/oauth/callback"

def exchange_code_for_tokens(code: str) -> dict:
    resp = requests.post(
        "https://cloud.fastbound.com/OAuth/Token",
        data={
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": REDIRECT_URI,
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
        },
        timeout=15,
    )
    resp.raise_for_status()
    return resp.json()

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
    return resp.json()
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
            return t.json()
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

This is not optional. FastBound credentials grant read/write access to a regulated compliance system; if they leak, an attacker can fabricate or alter A&D records on behalf of an FFL.

### Storage rules by credential type

| Credential | Lifetime | Tier | Where it goes | Where it must NOT go |
|---|---|---|---|---|
| `client_secret` (auth code flow) | Long (until rotated) | Highest | AWS Secrets Manager / GCP Secret Manager / Azure Key Vault / HashiCorp Vault / 1Password Connect; OS keychain for local dev | `.env`, `.envrc`, source control (even private), CI env vars visible to team, log files, error trackers, browser local storage |
| `refresh_token` | Long (weeks–months) | Same as `client_secret` | Same | Same |
| `access_token` | Short (minutes–hours) | High | In-memory only; refresh on demand | Disk, logs |
| Device-flow `device_code` / `user_code` | Single-use, short | Low | Memory during the polling window | Logs |

### Practical patterns

- **Server-side apps**: at process start, fetch `client_secret` from a secrets manager and hold it in memory. Don't materialize it on disk. If you must use environment variables to bridge to a library that reads them, set the env var from the secrets manager fetch — don't load it from a file.
- **Local development**: macOS Keychain via `security add-generic-password` and `security find-generic-password`; Linux `secret-tool` (libsecret); Windows Credential Manager. There are language wrappers: Python `keyring`, Node `keytar`, Go `zalando/go-keyring`. **Not `.env`.**
- **CLIs (device flow)**: persist the refresh token to the OS keychain under a service name like `fastbound:<client_id>:<account>`. On startup, read it from the keychain and use it to mint an access token. Never write tokens to `~/.config/<app>/credentials.json` or similar plain files.
- **Mobile apps (device flow)**: iOS Keychain, Android EncryptedSharedPreferences / Keystore.
- **CI/CD**: store `client_secret` in the platform's secrets store (GitHub Actions secrets, GitLab CI/CD variables marked masked + protected, etc.). Never echo it. Be aware that secrets are visible to anyone who can edit workflows — limit who can.
- **Logging**: scrub `Authorization` headers and any field named `client_secret`, `access_token`, `refresh_token`, `signingSecret`. Many HTTP client loggers do this by default; verify yours does for FastBound calls.

### When (not if) a leak happens

If a `client_secret` ends up in a Git commit, an error tracker, a Slack message, or a `.env` file someone screenshots: rotate it immediately via the FastBound app, audit recent API activity for the client, and document it. Don't silently replace and move on — for a compliance-adjacent integration, the audit log is part of the story.

## HTTP Basic Authentication (legacy API Key)

> **Status:** Legacy but currently supported. New integrations should use OAuth. This section exists so the skill can review, debug, and help migrate existing Basic Auth code.

For years, FastBound accepted HTTP Basic Authentication with a per-account API Key. Existing integrations built against this scheme continue to work during the OAuth transition; they will need to migrate before the legacy mechanism is retired. Timeline questions belong on https://fastbound.help.

### How it works

Each FastBound account can generate an **API Key** in the FastBound UI (Settings → Generate API Key). The key is presented as either the username, the password, or both, in a standard HTTP Basic header:

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

```python
import requests
r = requests.get(
    "https://cloud.fastbound.com/12345/api/Account",
    auth=(api_key, api_key),
    headers={"X-AuditUser": "jdoe@example.com"},
)
```

### What's the same as OAuth

Everything else: `X-AuditUser` requirement on writes, the `X-RateLimit-*` headers, the `X-FastBound-*` response headers (Multiple Sale, Existing Contact, Auto-Acquisition), the JSON request/response shapes, the error format, the URL pattern. **Only the `Authorization` header differs.**

### Secret hygiene for the API Key

The API Key is a long-lived bearer credential that grants full read/write access to the account. It belongs in the same handling tier as the OAuth `client_secret`:

- **Never** commit it to source control, paste it into Slack, write it to `.env`, or log it.
- **Store** it in a secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, 1Password Connect, OS keychain).
- **Rotate** it via the FastBound UI when staff with access leave or any time exposure is suspected. Treat any leak as an incident, not a typo.
- Some help articles (e.g., the integrator guides) say to "store the API Key" — they assume secure storage; they don't mean disk in plain text.

### Migrating Basic Auth → OAuth

The recommended path:

1. **Request OAuth credentials from FastBound** by emailing **support@fastbound.com**. Specify the grant type you need (`authorization_code` for server-side apps with a confidential `client_secret`, or `urn:ietf:params:oauth:grant-type:device_code` for CLIs/desktop/public clients), your redirect URIs (for auth-code flow), scopes, and a brief description of the integration. There's no self-serve UI screen — credentials are issued by FastBound after a quick review.
2. **Decide on the audit user.** OAuth is user-delegated — the user who authorizes the integration is identified. The `X-AuditUser` value can still be a service-account user; it doesn't have to match the OAuth-authenticating user. If you want a single deterministic audit identity for backend writes, enroll a service-account FastBound user and use that email.
3. **Run both auth methods in parallel** behind a feature flag (or environment variable) during cutover. Verify that the new OAuth code path produces identical writes for a representative sample of operations.
4. **Cut traffic over to OAuth.** The API call shapes are identical; only the credential acquisition and the `Authorization` header change.
5. **Rotate the API Key** in FastBound after cutover so the legacy credential can no longer be used (even if it leaked). Don't skip this step — leftover API Keys are common attack surface.

### Reviewing existing Basic Auth code — what to look for

When the user asks for a review of an existing FastBound integration that uses Basic Auth, check for:

- API Key stored in `.env`, source control, or plain config files → flag and recommend secrets manager.
- API Key passed via query string or URL (it shouldn't be — it must be in the `Authorization` header) → security bug.
- Missing `X-AuditUser` on writes → flag; this works the same whether auth is Basic or OAuth.
- Hardcoded `time.sleep(1)` or fixed RPS limits → replace with header-driven adaptive throttling (see `references/common-patterns.md`).
- No handling of `X-FastBound-MultipleSale` / `X-FastBound-ExistingContactId` / `X-FastBound-Acquisition*` headers → flag; these are easy to miss and have compliance/UX consequences.
- Long-running write queue between the app and FastBound → flag (see "Don't queue requests" in common-patterns).

These are all auth-independent; migrating to OAuth doesn't fix them, but it's a natural moment to address them in the same change.

## Single Sign-On (SSO) — for the FastBound UI, NOT the API

> **Read this only if the user is asking about SSO.** SSO is a separate concern from API authentication. The two systems do different things and don't share credentials.

FastBound supports SSO so an organization's existing identity provider (IdP) can log users into the FastBound web UI without a separate password. Two SSO mechanisms exist:

- **SAML 2.0** — the path going forward. See the separate help-site article on SAML SSO; reach out to FastBound to enable. Recommend this for any new SSO setup.
- **JWT-based SSO (HS256)** — legacy mechanism, **not widely used and may be deprecated**. Documented at https://fastbound.help/en/articles/3963631-single-sign-on. Recognize it in existing integrations; don't steer new setups toward it.

### How JWT SSO works (HS256) — legacy reference

1. FastBound provides you with a signing secret (treat with the same care as `client_secret`/API Key — secrets manager, never `.env`, never in client-side JavaScript).
2. When your app needs to sign a user into FastBound, generate a JWT with these claims:

| Claim | Value |
|---|---|
| `alg` (header) | `"HS256"` |
| `typ` (header) | `"JWT"` |
| `jti` | Unique, non-null, **≥128 bits** (≥16 chars). Per-token nonce for replay protection. |
| `iss` | Issuer URL FastBound provides on enablement. |
| `iat` | UNIX timestamp (seconds), at issuance. |
| `aud` | `"https://cloud.fastbound.com"` (in standard deployments). |
| `sub` | A user identifier from your system that **uniquely identifies the user and never changes**. (Don't use email if email can change.) |

3. Sign with HS256 using the shared secret.
4. Sign the user in by appending the JWT to any FastBound URL as a query parameter:

```
https://cloud.fastbound.com/SomePath?token=<YOUR_JWT_HERE>
```

Validity window is **±5 minutes from `iat`**. Generate the JWT *at the moment* the user clicks "Open FastBound" — not at page load — so you don't burn the window.

> **Reminder**: this JWT SSO path is legacy. New SSO setups should use SAML 2.0. The content above is for reading existing integrations and answering questions — not for building new ones.

### What SSO is NOT

- **Not API authentication.** SSO tokens cannot be used as `Authorization: Bearer ...` against the API. The API uses OAuth (going forward) or Basic Auth (legacy). They're separate systems.
- **Not a substitute for `X-AuditUser` under Basic Auth.** SSO doesn't change anything about API request requirements.
- **Not JIT user provisioning by itself.** Whether SSO auto-creates FastBound users on first sign-in depends on your account configuration; check with FastBound.

### Common confusion

> "We use Okta — does the FastBound API support that?"

What they're asking varies. Disambiguate:
- **"Our employees should be able to sign into the FastBound UI with Okta"** → that's SSO. Use SAML or JWT SSO; configure on FastBound side.
- **"Our integration server needs to call the FastBound API on their behalf"** → that's OAuth. Register an OAuth client; do `authorization_code` or `device_code` flow. Okta is not in the loop here.
- **"We want our internal app to be the OAuth identity provider for FastBound"** → not how it works. FastBound is the OAuth authorization server; your app is the client.

## Common pitfalls

- **Sending `client_id`/`client_secret` in both the body and the Authorization header** — FastBound rejects this. Pick one.
- **Mismatched `redirect_uri`** — must be byte-identical between the authorization step and the token exchange. Trailing slashes count.
- **Polling too fast in device flow** — respect the `interval` and back off on `slow_down`.
- **Trying to use `client_credentials`** — there is no such grant. If you need machine-to-machine, enroll a service-account user in the customer's FastBound account and use authorization code or device flow on its behalf.
- **Storing `client_secret` in `.env`** — see the section above. Use a real secrets manager.
- **Treating the access token as long-lived** — refresh on demand using `refresh_token`. Don't hardcode "tokens last 1 hour" assumptions; read `expires_in`.
