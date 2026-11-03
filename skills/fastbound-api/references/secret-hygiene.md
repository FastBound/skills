# Secret hygiene for FastBound credentials

Applies to every credential in `authentication.md`: OAuth `client_secret`, access and refresh tokens, legacy API Keys, and webhook signing secrets.

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

A `client_secret`, API Key, refresh token, or signing secret that lands in a Git commit, an error tracker, a Slack message, a screenshotted `.env`, a support ticket, or **a chat window with an AI assistant** is burned. Not "probably fine." Not "only I saw it." Replace it, audit recent API activity for that credential, and write down what happened — for a compliance-adjacent integration, the audit trail is part of the story.

**If the user pastes a live secret into the conversation, say so immediately and treat replacing it as the task.** Don't accept it and carry on, don't soften it into a suggestion, and don't wait to be asked. It is now in a transcript, and likely in logs and scrollback too. Ask them to confirm it's real (not a placeholder) — and if it is:

1. **Start the replacement.** A legacy API Key: generate a new one under Account Settings → Tokens tab. A `client_secret`: email **support@fastbound.com** and ask for the client's secret to be reissued — that's not self-serve. Get the replacement in hand *before* killing the old one.
2. **Move the new value by clipboard, never through the chat.** Have them copy it from FastBound, then pipe it straight from the clipboard into wherever it belongs. It should never be typed into the conversation, echoed back, written to a scratch file, or passed as a command-line argument (argv is visible to every process on the box, and lands in shell history).

   | Platform | Read clipboard | Clear it afterward |
   |---|---|---|
   | macOS | `pbpaste` | `pbcopy </dev/null` |
   | Linux (Wayland) | `wl-paste` | `wl-copy --clear` |
   | Linux (X11) | `xclip -selection clipboard -o` (or `xsel -b`) | `xclip -selection clipboard /dev/null` |
   | Windows (PowerShell) | `Get-Clipboard` | `Set-Clipboard -Value ''` |

   ```bash
   # macOS / Linux — swap pbpaste for wl-paste or xclip -selection clipboard -o.
   pbpaste | gcloud secrets versions add fastbound-client-secret --data-file=-
   pbpaste | aws secretsmanager put-secret-value --secret-id fastbound/client-secret \
       --secret-string file:///dev/stdin
   pbpaste | vault kv put -mount=secret fastbound client_secret=-

   # Local dev keychains, no value in argv:
   security add-generic-password -s fastbound -a client_secret -w        # macOS, -w prompts
   wl-paste | secret-tool store --label="FastBound" service fastbound key client_secret  # Linux
   ```

   ```powershell
   # Windows PowerShell. Get-Clipboard stays in-process, so nothing hits argv or history.
   Get-Clipboard | gcloud secrets versions add fastbound-client-secret --data-file=-
   Get-Clipboard | vault kv put -mount=secret fastbound client_secret=-

   # Windows Credential Manager via PowerShell SecretManagement:
   Set-Secret -Name fastbound-client-secret `
       -Secret (Get-Clipboard | ConvertTo-SecureString -AsPlainText -Force)
   ```

   Avoid the shapes that *look* equivalent but aren't: `--secret-string "$(pbpaste)"`, `-w "$(pbpaste)"`, and `cmdkey /pass:<value>` all put the secret in argv, where any local process can read it and the shell records it in history.

3. **Install it in every environment that needs it** — prod, staging, CI — from the secret store, not by copying the value around. **Not a `.env` file**, not `~/.config/<app>/credentials.json`, not a CI variable pasted into a web form if a secrets store is available. If a `.env` is what they have today, this is the moment to move off it; the exposure just proved the point.
4. **Kill the old credential.** Delete the leaked API Key under Account Settings → Tokens tab, or confirm with support that the old `client_secret` is revoked. Until this step, the leak is still live.
5. **Clean up the trail.** Clear the clipboard when done — see the table above for the command on their platform. If it went through a shell, remove it from history (`history -d`, or delete the line from `~/.zsh_history` / `~/.bash_history` / PSReadLine's `ConsoleHost_history.txt`). If it's in a Git commit, replacing the credential is mandatory and rewriting history is optional — the credential is what matters, and it's already dead.
6. **Audit.** Check recent API activity for that credential for anything the user doesn't recognize, and — because this is an FFL's compliance system — check whether any A&D records were written that they can't account for.

Never ask a user to paste a secret into the chat in the first place. If you need to know whether a value is set, have them check a property of it (whether the store returns a value, its first few characters), not the value itself.
