# Security

This document covers security considerations for the KeePassXC multi-device setup described in this guide, and how to report security issues with the guide itself.

---

## Security Properties of This Setup

### Encryption

- Both `.kdbx` vault files use **AES-256** encryption with **PBKDF2** key derivation by default
- The Master Password is never stored by KeePassXC — not in the file, not on any server
- macOS Keychain entries are encrypted by the macOS security subsystem and protected by your login password
- Encrypted disk images created with `hdiutil` use **AES-256**

### Data Ownership

- No credential data is sent to any third-party service
- Vault files live in your own Google Drive account — Google can see encrypted blobs but cannot read the contents without the Master Password
- macOS Keychain is local to your Mac — it is never synced externally

### Weakest Points

| Risk | Mitigation |
|---|---|
| Master Password forgotten or weak | Use a long, memorable passphrase. Write it down physically and store it securely. |
| Master Password exposed | Never store it digitally on the same machine. Never type it while screen sharing. |
| `.kdbx` file deleted | Google Drive version history allows recovery. KeePassXC creates `.kdbx.bak` on every save. |
| Token leaked via terminal history | Use `printf` + `read -s` pattern — never inline secrets in commands. |
| Exported credential file left undeleted | Delete immediately after import. Empty Trash. Never leave CSVs or JSONs in Downloads. |
| SSH private key shared insecurely | Use only `scp` on local network or USB. Never email, Slack, or Drive for private keys. |
| Same vault open on two Macs simultaneously | Only edit on one machine at a time. Let Drive sync before switching. |

### What Is Not Covered

- **Physical machine security** — disk encryption (FileVault on Mac) is separate from this setup but strongly recommended
- **Google account security** — securing your Google account (strong password + 2FA) is critical since Drive is the sync layer
- **Network security** — this setup does not protect against network-level attacks

---

## Master Password Guidance

The Master Password is the single most important piece of this setup. Everything else is only as secure as it.

**Characteristics of a strong Master Password:**
- At least 20 characters
- A passphrase (multiple random words) is often stronger and easier to remember than a complex short password
- Not used anywhere else
- Not based on personally identifiable information (name, birthday, pet name, etc.)

**Where to store it:**
- Written on paper, stored in a physically secure location (not at your desk)
- Not in a digital note, cloud document, or email to yourself
- Not inside KeePassXC itself

---

## Keychain Security Notes

macOS Keychain auto-unlocks when you log in to your Mac. This means:

- Any process running as your user can read Keychain values using the `security` command
- If your Mac account is compromised, Keychain values are accessible without additional authentication
- Keychain is the right layer for **convenience** (terminal and AI tool access), not for long-term archival of secrets
- KeePassXC remains the authoritative, encrypted source of truth — Keychain is a read-through cache

**What Keychain is not:**
- Not a replacement for KeePassXC
- Not backed up automatically
- Not synced across Macs
- Not accessible on Android

---

## Reporting Security Issues with This Guide

If you find a security issue with this guide — for example, advice that would weaken a user's setup, a command that exposes sensitive data, or an error that could lead to data loss — please do **not** open a public GitHub Issue.

Instead:

1. Go to the repository's **Security** tab on GitHub
2. Click **Report a vulnerability**
3. Describe the issue clearly: what the problem is, where in the guide it appears, and what the impact could be

We will review and respond as quickly as possible.

---

## KeePassXC Security

For security issues in KeePassXC itself (not this guide), refer to the official KeePassXC security disclosure process at [keepassxc.org](https://keepassxc.org).
