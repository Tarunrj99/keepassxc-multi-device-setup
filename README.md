# KeePassXC Multi-Device Setup Guide

**A practical, self-hosted password manager setup across multiple Macs and Android.**  
One encrypted vault file per context. No third-party password cloud. Google Drive for sync. Works with browser autofill, SSH agent, TOTP, passkeys, and AI tools.

_Two vaults. Any number of devices. One Google Drive account._

![keepassxc](https://img.shields.io/badge/KeePassXC-2.7.7+-5085A5?logo=keepassxc&logoColor=white) ![platform](https://img.shields.io/badge/platform-macOS%20%7C%20Android-lightgrey) ![sync](https://img.shields.io/badge/sync-Google%20Drive-4285F4?logo=googledrive&logoColor=white) ![license](https://img.shields.io/badge/license-CC%20BY%204.0-green)

[Setup Guide](#quick-start) · [SSH Agent](docs/SSH_AGENT.md) · [Secrets & Keychain](docs/SECRETS_AND_CONFIGS.md) · [TOTP & Passkeys](docs/TOTP_AND_PASSKEYS.md) · [Maintenance](docs/MAINTENANCE.md) · [FAQ](docs/FAQ.md)

---

## What this covers

- **Two separate encrypted vaults** — one for work credentials, one for personal
- **Two Mac machines** — both access the work vault; only Mac 1 accesses the personal vault (Mac 2 is work-only by design)
- **One Android phone** — accesses both vaults via Keepass2Android
- **Google Drive** as the sync layer — vault files live in your own Drive, encrypted at rest, no third-party password server
- **Browser autofill** via the KeePassXC Chrome extension across all Chrome profiles
- **SSH agent integration** — unlocking the vault automatically loads your SSH private keys
- **TOTP (2FA codes)** stored inside vault entries alongside passwords
- **Passkeys** stored and used directly via the KeePassXC browser extension
- **macOS Keychain bridge** — any terminal session including AI coding tools can fetch API secrets without copy-pasting

> **This guide contains no real names, org identifiers, or account IDs.** All examples use generic placeholders — replace them with your own values.

---

## Why this guide exists

Most password manager setups either:

- Use a third-party cloud (Bitwarden, 1Password, LastPass) — credential data lives on someone else's server
- Stay confined to one machine — no sync, no mobile access
- Handle SSH keys, API tokens, and passwords as three separate problems — more tools, more friction

This setup is different:

- **You own the data** — `.kdbx` files in your own Google Drive, encrypted with a Master Password only you know
- **Full device coverage** — work and personal credentials across multiple Macs and Android, all synced from one Drive account
- **One tool for everything** — web logins, SSH key references, 2FA codes, passkeys, and API token documentation all in KeePassXC
- **Terminal and AI ready** — secrets reach any shell session (including AI agent tools) via macOS Keychain, without copy-pasting

---

## Architecture

### Vault layout and sync

```
                 ┌──────────────────────────────────────────────────┐
                 │              Personal Google Drive                │
                 │                                                   │
                 │   My Drive / Synchronous Backup / KeePass backup │
                 │                                                   │
                 │  ┌──────────────────┐   ┌──────────────────────┐ │
                 │  │   Work_Vault/    │   │   Personal_vault/    │ │
                 │  │  work_vault.kdbx │   │ personal_vault.kdbx  │ │
                 │  │                  │   │                      │ │
                 │  │  ← shared with   │   │  ← private only      │ │
                 │  │    work Gmail    │   │    never shared      │ │
                 │  └────────┬─────────┘   └──────────┬───────────┘ │
                 └──────────┼────────────────────────┼──────────────┘
                            │   Google Drive sync     │
           ┌────────────────┼─────────────────────────┘
           │                │
  ┌────────┴──────────┐  ┌──┴────────────────┐    ┌──────────────────────┐
  │       Mac 1       │  │      Mac 2         │    │       Android        │
  │                   │  │   (work-only)      │    │                      │
  │  KeePassXC        │  │  KeePassXC         │    │  Keepass2Android     │
  │  ├ work_vault     │  │  └ work_vault      │    │  ├ work_vault        │
  │  └ personal_vault │  │                   │    │  └ personal_vault    │
  │                   │  │                   │    │                      │
  │  Chrome (work)    │  │  Chrome (work)    │    │  Android Autofill    │
  │  Chrome (personal)│  │                  │    │  (system autofill)   │
  │  SSH Agent        │  │  SSH Agent        │    │  Biometric unlock    │
  │  macOS Keychain   │  │  macOS Keychain   │    │                      │
  └───────────────────┘  └───────────────────┘    └──────────────────────┘
```

### Device access matrix

| Device | `work_vault` | `personal_vault` | Browser autofill | SSH agent | macOS Keychain |
|---|---|---|---|---|---|
| Mac 1 | ✓ | ✓ | Work + Personal Chrome profiles | ✓ | ✓ |
| Mac 2 | ✓ | ✗ by design | Work Chrome profile only | ✓ | ✓ |
| Android | ✓ | ✓ | System autofill service | ✗ | ✗ |

### Secrets management flow

```
  ┌────────────────────────────────────────────────────────────────────┐
  │                          KeePassXC                                 │
  │                       (master store)                               │
  │                                                                    │
  │  Encrypted .kdbx · Synced via Google Drive · Android-accessible   │
  │  Passwords · SSH key paths · API tokens · TOTP codes · Passkeys   │
  └────────────────────────────────┬───────────────────────────────────┘
                                   │
                       Store token once per Mac
                       (printf ... ; read -s t &&
                        security add-generic-password ...)
                                   │
                                   ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │                        macOS Keychain                              │
  │                      (terminal bridge)                             │
  │                                                                    │
  │  • Auto-unlocks on Mac login — no password or Touch ID required   │
  │  • Local to each Mac — not synced, not available on Android       │
  │  • Any process reads via: security find-generic-password          │
  └──────────────────────────────┬─────────────────────────────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
              ▼                                     ▼
  ┌──────────────────────────┐       ┌───────────────────────────────┐
  │      Your terminal       │       │    AI coding tools            │
  │                          │       │    (any agent, any session)   │
  │  export MY_TOKEN=$(      │       │                               │
  │    security find-        │       │  Runs the exact same command  │
  │    generic-password      │       │  in its own shell session —   │
  │    -s "my-service" -w    │       │  no copy-paste, no user       │
  │  )                       │       │  interaction needed           │
  └──────────────────────────┘       └───────────────────────────────┘
```

---

## Tools & What They Do

| Tool | Purpose |
|---|---|
| **KeePassXC** | Offline, open-source password manager. Stores credentials in an encrypted `.kdbx` file you own. |
| **Google Drive for Desktop** | Mounts your Drive as a local folder in Finder — the sync layer for `.kdbx` files across Macs. |
| **KeePassXC Browser Extension** | Chrome extension that autofills credentials from KeePassXC via a local socket — no internet involved. |
| **Keepass2Android** | Android KeePass client. Connects directly to Google Drive. Supports biometric unlock and simultaneous multiple databases. |
| **macOS Keychain** | System-level encrypted store that auto-unlocks on login. Used as a terminal bridge for API tokens. |
| **Homebrew** | macOS package manager. Installs KeePassXC with one command. |

---

## Vault Group Structure

**`work_vault.kdbx`:**
```
work_vault
├── Infrastructure Logins
│    ├── Passwords       ← web logins, cloud consoles, admin portals
│    └── Passkeys        ← passkeys for work accounts
├── SSH Identities       ← at vault root level
└── Secrets & Configs    ← API tokens, connection strings, .env contents
```

**`personal_vault.kdbx`:**
```
personal_vault
├── Personal Logins
│    └── Login
│         ├── Websites   ← personal web logins
│         └── Passkeys   ← passkeys for personal accounts
├── SSH Identities       ← at vault root level
└── Secrets & Configs    ← personal API keys, tokens
```

---

## Directory Layout

### Google Drive (cloud — synced to both Macs)

```
[Personal Gmail — Google Drive]
 └── My Drive
      └── Synchronous Backup
           └── KeePass backup
                ├── Work_Vault/                  ← shared with work Gmail (Editor access)
                │    └── work_vault.kdbx
                └── Personal_vault/              ← personal Gmail only, never shared
                     └── personal_vault.kdbx
```

### Local Mac (on each machine — not synced)

```
~/.ssh/
 ├── <key-filename>                  ← Private key (chmod 600)
 ├── <key-filename>.pub              ← Public key
 ├── config
 └── agent/
      └── ssh-agent.sock             ← Persistent agent socket (Mac 2 only)

~/Library/Keychains/
 └── login.keychain-db               ← macOS Keychain — stores API tokens for terminal/AI access
```

---

## Quick Start

> Full step-by-step instructions with all options and notes are in [docs/SETUP_GUIDE.md](docs/SETUP_GUIDE.md).

**High level order:**

1. Install Google Drive for Desktop on Mac 1 — sign in with personal Gmail
2. Create folder structure: `My Drive > Synchronous Backup > KeePass backup > Work_Vault`
3. Share `Work_Vault/` with work Gmail (Editor access)
4. Install KeePassXC on Mac 1 — `brew install --cask keepassxc`
5. Create `work_vault.kdbx` inside `Work_Vault/` — set Master Password — create group structure
6. Harden KeePassXC settings (auto-lock 5 min, lock on lid close)
7. Install KeePassXC Browser extension — pair with work Chrome profile as `Mac1-Chrome`
8. Set up Mac 2: install Drive for Desktop (work Gmail) — surface `Work_Vault` in Finder — install KeePassXC — open existing vault — pair extension as `Mac2-Chrome`
9. Migrate passwords from Chrome or another password manager
10. Set up Android: install Keepass2Android — connect via system file picker
11. Create `personal_vault.kdbx` in `Personal_vault/` on Mac 1 — pair personal Chrome profile
12. Set up [SSH agent integration](docs/SSH_AGENT.md) — both Macs
13. Set up [Secrets & Configs](docs/SECRETS_AND_CONFIGS.md) — store API tokens in KeePassXC + macOS Keychain
14. Set up [TOTP and Passkeys](docs/TOTP_AND_PASSKEYS.md) for applicable accounts

---

## Docs Index

| Document | What it covers |
|---|---|
| [SETUP_GUIDE.md](docs/SETUP_GUIDE.md) | Complete step-by-step setup — Parts 1, 2, 3 (Steps 1–23), security notes, progress checklist |
| [SSH_AGENT.md](docs/SSH_AGENT.md) | SSH agent integration — vault entries, Mac 1 (Prezto) and Mac 2 (manual) agent setup |
| [SECRETS_AND_CONFIGS.md](docs/SECRETS_AND_CONFIGS.md) | Secrets management — KeePassXC + macOS Keychain bridge, notes template, step-by-step, revocation |
| [TOTP_AND_PASSKEYS.md](docs/TOTP_AND_PASSKEYS.md) | TOTP setup and usage, passkeys setup and usage |
| [MAINTENANCE.md](docs/MAINTENANCE.md) | Change Master Password, rename Drive directories, add new SSH keys |
| [FAQ.md](docs/FAQ.md) | Answers to common questions about the setup |

---

## Contributing

Improvements, corrections, and additions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to contribute.

---

## Security

For security notes about this setup — Master Password handling, vault file storage, Keychain security — see [SECURITY.md](SECURITY.md).

---

## License

This guide is published under [CC BY 4.0](LICENSE) — free to share and adapt with attribution.
