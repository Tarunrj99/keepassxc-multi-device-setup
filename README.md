# KeePassXC Multi-Device Setup Guide

**A practical, self-hosted password manager setup across multiple Macs and Android.**  
One encrypted vault file per context. No third-party password cloud. Google Drive for sync. Works with browser autofill, SSH agent, TOTP, passkeys, and AI tools.

_Two vaults. Any number of devices. One Google Drive account._

![keepassxc](https://img.shields.io/badge/KeePassXC-2.7.7+-5085A5?logo=keepassxc&logoColor=white) ![platform](https://img.shields.io/badge/platform-macOS%20%7C%20Android-lightgrey) ![sync](https://img.shields.io/badge/sync-Google%20Drive-4285F4?logo=googledrive&logoColor=white) ![license](https://img.shields.io/badge/license-CC%20BY%204.0-green)

[What this covers](#what-this-covers) · [Why this exists](#why-this-guide-exists) · [Architecture](#architecture) · [Tools](#tools--what-they-do) · [Work vault setup](#part-1-work-vault-setup) · [Android](#part-2-android-setup) · [Personal vault](#part-3-personal-vault-setup) · [SSH + Secrets + TOTP](#feature-steps) · [FAQ](#frequently-asked-questions)

---

## What this covers

This guide documents a complete self-hosted password management setup:

- **Two separate encrypted vaults** — one for work credentials, one for personal
- **Two Mac machines** — both access the work vault; only Mac 1 accesses the personal vault (Mac 2 is work-only by design)
- **One Android phone** — accesses both vaults via Keepass2Android
- **Google Drive** as the sync layer — vault files live in your own Drive, encrypted at rest, no third-party password server involved
- **Browser autofill** via the KeePassXC Chrome extension across all Chrome profiles
- **SSH agent integration** — unlocking the vault automatically loads your SSH private keys into memory
- **TOTP (2FA codes)** stored inside vault entries alongside passwords — no separate authenticator app needed
- **Passkeys** stored and used directly via the KeePassXC browser extension
- **macOS Keychain bridge** — any terminal session (including AI coding tools) can fetch API secrets without copy-pasting

> **This guide contains no real names, org identifiers, or account IDs.** All examples use generic placeholders. Replace them with your own values.

---

## Why this guide exists

Most password manager setups either:

- Use a third-party cloud service (Bitwarden, 1Password, LastPass) — credential data lives on someone else's server
- Stay confined to one machine — no sync, no mobile, no team access
- Handle SSH keys, API tokens, and passwords as three separate problems — more tools, more friction

This setup is different:

- **You own the data** — `.kdbx` files live in your own Google Drive, encrypted with a Master Password only you know
- **Full device coverage** — work and personal credentials across multiple Macs and Android, all synced from one Drive account
- **One tool for everything** — web logins, SSH key references, 2FA codes, passkeys, and API token documentation all live in KeePassXC
- **Terminal and AI ready** — secrets reach any shell session (including AI agent tools) via macOS Keychain, no manual copy-paste required

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
  │      Your terminal       │       │    AI / Agent tools           │
  │                          │       │    (Cursor, Claude, etc.)     │
  │  export MY_TOKEN=$(      │       │                               │
  │    security find-        │       │  Runs the exact same command  │
  │    generic-password      │       │  in its own shell session —   │
  │    -s "my-service" -w    │       │  no copy-paste, no user       │
  │  )                       │       │  interaction needed           │
  └──────────────────────────┘       └───────────────────────────────┘
```

---

## Tools & What They Do

Before the steps, here is what we are working with and why each tool matters.

### Google Drive for Desktop

The desktop app that mounts your Google Drive as a folder in Mac Finder. Without it, Drive files only exist in the browser. Once installed, you get a real local path like `Google Drive > My Drive > ...` that KeePassXC can read and write as a normal folder. This is the sync layer — both Macs mount the same Drive folder, and Drive keeps the vault file up to date automatically.

### KeePassXC

An offline, open-source password manager. Credentials are stored in an encrypted `.kdbx` database file that lives on your own disk (or in your Drive folder). The file is just a file — you control it entirely.

**`work_vault.kdbx` (work vault) group structure:**
```
work_vault
├── Infrastructure Logins
│    ├── Passwords       ← web logins, cloud consoles, admin portals
│    └── Passkeys        ← passkeys for work accounts
├── SSH Identities       ← server access key paths (at vault root level)
└── Secrets & Configs    ← API tokens, connection strings, .env contents
```

**`personal_vault.kdbx` (personal vault) group structure:**
```
personal_vault
├── Personal Logins
│    └── Login
│         ├── Websites   ← personal web logins
│         └── Passkeys   ← passkeys for personal accounts
├── SSH Identities       ← personal server access (at vault root level)
└── Secrets & Configs    ← personal API keys, tokens
```

### KeePassXC Browser Extension

A Chrome extension that talks directly to the KeePassXC desktop app over a local socket — no internet involved. When you land on a login page, the extension asks KeePassXC for matching credentials and fills the form automatically.

### Keepass2Android

The Android version of KeePass. It connects to Google Drive directly through the Android system file picker — no separate Drive app needed. Supports biometric unlock (fingerprint or face) and can have multiple databases open simultaneously.

### Homebrew

macOS package manager. Used to install KeePassXC cleanly with a single command.

---

## Directory Layout

### Google Drive (cloud — synced to both Macs)

```
[Personal Gmail — Google Drive]
 └── My Drive
      └── Synchronous Backup
           └── KeePass backup
                ├── Work_Vault/                  ← shared with work Gmail (Editor access)
                │    └── work_vault.kdbx          ← work credentials vault
                └── Personal_vault/              ← personal Gmail only, never shared
                     └── personal_vault.kdbx      ← personal credentials vault
```

Only `Work_Vault/` is visible to the work Gmail. `Personal_vault/` and everything above it stays private to the personal account.

### Local Mac (on each machine — not synced)

```
~/.ssh/                              ← SSH private and public key files
 ├── <key-filename>                  ← Private key (chmod 600, never share)
 ├── <key-filename>.pub              ← Public key (safe to share)
 ├── config                          ← SSH config file (optional)
 └── agent/
      └── ssh-agent.sock             ← Persistent agent socket (Mac 2 only)

~/Library/Keychains/
 └── login.keychain-db               ← macOS Keychain (auto-unlocks on login)
                                        Stores API tokens for terminal / AI access
                                        Not synced — must be set up on each Mac
```

> SSH key files stay in `~/.ssh/` on each Mac. KeePassXC stores only the file path — not the key content itself. The Keychain is managed via the `security` command — see **Step B** for exact commands.

---

## Part 1: Work Vault Setup

### Step 1: Install Google Drive for Desktop (Mac 1)

Download and install: [https://www.google.com/drive/download](https://www.google.com/drive/download)

Sign in with your **personal Gmail** account. After installation, the Drive folder appears in Finder:

```
Finder sidebar > Google Drive > My Drive
```

---

### Step 2: Create the Folder Structure in Personal Drive

Inside personal Google Drive (via Finder or drive.google.com), create the hierarchy:

```
My Drive > Synchronous Backup > KeePass backup > Work_Vault
```

The nesting keeps vaults clearly organised under a backup directory, and isolates `Work_Vault` as the only thing ever shared externally.

---

### Step 3: Share Only the `Work_Vault` Folder with Work Gmail

1. Right-click the `Work_Vault` folder on drive.google.com
2. Share → add work Gmail address
3. Set permission: **Editor**

This is the security boundary. The work Gmail can read and write files inside `Work_Vault`, but cannot see anything outside it — not `KeePass backup`, not `Synchronous Backup`, nothing else in the personal Drive root.

---

### Step 4: Install KeePassXC on Mac 1

```bash
brew install --cask keepassxc
```

---

### Step 5: Create the Vault File

Launch KeePassXC → **Create new database**.

- **Database name:** `work_vault`
- **File name:** `work_vault.kdbx`
- **Save location:** `Google Drive > My Drive > Synchronous Backup > KeePass backup > Work_Vault`

Set a strong Master Password. This is the only thing that unlocks the vault.

**Create the group structure** (right-click root group → **Add group** for each level):

1. `Infrastructure Logins` — then two subgroups inside:
   - `Passwords` ← web logins, cloud consoles, admin portals
   - `Passkeys` ← passkeys for work accounts
2. `SSH Identities` ← at vault root level
3. `Secrets & Configs`

---

### Step 6: Harden Security Settings on Mac 1

`Cmd + ,` → configure:

- Lock after **300 seconds** (5 minutes) of inactivity
- Lock when the Mac **session locks**
- Lock when the **laptop lid is closed**

---

### Step 7: Set Up Browser Integration on Mac 1

1. KeePassXC Preferences → **Browser Integration** → enable for Chrome
2. Install **KeePassXC Browser** extension from the Chrome Web Store
3. Click **Connect** in the extension → name the connection `Mac1-Chrome`

This pairs the **work Gmail Chrome profile** with `work_vault`. The personal Chrome profile is paired separately in Step 19.

---

### Step 8: Set Up Mac 2 — Install Google Drive for Desktop

Same installation: [https://www.google.com/drive/download](https://www.google.com/drive/download)

Sign in with the **work Gmail** account — that account has access to the shared `Work_Vault` folder.

---

### Step 9: Surface the Shared Folder in Finder (Mac 2)

Shared folders only appear under "Shared with me" in the browser, not in the Drive desktop app automatically. Fix:

1. Open [drive.google.com](https://drive.google.com) → **Shared with me**
2. Find `Work_Vault`
3. Right-click → **Organize** → **Add shortcut to My Drive**
4. Place shortcut in **My Drive** root

After Drive for Desktop syncs, `Work_Vault` appears in Finder on Mac 2.

---

### Step 10: Install KeePassXC on Mac 2 and Open the Existing Vault

```bash
brew install --cask keepassxc
```

Select **Open existing database** → navigate to:

```
Google Drive > My Drive > Work_Vault > work_vault.kdbx
```

Enter the Master Password set in Step 5. Both Macs now point at the same encrypted file.

Apply the same auto-lock settings (Step 6). Enable Browser Integration and pair with `Mac2-Chrome`.

---

### Step 11: Migrate Passwords from Google Password Manager

**Export from Chrome:**
1. Chrome → `Settings > Autofill > Password Manager > Saved passwords`
2. Three-dot menu → **Export passwords**
3. Chrome saves `Chrome Passwords.csv`

**Import into KeePassXC:**
1. Top menu → **Database** → **Import** → **CSV file**
2. Map columns: `name` → Title, `url` → URL, `username` → Username, `password` → Password
3. Set destination group: `Infrastructure Logins/Passwords`

---

### Step 12: Clean Up and Verify

`Chrome Passwords.csv` is plain text — everyone with that file has all your passwords.

1. Delete `Chrome Passwords.csv` → **Empty Trash immediately**
2. Chrome → `Settings > Autofill > Password Manager` → turn off **Offer to save passwords**
3. Clear Chrome's saved password storage

Verify: navigate to a login page and confirm the extension autofills from the vault.

---

## Part 2: Android Setup

### Step 13: Install Keepass2Android on Android

Google Play Store → search **Keepass2Android Password Safe** (by Philipp Crocoll / Croco Apps).

> Install the regular version, not the "Offline" variant — the offline version cannot connect to cloud storage.

---

### Step 14: Connect to `work_vault.kdbx` on Android

There is a known bug with the native KeePass file browser on some Android versions. Use the system file picker instead:

1. Open Keepass2Android → **Open file...**
2. Select **System file picker** (native Android file icon)
3. Tap the menu icon (top-left) → select the **personal Google account**
4. Navigate to: `Synchronous Backup` → `KeePass backup` → `Work_Vault`
5. Tap `work_vault.kdbx`
6. Accept Google Drive permission prompt
7. Enter Master Password → **Unlock**

---

### Step 15: Enable Biometric Unlock on Android

After the vault opens for the first time, Keepass2Android offers biometric unlock. Accept — from now on, unlocking on Android only needs a fingerprint tap.

> The Master Password remains the actual protection. Biometrics is a convenience shortcut only.

---

### Step 16: Open `personal_vault.kdbx` on Android

> **Note on order:** This step is performed after the personal vault is created on Mac (Steps 17–18). It is documented here to keep all Android steps together.

1. Return to Keepass2Android's database list → **Open file...**
2. System file picker → personal Gmail → navigate to `Personal_vault`
3. Tap `personal_vault.kdbx` → accept permission → enter personal vault Master Password
4. Enable biometric unlock when prompted

Both vaults now appear in Keepass2Android's database list.

---

## Part 3: Personal Vault Setup

### Step 17: Create the `Personal_vault` Folder in Google Drive

In Finder, navigate to:

```
Google Drive > My Drive > Synchronous Backup > KeePass backup
```

Right-click → **New Folder** → name it `Personal_vault`. **Do not share this folder with anyone.**

---

### Step 18: Create `personal_vault.kdbx` in KeePassXC

1. Top menu → **Database** → **New database**
2. Name: `personal_vault`
3. Set a Master Password — **different from `work_vault`**
4. Save to `Google Drive > My Drive > Synchronous Backup > KeePass backup > Personal_vault`

**Create group structure** (right-click root → **Add group**):

1. `Personal Logins` → then subgroup `Login` → then two subgroups:
   - `Websites` ← personal web logins
   - `Passkeys` ← passkeys for personal accounts
2. `SSH Identities` ← at vault root level
3. `Secrets & Configs`

> Auto-lock settings (Step 6) apply to all databases in the same KeePassXC install — no need to configure again.

---

### Step 19: Pair Chrome Extension to Personal Vault (Mac 1, Personal Profile)

1. Confirm `personal_vault.kdbx` is open and unlocked
2. Switch to the **personal Gmail Chrome profile**
3. Install **KeePassXC Browser** from the Chrome Web Store
4. Click the extension → **Connect** → name the connection `Mac1-Chrome-Personal`

The personal Chrome profile now autofills from `personal_vault`. The work Chrome profile autofills from `work_vault`. Both use the same KeePassXC desktop app but are paired as separate connections.

> KeePassXC matches credentials by URL — entries go to the right vault automatically based on which one has the matching URL stored.

---

### Step 20: Personal Vault on Mac 2 (Skipped by Design)

Mac 2 is work-only. Personal credentials are not needed there. Only `work_vault.kdbx` is set up on Mac 2.

**If you ever need to add it in the future:**
- Drive desktop app → profile icon → **Add another account** → sign in with personal Gmail
- Open `personal_vault.kdbx` from the newly mounted personal Drive folder

---

### Step 21: Migrate Credentials from Bitwarden to `personal_vault.kdbx`

> **Export format note:** Bitwarden's JSON export maps more reliably into KeePassXC than CSV. Use JSON.

1. Log into [vault.bitwarden.com](https://vault.bitwarden.com) → **Tools** → **Export vault** → **JSON** format
2. Enter Bitwarden Master Password → download the file
3. KeePassXC → confirm `personal_vault` is the active tab
4. Top menu → **Database** → **Import** → **JSON file** → select the Bitwarden file
5. All entries land in `Personal Logins/Login/Websites`
6. **Delete the JSON file from Downloads immediately → empty Trash**

---

### Step 22: Enable Keepass2Android as System Autofill on Android

Android has a system-level Autofill Service that Keepass2Android plugs into — so it can autofill credentials in Chrome and any other app on the phone, without a browser extension.

**Enable:**
1. Android **Settings** → search "Autofill"
2. **Autofill service** → select **Keepass2Android**

**How it works:**
- Unlock the vault in Keepass2Android first
- Tap a username or password field in any app → Keepass2Android popup appears above the keyboard
- Authenticate with fingerprint → credentials fill automatically

**Disable Chrome's own password manager to avoid conflicts:**

Chrome → three-dot menu → **Settings** → **Password Manager** → turn off **Save passwords** and **Auto sign-in**

> **Banking sites:** Some banks deliberately block autofill on specific fields. This is a bank-side security decision. Workaround: long-press the username field in Keepass2Android → **Copy username** → paste manually. The password field will still autofill.

---

### Step 23: Configure KeePassXC Browser Extension Settings (All Profiles)

Right-click extension icon → **Options** — configure per profile:

**Settings for every profile:**

| Setting | Value |
|---|---|
| Always ask where to save new credentials | ON |
| Set as default password manager | ON |
| Enable passkeys | ON |
| Enable passkeys fallback | ON |
| Auto-submit login forms | OFF |
| Automatically fill single-credential entries | OFF |

**Mac 1 — Work Chrome profile (`work_vault`):**

| Setting | Value |
|---|---|
| Default group for saving new passwords | `Infrastructure Logins/Passwords` |
| Default group for saving new passkeys | `Infrastructure Logins/Passkeys` |

**Mac 1 — Personal Chrome profile (`personal_vault`):**

| Setting | Value |
|---|---|
| Default group for saving new passwords | `Personal Logins/Login/Websites` |
| Default group for saving new passkeys | `Personal Logins/Login/Passkeys` |

**Mac 2 — Work Chrome profile (`work_vault`):**

| Setting | Value |
|---|---|
| Default group for saving new passwords | `Infrastructure Logins/Passwords` |
| Default group for saving new passkeys | `Infrastructure Logins/Passkeys` |

> **Cross-Origin iframe prompt:** If the extension warns about a cross-origin iframe on a login page, click Yes only for sites you fully trust. KeePassXC remembers the choice per site.

---

## Setup Progress Checklist

Use this to track what has been completed on your own setup.

**Work vault (`work_vault.kdbx`)**

| Item | Done? |
|---|---|
| Mac 1 — Chrome extension paired and configured (Steps 7, 23) | |
| Mac 2 — Chrome extension paired and configured (Steps 10, 23) | |
| Google Drive sync — both Macs reading the same file | |
| Chrome password migration — CSV destroyed (Steps 11, 12) | |
| Android — `work_vault` connected, biometrics enabled (Steps 14, 15) | |
| `SSH Identities` group — keys loaded (Step A) | |
| TOTP — all work accounts set up (Step C) | |
| `Secrets & Configs` — API tokens stored and Keychain set up (Step B) | |

**Personal vault (`personal_vault.kdbx`)**

| Item | Done? |
|---|---|
| `Personal_vault/` folder and database created (Steps 17, 18) | |
| Chrome extension paired — personal profile (Steps 19, 23) | |
| Android — `personal_vault` connected, biometrics enabled (Step 16) | |
| Android autofill service enabled (Step 22) | |
| Previous password manager migrated (Step 21) | |
| Old password manager account deactivated | |
| `SSH Identities` group — keys loaded (Step A) | |
| TOTP — personal accounts set up (Step C) | |
| Passkeys — set up for applicable accounts (Step D) | |

---

## Important Notes & Security Reminders

**Master Password**
- Do not save the Master Password anywhere on your Mac — not in Notes, a text file, or another password manager on the same machine
- Do not store it inside KeePassXC itself — that is like locking a key inside the box it opens
- The only safe places are your memory or a physical note stored somewhere secure (not at your desk)
- If you forget the Master Password, the vault cannot be recovered — there is no reset option

**The `.kdbx` Files**
- Both vault files are encrypted at rest — even if someone gets the file, they cannot open it without the Master Password
- `work_vault.kdbx` — keep it only inside `Work_Vault/`. Moving it breaks the work Gmail's access
- `personal_vault.kdbx` — keep it only inside `Personal_vault/`. Never share this folder

**Export Files**
- Any exported credential file (CSV or JSON) is completely plain text — no encryption
- Delete immediately after import and empty the Trash right away

**Auto-lock**
- These are set to 5 minutes, but lock the vault manually before screen sharing — press `Ctrl + L` inside KeePassXC

---

## Feature Steps

These are ongoing-setup steps that are done once and then maintained over time.

### Step A: SSH Identities Setup

KeePassXC's built-in SSH Agent integration automatically loads your private SSH keys when you unlock the vault — so `ssh user@server` works without specifying a key path or typing a passphrase each time. Keys are unloaded when you lock the vault.

Both vaults have an `SSH Identities` group at the root level:
- **`work_vault`** → work servers (cloud VMs, staging, production environments)
- **`personal_vault`** → personal servers, GitHub, home lab

> **macOS Sequoia note:** KeePassXC's internal SSH agent socket has a compatibility issue on macOS Sequoia — `ssh-add -l` returns "communication with agent failed" even when the vault is unlocked. The fix is to point KeePassXC at an already-running SSH agent socket. How this is done differs between machines with and without a zsh framework like Prezto.

> **SSH keys are not stored inside the vault:** The SSH Agent tab uses **External File** — KeePassXC stores only the **path** to the private key file. When the vault unlocks, it reads the key from disk. If the key file does not exist at that path on a given machine, the entry silently does nothing on that machine. This is expected — the same vault file is shared but each machine has different key files on disk.

---

#### How to create a vault entry for an SSH key

First check which keys exist:

```bash
ls -la ~/.ssh/
```

Any file **without a `.pub` extension** is a private key. Skip `config`, `known_hosts`, and `known_hosts.old`.

For each key:

1. Open KeePassXC → click the correct vault tab
2. In the left sidebar, click the `SSH Identities` group
3. Press `Ctrl + N` to open a new entry

Fill in the entry:

**General tab:**
- **Title:** e.g. `SSH Key - GCP prod VM` or `SSH Key - GitHub`
- **Username:** the SSH user for that server, e.g. `ubuntu`, `ec2-user`
- **Password:** leave blank (unless the key file has a passphrase)
- **URL:** server hostname or IP (optional)

**Notes tab:**
```
Key: /Users/<your-username>/.ssh/<key-filename>
Port: 22
Purpose: what this server or service is used for
```

**SSH Agent tab:**
- Under **Private key** → select **External File**
- Enter the full path: `/Users/<your-username>/.ssh/<key-filename>`
- KeePassXC auto-fills Fingerprint, Comment, and Public Key — confirms the file was found
- Tick **Add key to agent when database is unlocked**
- Tick **Remove key from agent when database is locked**

Click **OK** → `Ctrl + S` to save.

---

#### Group placement — must be at vault root level

The `SSH Identities` group must sit at the **root level** of each vault — above all other groups, directly under the vault name.

- Right-click the vault root (topmost item in the left sidebar) → **Add group** → name it `SSH Identities`
- If the group exists but is inside another group: right-click `SSH Identities` → **Move group** → select vault root

Correct structure:

```
work_vault
├── Infrastructure Logins
│    ├── Passwords
│    └── Passkeys
├── SSH Identities       ← root level ✓
└── Secrets & Configs

personal_vault
├── Personal Logins
│    └── Login
│         ├── Websites
│         └── Passkeys
├── SSH Identities       ← root level ✓
└── Secrets & Configs
```

---

#### Mac 1 — SSH Agent Setup (with Prezto)

> This applies when Mac 1 uses Prezto (a zsh framework) which manages its own SSH agent. If you do not use Prezto, follow the Mac 2 approach instead.

**Issue:** KeePassXC's internal socket causes "communication with agent failed" on macOS Sequoia.

**Fix:** Point KeePassXC at the Prezto-managed socket.

**Step 1 — Identify the active socket:**

```bash
ssh-add -l
echo $SSH_AUTH_SOCK
cat ~/.zshrc | grep prezto
```

Prezto's socket is at: `/Users/<your-username>/.cache/prezto/ssh-agent.sock`

**Step 2 — Set the override in KeePassXC:**

1. `Cmd + ,` → **SSH Agent**
2. Toggle **Enable SSH Agent** ON
3. Set **SSH_AUTH_SOCK override** to:
   ```
   /Users/<your-username>/.cache/prezto/ssh-agent.sock
   ```
4. Click **OK** → `Cmd + Q` → reopen KeePassXC → unlock vault

**Step 3 — Create entries, verify:**

```bash
# Vault locked
ssh-add -l
# The agent has no identities.

# Vault unlocked
ssh-add -l
# (lists all keys configured in SSH Identities)
```

---

#### Mac 2 — SSH Agent Setup (without Prezto)

Mac 2 has no Prezto — a persistent agent with a fixed socket path is set up instead.

**Step 1 — Check current state:**

```bash
ls -la ~/.ssh/
ssh-add -l
echo $SSH_AUTH_SOCK
```

**Step 2 — Set up a persistent SSH agent with a fixed socket:**

```bash
cat >> ~/.zshrc << 'EOF'

# SSH agent — fixed socket for KeePassXC integration
export SSH_AUTH_SOCK=~/.ssh/agent/ssh-agent.sock
if [ ! -S "$SSH_AUTH_SOCK" ]; then
    ssh-agent -a "$SSH_AUTH_SOCK" > /dev/null 2>&1
fi
EOF
```

```bash
mkdir -p ~/.ssh/agent
source ~/.zshrc
```

Verify:

```bash
echo $SSH_AUTH_SOCK
# /Users/<your-username>/.ssh/agent/ssh-agent.sock

ssh-add -l
# The agent has no identities.   ← socket works, no keys loaded yet — correct
```

**Step 3 — Set the override in KeePassXC:**

1. `Cmd + ,` → SSH Agent → Enable SSH Agent ON
2. **SSH_AUTH_SOCK override:**
   ```
   /Users/<your-username>/.ssh/agent/ssh-agent.sock
   ```
3. Click **OK** → `Cmd + Q` → reopen → unlock vault

**Step 4 — Create entries and verify:**

```bash
# Vault locked
ssh-add -l
# The agent has no identities.

# Vault unlocked
ssh-add -l
# (lists all keys configured in SSH Identities)

# Vault locked again
ssh-add -l
# The agent has no identities.
```

---

### Step B: Populate `Secrets & Configs` Group

Both vaults have a `Secrets & Configs` group:
- **`work_vault`** → work API tokens, database connection strings, work `.env` files
- **`personal_vault`** → personal API keys, tokens, personal project configs

---

#### How secrets are stored and used — the full picture

KeePassXC is the **master store** — encrypted, synced via Google Drive, accessible on all devices. However, KeePassXC only lets you manually copy a value. For terminal and AI access, a second layer is used: **macOS Keychain**.

| Layer | What it is | What it solves |
|---|---|---|
| KeePassXC | Encrypted `.kdbx`, synced via Drive | Master store — secure, backed up, works on Android |
| macOS Keychain | System-level encrypted store, auto-unlocks on login | Terminal access — any session (yours or AI) can read without a password |

They serve different purposes. KeePassXC is where secrets are managed. Keychain is how secrets reach the terminal.

---

#### Why macOS Keychain works for AI tools

When an AI coding tool (Cursor, Claude, etc.) runs a terminal command, it opens a **fresh shell session** separate from your terminal. It cannot inherit variables you set manually.

macOS Keychain solves this because it is a **system daemon** — auto-unlocks when you log in to your Mac. Every terminal — yours, the AI's, any process — can read from it:

```bash
# Any terminal, any session, no password, no Touch ID needed
export MY_TOKEN=$(security find-generic-password -s "my-service-name" -w)
```

---

#### What to put in the KeePassXC Notes field

Always fill the Notes field of a secret entry with the exact Keychain commands for that secret. This way you never have to remember the command — open KeePassXC, read Notes, run the command.

**Notes field template (copy this for every secret entry):**

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

---

#### Step-by-step — adding a new secret

**Part 1 — Add to KeePassXC (master store)**

1. Open KeePassXC → click the correct vault tab
2. Left sidebar → click `Secrets & Configs`
3. Press `Ctrl + N`
4. Fill in:
   - **Title:** `Secret - <Service> <Key Type>` e.g. `Secret - Cloudflare API Key`
   - **Username:** the environment variable name e.g. `CF_API_TOKEN`
   - **Password field:** paste the actual secret value — encrypted in the vault
   - **Notes:** paste the template above, filled in for this specific secret
5. Click **OK** → `Ctrl + S`

**Part 2 — Add to macOS Keychain on each Mac**

Keychain is local — must be set up on **each Mac separately**.

> **Why not paste the value directly in the command?** Putting the token inline (e.g. `-w "actual-token"`) stores it in `~/.zsh_history`. Anyone who reads that file sees the token. The `printf` + `read -s` approach reads input silently — nothing is echoed, nothing lands in history. The `-U` flag handles both first-time setup and future rotations.

1. Open the KeePassXC entry → read the Notes field for the exact command
2. Run:

```bash
printf 'Paste token (hidden): '; read -s t && echo && security add-generic-password -s "my-service-name" -a "MY_ENV_VAR" -w "$t" -U && unset t && echo "Done — stored in Keychain"
```

3. When prompted — switch to KeePassXC → entry → click the **eye icon** to reveal the password → select text → `Cmd+C` → switch back to Terminal → paste → Enter
   > Use the eye icon + manual select instead of right-click Copy Password — the protected clipboard may not paste correctly in Terminal.
4. Terminal prints `Done — stored in Keychain`
5. Repeat on the other Mac

---

**Naming convention:**

| Field | Format | Example |
|---|---|---|
| Title | `Secret - <Service Name> <Key Type>` | `Secret - Cloudflare API Key` |
| Username | Environment variable name | `CF_API_TOKEN` |
| Password | The actual secret value | `(paste value here)` |
| Keychain service name | lowercase with hyphens | `cloudflare-api-key` |
| Notes | Template above, filled in | Keychain commands + rotation info |

---

**How to retrieve a secret manually:**
- Open KeePassXC → `Secrets & Configs` → find entry → right-click → **Copy Password**
- Value goes to clipboard — paste wherever needed

> **Known behaviour — Copy Password may not paste in Terminal:**
> KeePassXC's right-click → Copy Password uses a protected clipboard that auto-clears after 10 seconds. Some terminal apps do not accept content from this protected clipboard.
> **Workaround:** Entry → click **eye icon** → select text → `Cmd+C` → paste in Terminal. This uses the normal system clipboard and always works.

**How to use a secret in your terminal:**

```bash
export MY_TOKEN=$(security find-generic-password -s "my-service-name" -w)
echo $MY_TOKEN
```

---

**Optional — quick access via `.zshrc` (plain text, use with caution)**

```bash
export MY_TOKEN="your-actual-token-value"
```

| Acceptable for | Not acceptable for |
|---|---|
| Low-sensitivity dev / staging tokens | Production tokens with broad permissions |
| Tokens with limited permissions | Anything controlling real resources |
| Temporary use (remove it soon) | Tokens that could cause damage if exposed |

> `.zshrc` is a plain text file. Keychain is encrypted and protected by your macOS login — always the safer choice.

---

**Examples of entries to create:**

| Title | Username (variable) | Keychain service name | Used for |
|---|---|---|---|
| `Secret - Cloudflare API Key` | `CF_API_TOKEN` | `cloudflare-api-key` | DNS, Workers |
| `Secret - Datadog API Key` | `DD_API_KEY` | `datadog-api-key` | Monitoring, log forwarding |
| `Secret - MongoDB Connection String` | `MONGODB_URI` | `mongodb-uri` | App database |
| `Secret - Stripe Secret Key` | `STRIPE_SECRET_KEY` | `stripe-secret-key` | Payment processing |
| `Secret - GCP Service Account` | `GOOGLE_APPLICATION_CREDENTIALS` | `gcp-service-account` | GCP API access |
| `Secret - GitHub Token` | `GITHUB_TOKEN` | `github-token` | GitHub API, CI/CD |

---

#### Emergency — if a token is misused or needs to be revoked

**Immediate kill switch — revoke at the service:**

- **Cloudflare** → dash.cloudflare.com → My Profile → API Tokens → **Revoke**
- **GitHub** → Settings → Developer settings → Personal access tokens → **Delete**
- **AWS** → IAM → Users → Security credentials → **Deactivate / Delete**

Once revoked at the source, the token stops working everywhere — no matter who has a copy.

**Then clean up locally:**

```bash
security delete-generic-password -s "my-service-name"
source ~/.zshrc   # if you added it to .zshrc, remove that line first
```

**Then** generate a new token, store it in KeePassXC, and run the Keychain setup again.

---

### Step C: Enable TOTP

KeePassXC generates 6-digit TOTP codes for accounts with 2FA — keeping the password and the 2FA code in the same vault entry.

> **Important:** Once both password and TOTP are in the vault, the Master Password protects both factors. This makes it convenient but means Master Password security becomes even more critical.

---

**How to add TOTP to any account:**

**Part 1 — Get the secret key from the website**

1. Log into the account → **Settings → Security → Two-Factor Authentication**
2. Select **Authenticator App** as the 2FA method
3. When the QR code appears, look for a link like "Can't scan QR code?" or "Enter key manually"
4. Click it — a Base32 text key appears (e.g. `JBSWY3DPEHPK3PXP`)
5. Copy that key

**Part 2 — Find the matching entry in KeePassXC**

1. Open the correct vault
2. Navigate to the group where the login is stored
3. Right-click the entry → **TOTP → Set up TOTP**

**Part 3 — Configure**

1. Paste the Base32 key into the **Secret key** field
2. Select **Default TOTP Settings (RFC 6238)**
3. Click **OK** → `Ctrl + S`

**Part 4 — Verify**

1. The site asks you to enter the 6-digit code to confirm setup
2. In KeePassXC, right-click the entry → **TOTP → Copy TOTP**
3. Paste into the site's verification field (within 30 seconds)

**Using TOTP on subsequent logins:**

1. Log in — browser extension fills username and password
2. Site asks for 2FA code → right-click entry in KeePassXC → **TOTP → Copy TOTP**
3. Paste the 6-digit code

The code rotates every 30 seconds.

**Common accounts to set up TOTP for:**

- GitHub, GitLab
- AWS SSO / IAM
- Google Cloud Console
- Cloudflare
- MongoDB Atlas
- Azure / Microsoft 365
- Any service under Settings → Security → Two-Factor Authentication → Authenticator App

---

### Step D: Passkeys

Passkeys are a newer authentication standard replacing passwords on supported sites. KeePassXC added passkey support in version 2.7.7+.

**What a passkey is:** A cryptographic key pair generated per site. The private key stays in your vault, the site stores the public key. No password to type or steal.

**How passkeys work with KeePassXC:**

1. Check KeePassXC version: **About KeePassXC** → must be 2.7.7 or later
2. When a site offers "Create a passkey", the KeePassXC browser extension intercepts the prompt and asks which database to save it in
3. Passkey gets stored as an entry in the vault:
   - Work passkeys → `work_vault → Infrastructure Logins → Passkeys`
   - Personal passkeys → `personal_vault → Personal Logins → Login → Passkeys`
4. On next login, the site asks to use the passkey — the extension retrieves it from the vault automatically

> **Limitation:** Passkeys stored in KeePassXC only work on desktop via the browser extension. Keepass2Android does not currently support passkeys — on Android, use username + password from the vault for these sites instead.

---

## Future Maintenance Steps

### How to Change the Master Password

1. Open KeePassXC, unlock the vault
2. Top menu → **Database** → **Database settings** → **Security** tab
3. Under **Password** → **Change password**
4. Enter current password, then the new one twice
5. Click **OK** → `Ctrl + S`

**Impact on other devices:**

| What changes | Detail |
|---|---|
| Other Mac needs new password | Next vault open will prompt for the new password |
| Android needs new password | Keepass2Android asks for new password — biometric unlock resets and must be re-enabled |
| Old password stops working | Immediately after saving |
| Vault contents | Completely untouched |

---

### How to Rename the Google Drive Directory

1. **Close KeePassXC on both Macs first**
2. Rename the folder on drive.google.com (right-click → **Rename**) — syncs automatically
3. **Mac 1:** KeePassXC → **Database** → **Open database** → navigate to renamed folder → select `.kdbx` → enter Master Password
4. **Mac 2:** Same process
5. **Android:** Keepass2Android may still work (Drive tracks files by ID not name) — test it. If it fails, re-navigate using the system file picker

---

### How to Add a New SSH Key in the Future

**Step 1 — Generate or place the key file**

Generate new key:
```bash
ssh-keygen -t ed25519 -C "description of what this key is for"
# Save to ~/.ssh/<descriptive-name>
```

Received key from someone:
```bash
chmod 600 ~/.ssh/<key-filename>
```

**Step 2 — Decide which vault**
- Work server → `work_vault → SSH Identities`
- Personal server or GitHub → `personal_vault → SSH Identities`

**Step 3 — Create the vault entry** — follow the same process as [Step A: SSH Identities Setup](#step-a-ssh-identities-setup).

**Step 4 — Verify**

```bash
ssh-add -l   # new key should appear after vault unlock
```

**Step 5 — Copy to Mac 2 if needed** (use `scp` or USB — never email or Drive):

```bash
scp ~/.ssh/<key-filename> <mac2-username>@<mac2-ip>:~/.ssh/<key-filename>
ssh <mac2-username>@<mac2-ip> "chmod 600 ~/.ssh/<key-filename>"
```

---

## Frequently Asked Questions

**Q: Can I access Mac 1's SSH keys from Mac 2 since both open the same `work_vault`?**

No. The vault stores only the **path** to each key file. When `work_vault` unlocks on Mac 2, KeePassXC looks for key files at those paths on Mac 2's disk — but the files only exist on Mac 1. The entries silently do nothing on Mac 2.

To make Mac 1's keys available on Mac 2: copy the private key files via `scp` or USB. Once the files exist at the same paths on Mac 2, the existing vault entries load them automatically.

---

**Q: Are Secrets & Configs entries synced between both Macs?**

The **KeePassXC entries** are fully synced — the `.kdbx` file on Google Drive is the same file both Macs read. Save on one Mac, Drive syncs it, the other Mac picks it up.

The **macOS Keychain is not synced** — each Mac has its own Keychain. When you add a new secret to KeePassXC, run the `security add-generic-password` command on each Mac separately. The Notes field in each entry contains the exact command — just open the entry and run it on each machine.

---

**Q: What happens if I forget the Master Password?**

There is no recovery option. The vault is encrypted with the Master Password and KeePassXC does not store it anywhere. If forgotten, the vault cannot be opened.

Store the Master Password somewhere physically secure (written on paper, stored safely) — not in any digital note on the same machine.

---

**Q: What happens if the `.kdbx` file gets corrupted or accidentally deleted?**

Google Drive keeps version history. Right-click the file on drive.google.com → **Manage versions** (or navigate to Trash if deleted) → restore a previous version.

KeePassXC also creates a `.kdbx.bak` backup file in the same folder every time you save.

---

**Q: Can I open the same vault on two Macs at the same time?**

Yes, but only make edits on one at a time. If both Macs save simultaneously, KeePassXC detects the conflict and prompts you to choose which version to keep. Treat one Mac as the primary editing machine — let Drive sync before switching to the other.

---

**Q: Is it safe to store SSH keys and secrets in KeePassXC?**

The vault is encrypted with AES-256. SSH key files are not stored inside the vault — only path references are (the actual file stays in `~/.ssh/`). API tokens and passwords in the vault are only readable when unlocked with the Master Password.

The weakest point is the Master Password itself — a strong, unique Master Password is the most important security measure.

---

**Q: How do I actually use an SSH key to connect to a server?**

Once KeePassXC has loaded the key into the agent (vault unlocked), just use a normal `ssh` command:

```bash
ssh <username>@<server-ip-or-hostname>
```

No `-i ~/.ssh/<key-filename>` flag needed — the agent has the key in memory. Verify loaded keys before connecting:

```bash
ssh-add -l
```

---

**Q: If KeePassXC stores my secrets, how does the AI (Cursor, etc.) fetch them automatically?**

KeePassXC alone only allows manual copy. The solution is **macOS Keychain** — a system daemon that auto-unlocks on Mac login. Every terminal, including the AI's, can read from it:

```bash
export MY_TOKEN=$(security find-generic-password -s "my-service-name" -w)
```

The AI runs this in its own session — Keychain returns the value immediately with no user interaction.

---

**Q: Why both KeePassXC AND macOS Keychain? Why not just one?**

| Store | What it solves |
|---|---|
| KeePassXC | Master store — encrypted, synced via Drive, works on Android, survives Mac wipe |
| macOS Keychain | Terminal bridge — any session fetches values without user interaction |

You cannot use Keychain alone: it is local, does not sync, has no Android support, and has no proper management UI. You cannot use KeePassXC alone for terminal access: AI tools open fresh sessions that cannot inherit your clipboard or manual actions.

---

**Q: Does macOS Keychain protect values as securely as KeePassXC?**

| | KeePassXC | macOS Keychain |
|---|---|---|
| Encrypted | Yes | Yes |
| Requires unlock | Master Password or Touch ID | Auto-unlocks on Mac login |
| Synced across Macs | Yes (via Google Drive) | No — local to each Mac |
| Works on Android | Yes | No |
| If Mac is wiped | File survives on Drive | Gone permanently |
| Management UI | Full GUI | No good UI |

Keychain is convenient but not a replacement. KeePassXC is the source of truth.

---

**Q: When I copied a password using right-click → Copy Password, I could not paste it in Terminal. Why?**

KeePassXC's right-click → Copy Password writes to a **protected clipboard** that auto-clears after 10 seconds. Some terminal apps do not accept content from this protected clipboard.

**Workaround:** Entry → click **eye icon** to reveal the password → select text → `Cmd+C` → switch to Terminal → paste.

Optionally increase the timeout: `KeePassXC → Settings → Security → Clear clipboard after` → set to 30 seconds.

---

**Q: If a token is leaking, how do I stop it immediately?**

**Revoke at the service first** — this stops the token everywhere instantly:

- **Cloudflare** → API Tokens → **Revoke**
- **GitHub** → Personal access tokens → **Delete**
- **AWS** → IAM → Security credentials → **Deactivate / Delete**

Then clean up locally:

```bash
security delete-generic-password -s "my-service-name"
```

Generate a new token, store it in KeePassXC, run the Keychain setup again.

---

**Q: I generated a new SSH key — where do I store it and how do I secure it?**

No `mv` needed. `~/.ssh/` is already the correct location. Private keys stored there have `600` permissions — only you can read them.

1. Generate the key:
```bash
ssh-keygen -t ed25519 -C "purpose@machine" -f ~/.ssh/<key-filename>
```
2. Create a KeePassXC entry in the `SSH Identities` group — same process as Step A.

---

**Q: I have credential files (JSON, PDF, `.env`, `.pem`, text) — how do I secure them?**

KeePassXC supports **file attachments** directly inside entries. Files are stored encrypted inside the `.kdbx` — synced to all devices via Drive.

**Attach a file:**
1. KeePassXC → find or create the entry → right-click → **Edit Entry** → **Advanced** tab
2. Under **Attachments** → **Add** → select file
3. Click **OK** → `Ctrl + S`

**Use the file later:**
1. Right-click entry → **Edit Entry** → **Advanced** → select attachment → **Save** → choose a temporary location
2. Use the file
3. Delete the exported copy when done — it is unencrypted on disk once exported

Good for: GCP service account JSON files, `.pem` / `.crt` certificates, `.env` files, credential PDFs.

---

**Q: Can I encrypt a whole directory of files?**

Yes — using `hdiutil`, built into macOS. No extra tools needed.

**Create an encrypted disk image (one-time):**

```bash
# Creates a 500MB AES-256 encrypted .dmg
hdiutil create -size 500m \
  -encryption AES-256 \
  -fs HFS+ \
  -volname "SecureFiles" \
  ~/Documents/secure-files.dmg
```

Set a strong password — store it in KeePassXC as a new entry.

**Mount when needed:**

```bash
hdiutil attach ~/Documents/secure-files.dmg
# Enter password → mounts at /Volumes/SecureFiles/ in Finder
```

**Unmount when done:**

```bash
hdiutil detach /Volumes/SecureFiles
```

Store the `.dmg` in your Google Drive folder — it syncs to both Macs and is fully encrypted at rest.

**When to use which approach:**

| Situation | Best approach |
|---|---|
| One or a few files tied to a specific account | KeePassXC attachment — simpler |
| Many files or a whole project's secrets | Encrypted disk image (`.dmg`) |
| Files you access frequently | Disk image — stays mounted while working |
| Files you rarely need | KeePassXC attachment — export when needed, delete after |

---

**Q: When I share my screen, KeePassXC is not visible to the other person — is that normal?**

Yes. KeePassXC has **screen capture protection** which blocks the app window from appearing in screen shares, screenshots, and recordings.

To disable temporarily for a demo:
1. `Cmd + ,` → **Security** tab → uncheck **Enable screen capture protection**
2. Restart KeePassXC

Always re-enable and restart after the session.

---

**Q: Should I share my private SSH key with someone who asks for it?**

No. Never share a private key for any reason.

If someone needs server access:
1. They generate their own key pair: `ssh-keygen -t ed25519 -C "their-name@purpose"`
2. They send you their **public key** (`.pub` file — safe to share)
3. You add their public key to the server's `~/.ssh/authorized_keys`
4. They connect with their own private key

If you must transfer a shared service key, use `scp` on the same local network or a USB drive — never email, Slack, or Drive.

---

## Security Notes

- The Master Password is never stored anywhere by KeePassXC — losing it means losing access to the vault permanently
- Both vaults use AES-256 encryption with PBKDF2 key derivation by default
- The `.kdbx` file format is an open standard — the file can be opened by any compatible KeePass client
- macOS Keychain entries are encrypted by the system and protected by your macOS login password
- No part of this setup requires any third-party cloud password service

---

## License

This guide is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — you are free to share and adapt it for your own use, with attribution.
