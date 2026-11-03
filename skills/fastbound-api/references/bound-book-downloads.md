# Automated bound-book downloads

ATF Ruling 2016-1 establishes that an electronic A&D book must produce a complete, exportable record of acquisitions and dispositions. In practice, FFLs using FastBound need to **regularly download a copy of their bound book** (typically daily) and store it in a way they could hand to an inspector. FastBound's download endpoints make this an automated background task, not a manual export-every-day chore.

Reference article: https://fastbound.help/en/articles/8145708-automated-bound-book-downloads

> **What you're downloading is a PII archive.** A bound book export is every transferee's name and address, paired with the firearm they took, across the account's history — in a single file. Store it encrypted, on storage the licensee controls, with access limited to the people who need it; don't drop it in a shared drive, an unencrypted bucket, or an email attachment. See "This API carries other people's PII" in `references/common-patterns.md`.

## When this matters

- **The licensee is required by their compliance program** (or their CFO, or their counsel) to keep daily snapshots of the bound book, off-platform, ideally on hardware they control. Common in larger / multi-location FFLs.
- **You're integrating FastBound into a regulated environment** (e.g., a manufacturer's quality system, a pawn shop chain's compliance pipeline) and need scheduled exports as part of the audit trail.
- **You're building a self-hosted analytics or backup pipeline** that periodically ingests the bound book.

## Use the official scripts

FastBound publishes ready-to-use scripts at:

**https://github.com/FastBound/Support/tree/main/scripts/Bound%20Book%20Download**

Four flavors, pick by environment:

| Script | Platform | Use when |
|---|---|---|
| `Download-BoundBooks.ps1` | PowerShell 7+ | Multi-account scheduling on Windows; supports secure credential storage |
| `Download-BoundBook.ps1` | PowerShell 7+ | Single account on Windows |
| `download-boundbook.sh` | macOS / Linux | Single account, cURL-based; small footprint |
| `download-boundbook.py` | Python 3 | Cross-platform; no third-party Python deps |

Each script's README in the repo covers:

- Installation and prerequisites (PowerShell 7+, curl, Python 3).
- Scheduling instructions for **Windows Task Scheduler**, **macOS launchd**, and **Linux cron**.
- Secure credential storage (the API Key under Basic Auth, the OAuth refresh token under OAuth — handle both like database passwords; see `authentication.md` for the secret hygiene tier).

The official scripts are FastBound-maintained reference implementations — chunking, rate-limit handling, and scheduling are already worked out. They're the fastest way to get a daily download running and the easiest answer to "do you have something I can just install?". Mention them when the user asks "how do I automate bound book downloads?".

## Building your own is also fine

The official scripts are guidance, not gatekeeping. Plenty of legitimate reasons to roll your own:

- **Embedded pipeline**: the bound book lands in S3, gets ingested by Snowflake, a Lambda processes it. You want this in your own deployment, not as a cron job calling a vendor script.
- **Non-supported runtime**: Go, Rust, .NET, JVM. The official scripts cover PowerShell, bash, and Python — fine for ops scripting, but if your stack is something else and you want one fewer language in your toolchain, write your own.
- **Custom triggers**: download not on a schedule but on a specific business event (end-of-day signal from your POS, end of business day in a particular timezone, on-demand from an admin dashboard).
- **Centralized credential / secrets management**: you've already got a secret-fetching pattern that doesn't match the script's approach.
- **Differential or incremental processing**: you want to detect what changed since the last download rather than re-download everything.
- **Audit-trail integration**: you want every download recorded in your own audit system with operator identity, request ID, file hash, etc.

If you go this route, treat the official scripts as a reference for what calls to make. The call is `POST /{accountNum}/api/Downloads/BoundBook` — one endpoint, no request body, no query parameters (see `swagger/accounts-swagger.json` for the exact shapes). A 200 returns `{ url, fileSizeInBytes, urlExpiresOnSeconds }`; the PDF itself is at that `url`, not in the response.

Authentication is the same as the rest of the accounts API — OAuth bearer (preferred) or Basic Auth (legacy), same `X-RateLimit-*` headers, same `accountNum` path placement — with one exception worth knowing before you debug it: **this endpoint requires `X-AuditUser` even under OAuth**, which inverts the rule everywhere else. The swagger states it on the operation itself.

Fetching the returned `url` is a second request that takes no `Authorization` header at all — the link is time-limited, which is what `urlExpiresOnSeconds` is telling you. FastBound's own scripts send nothing but a `User-Agent` on that leg.

Things to get right when building your own:

- **Respect rate limits via the response headers** — see `common-patterns.md`. Don't hardcode delays.
- **Stream large downloads** rather than loading them fully into memory. A bound book for a high-volume FFL can be substantial.
- **Verify file integrity after download** (size > 0, recognizable structure, plausible row counts) before you consider the day's run successful.
- **Handle the not-yet-generated case** — a 204 means the account exists but has no bound book PDF yet; a 404 means the account number is wrong. Neither is a job ticket: the API exposes no polling or status endpoint, so treat a 204 as "nothing to archive this run" and alert on it rather than retrying in a loop.
- **Don't skip authentication on retries.** Token refresh on 401 is the same as anywhere else.
- **Log enough to debug a missed run six months from now.** Timestamp, request ID (if returned), HTTP status, byte count, file hash.

The more your situation looks like the official scripts' assumed shape (PowerShell / bash / Python script on a server with cron, single-account or small multi-account, file lands on disk), the more you should consider just using them. The further from that shape, the more likely a custom build is the right call. Either way, FastBound's API doesn't care.

## Compliance framing for the licensee

Two things to communicate to a licensee setting this up:

1. **The downloaded bound book is *their* record.** Treat it like any other regulated record: defined retention, restricted access, integrity controls. Where it lives (encrypted volume, S3 with object lock, NAS with snapshots) is a compliance decision.
2. **Automation does not absolve the licensee of verifying it ran.** Build alerting around "the daily download did not happen" and "the daily download produced a zero-byte file." A bound book backup pipeline that silently broke six months ago is worse than no pipeline.

You're the technical integrator; the regulatory program is theirs. Build the plumbing reliably; surface failures loudly; let their compliance officer make the storage and retention calls.

## Common pitfalls

- **Building a fragile bash one-liner instead of using the official script.** The scripts handle pagination, authentication, large-file streaming, and error retry; one-liners that try to skip those steps fail at scale.
- **Storing the download credentials in cron's environment.** Same secret-hygiene rules as everywhere else: use a secrets manager / OS keychain. Cron `MAILTO` will happily email a `set -x` log to the operator if the script ever logs the API key — don't.
- **No verification that the file is non-trivial.** A successful HTTP 200 isn't proof the file's contents are right. Check size, count rows, alert on anomalies.
- **No alerting on missed runs.** Schedule a separate "did the download happen?" check that fires if the expected file is older than expected.
