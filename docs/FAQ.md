# Frequently Asked Questions

Common questions about the KeePassXC multi-device setup.

[← Back to README](../README.md)

---

## Vault & Sync

**Q: Can I access Mac 1's SSH keys from Mac 2 since both open the same `work_vault`?**

No. The vault stores only the **path** to each key file — not the key content itself. When `work_vault` unlocks on Mac 2, KeePassXC looks for key files at those paths on Mac 2's disk — but those files only exist on Mac 1.

If you need Mac 1's keys on Mac 2, copy the private key files via `scp` or USB. Once they exist at the same paths on Mac 2, the existing vault entries load them automatically — no vault changes needed.

See [SSH_AGENT.md](SSH_AGENT.md) for how to copy keys safely.

---

**Q: Are Secrets & Configs entries synced between both Macs?**

The **KeePassXC entries** are fully synced — both Macs read the same `.kdbx` file from Google Drive. Add or edit a Secrets & Configs entry on Mac 1, save, Drive syncs it, and Mac 2 picks it up automatically.

The **macOS Keychain is not synced** — each Mac has its own Keychain. When you add a new secret to KeePassXC, also run the `security add-generic-password` command on each Mac separately. The Notes field in each entry contains the exact command — just open the entry and run it on whichever Mac you are setting up.

---

**Q: What happens if I forget the Master Password?**

There is no recovery option. The vault is encrypted with the Master Password and KeePassXC does not store it anywhere — not in the file, not on any server. If forgotten, the vault cannot be opened.

Store the Master Password somewhere physically secure (written on paper, stored safely) — not in any digital note on the same machine.

---

**Q: What happens if the `.kdbx` file gets corrupted or accidentally deleted?**

Google Drive keeps version history. If `work_vault.kdbx` or `personal_vault.kdbx` gets corrupted or deleted:

1. Go to [drive.google.com](https://drive.google.com)
2. Right-click the file → **Manage versions** (or go to Trash if deleted)
3. Restore a previous version

KeePassXC also automatically creates a `.kdbx.bak` backup file in the same folder every time you save — this is a one-version local backup.

---

**Q: Can I open the same vault on two Macs at the same time?**

Yes, both can have it open. But only make edits on one at a time. If both Macs save changes simultaneously, KeePassXC detects the conflict and prompts you to choose which version to keep.

Treat one Mac as the primary editing machine — let Drive sync before switching to the other.

---

**Q: Is it safe to store SSH keys and secrets in KeePassXC?**

The vault is encrypted with AES-256. SSH key files are not stored inside the vault — only path references are. The actual secrets (API tokens, passwords) in the Password and Notes fields are encrypted inside the `.kdbx` and only readable when the vault is unlocked with the Master Password.

The weakest point is the Master Password itself — a strong, unique Master Password is the most important security measure in this entire setup.

---

## SSH Keys

**Q: How do I actually use an SSH key to connect to a server?**

Once KeePassXC has loaded the key into the agent (vault unlocked), use the normal `ssh` command — no extra flags needed:

```bash
ssh <username>@<server-ip-or-hostname>
```

The SSH agent handles the key automatically in the background. No `-i ~/.ssh/<key-filename>` needed because the agent already has the key in memory.

Verify loaded keys before connecting:

```bash
ssh-add -l
```

If the vault is locked, SSH falls back to asking for a password or fails if password auth is disabled. Unlock the vault first and the key loads automatically.

---

**Q: Where are the public keys and what are they for?**

Every SSH key pair has two files:

| File | Example | Purpose |
|---|---|---|
| Private key | `~/.ssh/id_ed25519` | Stays on your machine only — never share |
| Public key | `~/.ssh/id_ed25519.pub` | Shared with servers — safe to send to anyone |

The public key is what gets added to a server's `~/.ssh/authorized_keys` to grant access. To view it:

```bash
cat ~/.ssh/<key-filename>.pub
```

Output looks like:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... description@machine
```

Safe to share, copy to GitHub, paste into server setup forms, or send over email.

---

**Q: How do I view the content of a private key file?**

```bash
cat ~/.ssh/<key-filename>
```

Output:
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXk...
-----END OPENSSH PRIVATE KEY-----
```

**Do not share this. Do not screenshot it. Do not copy it into any chat, email, or document.** Anyone who has this content can authenticate as you on any server that has your public key.

To check a key's fingerprint safely (does not expose key content):

```bash
ssh-keygen -lf ~/.ssh/<key-filename>
```

---

**Q: Should I share my private key with someone who asks for it?**

No. Never share a private key with anyone, for any reason.

The correct approach when someone needs access to the same server:

1. They generate their own key pair: `ssh-keygen -t ed25519 -C "their-name@purpose"`
2. They send you their **public key** (the `.pub` file)
3. You add their public key to the server's `~/.ssh/authorized_keys`
4. They connect using their own private key

Each person has their own key pair. If someone leaves, remove only their public key from the server — no one else is affected.

---

**Q: If I need to transfer a private key to another machine, how do I do it safely?**

| Method | When to use |
|---|---|
| `scp` on the same local network | Both machines on the same network |
| USB drive | When you can hand it over physically |
| GPG-encrypted transfer | When neither above is possible |

```bash
# Option 1 — scp
scp ~/.ssh/<key-filename> <username>@<ip>:~/.ssh/<key-filename>
ssh <username>@<ip> "chmod 600 ~/.ssh/<key-filename>"

# Option 3 — GPG encrypt before sending
gpg --symmetric --cipher-algo AES256 ~/.ssh/<key-filename>
# Share .gpg file and passphrase via separate channels
# Recipient decrypts:
# gpg --decrypt <key-filename>.gpg > ~/.ssh/<key-filename>
# chmod 600 ~/.ssh/<key-filename>
```

Never use email, Slack, Drive, or any messaging platform to transfer a private key unencrypted.

---

## Secrets & Keychain

**Q: If KeePassXC stores my secrets, how does the AI (Cursor, etc.) fetch them automatically?**

KeePassXC on its own only lets you manually copy a value. AI tools open **fresh shell sessions** that cannot inherit your clipboard or manually set variables.

The solution is **macOS Keychain** — a system daemon that auto-unlocks on Mac login. Every terminal, including the AI's, can read from it:

```bash
export MY_TOKEN=$(security find-generic-password -s "my-service-name" -w)
```

When you ask the AI to do something that needs a secret, it runs this command in its own session — Keychain returns the value immediately with no user interaction.

See [SECRETS_AND_CONFIGS.md](SECRETS_AND_CONFIGS.md) for the full setup.

---

**Q: Why do we need both KeePassXC AND macOS Keychain? Why not just one?**

They solve different problems:

| Store | Solves |
|---|---|
| KeePassXC | Master store — encrypted, synced via Drive, works on Android, backed up, survives Mac wipe |
| macOS Keychain | Terminal bridge — any session can fetch values automatically without user interaction |

- Keychain alone: local to one Mac, no sync, no Android, no backup, no management UI
- KeePassXC alone: AI tools open fresh sessions and cannot inherit your clipboard or manual actions

KeePassXC is where you manage secrets. Keychain is how secrets reach the terminal.

---

**Q: Does macOS Keychain protect values as securely as KeePassXC?**

Both are encrypted. But they are different in other ways:

| | KeePassXC | macOS Keychain |
|---|---|---|
| Encrypted at rest | Yes | Yes |
| Requires unlock | Master Password or Touch ID | Auto-unlocks on Mac login |
| Synced across Macs | Yes (via Google Drive) | No — local to each Mac |
| Works on Android | Yes | No |
| If Mac is wiped | File survives on Drive | Gone permanently |
| Management UI | Full GUI | No good native UI |

Keychain is convenient but not a replacement. KeePassXC is the source of truth.

---

**Q: Why not just put API tokens directly in `~/.zshrc`?**

`.zshrc` is a plain text file — no encryption, no protection. Anyone who runs `cat ~/.zshrc` on your Mac reads every token in plain text. It is also frequently committed to git repositories by accident.

Use `.zshrc` only for low-sensitivity dev / test tokens with very limited permissions. For production tokens or anything controlling real resources, always use Keychain.

---

**Q: When I copied a password using right-click → Copy Password, I could not paste it in Terminal. Why?**

KeePassXC's right-click → Copy Password writes to a **protected clipboard** that auto-clears after 10 seconds (configurable). Some terminal apps do not accept content from this protected clipboard.

**Workaround:** Entry → click the **eye icon** to reveal the password → select the text → `Cmd+C` → switch to Terminal → paste. This uses the standard system clipboard and always works.

Optionally increase the timeout: `KeePassXC → Settings → Security → Clear clipboard after` → set to 30 seconds.

---

**Q: If a token is leaking or misused, how do I stop it immediately?**

**Step 1 — Revoke at the service** (the only guaranteed kill switch):

| Service | Where to revoke |
|---|---|
| Cloudflare | dash.cloudflare.com → My Profile → API Tokens → **Revoke** |
| GitHub | Settings → Developer settings → Personal access tokens → **Delete** |
| AWS | IAM → Users → Security credentials → **Deactivate / Delete** |
| Any service | Settings → API / Security → delete the specific token |

Once revoked, the token stops working everywhere — in any terminal, any session, any process — no matter who has a copy of it.

**Step 2 — Clean up locally:**

```bash
security delete-generic-password -s "my-service-name"
```

**Step 3 — Generate a new token**, store it in KeePassXC, and run the Keychain setup again.

> There is no way to reach into another already-running terminal session and remove a variable that was already loaded. Revoking at the service is always the correct first action.

---

## Browser Extension & Autofill

**Q: The browser extension icon is grey — what does that mean?**

The extension is disconnected from KeePassXC. This means either:
- KeePassXC is not running — open the app
- The vault is locked — unlock it

Once KeePassXC is open and the vault is unlocked, the extension reconnects automatically and the icon turns green.

---

**Q: When I share my screen, KeePassXC is not visible to the other person — is that normal?**

Yes. KeePassXC has **screen capture protection** ON by default — the app window appears black or blank in screen shares, screenshots, and recordings. This is intentional.

To disable temporarily for a demo:
1. `Cmd + ,` → **Security** tab
2. Uncheck **Enable screen capture protection**
3. Restart KeePassXC

Always re-enable and restart after the session.

---

## TOTP & Passkeys

**Q: Can I use KeePassXC TOTP for services that require a proprietary MFA app?**

No. Some services (e.g. those using OneLogin Protect or Duo Mobile) only allow their proprietary app or hardware keys (YubiKey) as MFA options — standard TOTP authenticators are not available. This is a service-side policy decision, not a KeePassXC limitation.

For such services: KeePassXC autofills username and password normally. The MFA step is handled by the proprietary app on your phone.

---

**Q: Do passkeys work on Android with Keepass2Android?**

No. Keepass2Android does not currently support passkeys. For sites where you use a passkey on Mac, use username + password from the vault on Android instead. Most sites that offer passkeys still support password login as a fallback.

---

**Q: I set up TOTP in KeePassXC for an account. What if I want to add it to a different authenticator app as well?**

If you still have the original Base32 secret key (from your Notes field or the service's setup page), you can add it to another TOTP app by entering the key manually. However, many services only show the QR code / secret key once during initial setup.

**Best practice:** When setting up TOTP on a new account, copy the secret key into the KeePassXC Notes field before closing the setup page — this lets you re-add the TOTP to another app or recover if needed.

---

## Credential Files & Directory Encryption

**Q: I have credential files (JSON, PDF, `.env`, `.pem`) — how do I secure them?**

Use KeePassXC **file attachments** — files stored encrypted inside the `.kdbx`, synced via Drive.

1. KeePassXC → find or create the entry → **Edit Entry** → **Advanced** tab → **Attachments** → **Add**
2. To use the file: **Edit Entry** → **Advanced** → select attachment → **Save** → export to a temp location → use → delete the exported copy

Good for: service account JSON files, `.pem` / `.crt` certificates, `.env` files, credential PDFs.

See [MAINTENANCE.md — How to Secure Credential Files](MAINTENANCE.md#how-to-secure-credential-files) for full steps.

---

**Q: Can I encrypt a whole directory of files?**

Yes — using `hdiutil`, built into macOS. Creates an AES-256 encrypted `.dmg` file that mounts as a drive in Finder.

See [MAINTENANCE.md — How to Encrypt a Whole Directory](MAINTENANCE.md#how-to-encrypt-a-whole-directory) for the full process.

---

**Q: I generated a new SSH key — where do I store it?**

No `mv` needed. `~/.ssh/` is already the correct and secure location. Private key files there have `600` permissions by default.

1. Generate: `ssh-keygen -t ed25519 -C "purpose@machine" -f ~/.ssh/<key-filename>`
2. Create a vault entry in `SSH Identities` — see [MAINTENANCE.md — How to Add a New SSH Key](MAINTENANCE.md#how-to-add-a-new-ssh-key-in-the-future)
