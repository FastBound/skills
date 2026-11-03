# Single Sign-On (SSO) — for the FastBound UI, NOT the API

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

- **Not API authentication.** SSO tokens cannot be used as `Authorization: Bearer ...` against the API. The API uses OAuth (going forward) or Basic Auth (legacy) — see `authentication.md`. They're separate systems.
- **Not a substitute for `X-AuditUser` under Basic Auth.** SSO doesn't change anything about API request requirements.
- **Not JIT user provisioning by itself.** Whether SSO auto-creates FastBound users on first sign-in depends on your account configuration; check with FastBound.

### Common confusion

> "We use Okta — does the FastBound API support that?"

What they're asking varies. Disambiguate:
- **"Our employees should be able to sign into the FastBound UI with Okta"** → that's SSO. Use SAML or JWT SSO; configure on FastBound side.
- **"Our integration server needs to call the FastBound API on their behalf"** → that's OAuth. Register an OAuth client; do `authorization_code` or `device_code` flow. Okta is not in the loop here.
- **"We want our internal app to be the OAuth identity provider for FastBound"** → not how it works. FastBound is the OAuth authorization server; your app is the client.
