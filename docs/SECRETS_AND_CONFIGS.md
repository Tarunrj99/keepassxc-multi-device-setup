# Secrets & Configs — KeePassXC + macOS Keychain

How API tokens and sensitive config values are stored, managed, and made available to any terminal session or AI tool — without copy-pasting.

[← Back to README](../README.md)

---

## Table of Contents

- [The two-layer approach](#the-two-layer-approach)
- [Why macOS Keychain works for AI tools](#why-macos-keychain-works-for-ai-tools)
- [KeePassXC Notes field template](#keepassxc-notes-field-template)
- [Step-by-step: adding a new secret](#step-by-step--adding-a-new-secret)
- [Retrieving a secret manually](#retrieving-a-secret-manually)
- [Using a secret in your terminal](#using-a-secret-in-your-terminal)
- [Optional: .zshrc quick access](#optional--zshrc-quick-access)
- [Emergency: revoke a token immediately](#emergency--revoke-a-token-immediately)
- [Naming convention](#naming-convention)
- [Example entries to create](#example-entries-to-create)
- [Completed entries tracker](#completed-entries-tracker)

---

## The Two-Layer Approach

KeePassXC is the **master store** — encrypted, synced via Google Drive, accessible on both Macs and Android. It is the one place where every secret lives permanently and safely.

However, KeePassXC alone only lets you manually copy a value and paste it somewhere. For using secrets in the terminal or letting AI tools access them automatically, there is a second layer: **macOS Keychain**.

| Layer | What it is | What it solves |
|---|---|---|
| **KeePassXC** | Encrypted `.kdbx` file, synced via Drive | Master store — secure, backed up, works on Android and both Macs |
| **macOS Keychain** | System-level encrypted store, auto-unlocks on login | Terminal access — any session (yours or AI) can read without a password |

They serve different purposes. **KeePassXC is where secrets are managed. Keychain is how secrets reach the terminal.**

You cannot use Keychain alone because:
- It is local to one Mac — does not sync
- No Android support
- No proper UI to manage entries
- No backup — wiped if Mac is reset

You cannot use KeePassXC alone for terminal access because:
- AI tools open fresh shell sessions that cannot inherit your clipboard or manual actions
- Every new terminal session starts empty — secrets set manually in one window do not carry over

---

## Why macOS Keychain Works for AI Tools

When Cursor AI (or any AI coding tool) runs a terminal command, it opens a **fresh shell session** — completely separate from your terminal. It cannot inherit any variables you set manually.

macOS Keychain solves this because it is a **system daemon** that auto-unlocks the moment you log in to your Mac. Every terminal — yours, the AI's, any process — can ask it for a value using the built-in `security` command:

```bash
# Any terminal, any session — no password, no Touch ID, no user interaction
export MY_TOKEN=$(security find-generic-password -s "my-service-name" -w)
```

As long as your Mac is logged in and the secret is stored in Keychain, it works immediately — in any shell, at any time.

---

## KeePassXC Notes Field Template

When adding a secret entry to KeePassXC, always fill the Notes field with the Keychain commands for that specific secret. This way you never have to remember the exact command — open KeePassXC, read Notes, run the command.

**Copy this template for every secret entry:**

```
Environment: prod
Used by: <which project or tool uses this>
Rotation: every 90 days (or as required by the service)

--- Keychain setup (run once on each Mac) ---
Store in Keychain (safe — value never appears in terminal history):
  printf 'Paste token (hidden): '; read -s t && echo && security add-generic-password -s "<keychain-service-name>" -a "<ENV_VAR_NAME>" -w "$t" -U && unset t && echo "Done — stored in Keychain"

Verify entry exists:
  security find-generic-password -s "<keychain-service-name>" -a "<ENV_VAR_NAME>"

Fetch in any terminal:
  export <ENV_VAR_NAME>=$(security find-generic-password -s "<keychain-service-name>" -w)

Remove (if rotating or revoking):
  security delete-generic-password -s "<keychain-service-name>"
```

**Example — Cloudflare API Key:**

```
Environment: prod
Used by: Cloudflare DNS management, Workers deployments
Rotation: rotate if leaked or every 6 months

--- Keychain setup (run once on each Mac) ---
Store in Keychain (safe — value never appears in terminal history):
  printf 'Paste token (hidden): '; read -s t && echo && security add-generic-password -s "cloudflare-api-key" -a "CF_API_TOKEN" -w "$t" -U && unset t && echo "Done — stored in Keychain"

Verify entry exists:
  security find-generic-password -s "cloudflare-api-key" -a "CF_API_TOKEN"

Fetch in any terminal:
  export CF_API_TOKEN=$(security find-generic-password -s "cloudflare-api-key" -w)

Remove (if rotating or revoking):
  security delete-generic-password -s "cloudflare-api-key"
```

---

## Step-by-Step — Adding a New Secret

### Part 1 — Add to KeePassXC (master store)

1. Open KeePassXC → click the correct vault tab (`work_vault` for work, `personal_vault` for personal)
2. In the left sidebar, click `Secrets & Configs`
3. Press `Ctrl + N`
4. Fill in:
   - **Title:** `Secret - <Service Name> <Key Type>` e.g. `Secret - Cloudflare API Key`
   - **Username:** the environment variable name e.g. `CF_API_TOKEN`
   - **Password field:** paste the actual secret value — it is masked and encrypted in the vault
   - **Notes tab:** paste the template above, filled in with the correct values and commands for this secret
5. Click **OK** → `Ctrl + S`

---

### Part 2 — Add to macOS Keychain on each Mac

Keychain is local to each machine — it does not sync. This must be done on **each Mac separately**.

> **Why not paste the value directly in the command?**
> Putting the token inline (e.g. `-w "actual-token"`) writes it to `~/.zsh_history`. Anyone who reads that file sees the token in plain text. The `printf` + `read -s` approach reads your input silently — nothing is echoed to the screen and the value never appears in shell history. The `-U` flag means the same command works for both first-time setup and future rotations — if the entry already exists, it updates instead of failing.

**Run this on Mac 1:**

```bash
printf 'Paste token (hidden): '; read -s t && echo && security add-generic-password -s "<keychain-service-name>" -a "<ENV_VAR_NAME>" -w "$t" -U && unset t && echo "Done — stored in Keychain"
```

When the prompt appears:
1. Switch to KeePassXC → open the entry → click the **eye icon** to reveal the password
2. Select the text → `Cmd+C`
3. Switch back to Terminal → paste → press Enter
4. Terminal prints `Done — stored in Keychain`

> **Use the eye icon + manual select** instead of right-click → Copy Password. KeePassXC's protected clipboard may not paste correctly in Terminal. See [FAQ](FAQ.md#q-when-i-copied-a-password-using-right-click--copy-password-i-could-not-paste-it-in-terminal-why) for details.

**Then repeat the same command on Mac 2.**

The Keychain entry is now available to any terminal or AI session on that Mac.

---

## Retrieving a Secret Manually

- Open KeePassXC → `Secrets & Configs` group → find the entry
- Right-click → **Copy Password** — value goes to clipboard, paste wherever needed
- The value stays masked in the UI unless you click the eye icon to reveal it

---

## Using a Secret in Your Terminal

```bash
# Fetch from Keychain into the current session
export CF_API_TOKEN=$(security find-generic-password -s "cloudflare-api-key" -w)

# Verify it loaded
echo $CF_API_TOKEN
```

**How AI tools use it:** The AI runs the exact same `security find-generic-password` command in its own fresh session — Keychain returns the value immediately with no user input required.

---

## Optional: `.zshrc` Quick Access

If you want a secret always available in every terminal without any command, add an export to `~/.zshrc`:

```bash
export MY_TOKEN="your-actual-token-value"
```

| Acceptable for | Not acceptable for |
|---|---|
| Low-sensitivity dev / staging tokens | Production tokens with broad permissions |
| Tokens with limited permissions | Anything controlling real resources |
| Temporary use (remove it soon) | Long-lived or high-value tokens |

> `.zshrc` is a plain text file. Anyone with access to your files can read every token in plain text. It is also frequently committed to git repositories by accident. Keychain is always the safer choice.

---

## Emergency — Revoke a Token Immediately

**Step 1 — Kill the token at the service** (this stops it everywhere instantly):

| Service | Where to revoke |
|---|---|
| Cloudflare | dash.cloudflare.com → My Profile → API Tokens → **Revoke** |
| GitHub | Settings → Developer settings → Personal access tokens → **Delete** |
| AWS | IAM → Users → Security credentials → **Deactivate / Delete** |
| Datadog | Organization settings → API keys → **Revoke** |
| Any service | Settings → API / Security → delete the specific key or token |

Once revoked at the source, the token stops working everywhere — in any terminal, any session, any process — no matter who has a copy of it.

**Step 2 — Clean up locally:**

```bash
# Remove from Keychain
security delete-generic-password -s "<keychain-service-name>"

# If you added it to .zshrc, remove that line and reload
source ~/.zshrc
```

**Step 3 — Generate a new token**, store it in KeePassXC, and run the Keychain setup again on each Mac.

> There is no way to reach into another already-running terminal session and remove a variable that was already loaded into memory. Revoking at the service is always the correct first action — it is the only guaranteed kill switch.

---

## Naming Convention

| Field | Format | Example |
|---|---|---|
| Title | `Secret - <Service Name> <Key Type>` | `Secret - Cloudflare API Key` |
| Username | Environment variable name, uppercase with underscores | `CF_API_TOKEN` |
| Password | The actual secret value | `(paste value here)` |
| Keychain service name | Lowercase, hyphen-separated | `cloudflare-api-key` |
| Notes | Template above, filled in for this specific secret | Keychain commands + rotation info |

---

## Example Entries to Create

| KeePassXC Title | Username (env var) | Keychain service name | Used for |
|---|---|---|---|
| `Secret - Cloudflare API Key` | `CF_API_TOKEN` | `cloudflare-api-key` | DNS, Workers, pages |
| `Secret - Datadog API Key` | `DD_API_KEY` | `datadog-api-key` | Monitoring, log forwarding |
| `Secret - MongoDB Connection String` | `MONGODB_URI` | `mongodb-uri` | App database connection |
| `Secret - Stripe Secret Key` | `STRIPE_SECRET_KEY` | `stripe-secret-key` | Payment processing |
| `Secret - GCP Service Account` | `GOOGLE_APPLICATION_CREDENTIALS` | `gcp-service-account` | GCP API access |
| `Secret - GitHub Token` | `GITHUB_TOKEN` | `github-token` | GitHub API, CI/CD |
| `Secret - SendGrid API Key` | `SENDGRID_API_KEY` | `sendgrid-api-key` | Email sending |
| `Secret - OpenAI API Key` | `OPENAI_API_KEY` | `openai-api-key` | AI API calls |

> **For `.env` files with many variables:** create one KeePassXC entry per variable (cleaner, easier to copy individual values) — or paste the full file contents into the Notes field of a single entry named `Secret - .env - <project name>` (faster to set up, harder to manage individual values later).

---

## Completed Entries Tracker

Use this table to track which secrets have been fully configured. Copy and fill in your own setup.

| Entry | Vault | KeePassXC | Keychain Mac 1 | Keychain Mac 2 | API Verified |
|---|---|---|---|---|---|
| `Secret - Cloudflare API Key` | `work_vault` | | | | |
| `Secret - Datadog API Key` | `work_vault` | | | | |
| `Secret - GitHub Token` | `personal_vault` | | | | |
| _(add your own)_ | | | | | |
