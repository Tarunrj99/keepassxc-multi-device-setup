# Setup Guide — KeePassXC Multi-Device Setup

Complete step-by-step instructions for setting up KeePassXC across multiple Macs and Android.

[← Back to README](../README.md)

---

## Table of Contents

- [Part 1: Work Vault Setup (Mac 1, Mac 2)](#part-1-work-vault-setup)
  - [Step 1: Install Google Drive for Desktop](#step-1-install-google-drive-for-desktop-mac-1)
  - [Step 2: Create the Folder Structure](#step-2-create-the-folder-structure)
  - [Step 3: Share the Work Vault Folder](#step-3-share-only-the-work_vault-folder-with-work-gmail)
  - [Step 4: Install KeePassXC on Mac 1](#step-4-install-keepassxc-on-mac-1)
  - [Step 5: Create the Vault File](#step-5-create-the-vault-file)
  - [Step 6: Harden Security Settings](#step-6-harden-security-settings)
  - [Step 7: Set Up Browser Integration on Mac 1](#step-7-set-up-browser-integration-on-mac-1)
  - [Step 8: Set Up Mac 2 — Install Google Drive](#step-8-set-up-mac-2-install-google-drive-for-desktop)
  - [Step 9: Surface the Shared Folder in Finder](#step-9-surface-the-shared-folder-in-finder-mac-2)
  - [Step 10: Install KeePassXC on Mac 2](#step-10-install-keepassxc-on-mac-2-and-open-the-existing-vault)
  - [Step 11: Migrate Passwords from Chrome](#step-11-migrate-passwords-from-google-password-manager)
  - [Step 12: Clean Up and Verify](#step-12-clean-up-and-verify)
- [Part 2: Android Setup](#part-2-android-setup)
  - [Step 13: Install Keepass2Android](#step-13-install-keepass2android)
  - [Step 14: Connect work_vault on Android](#step-14-connect-work_vaultkdbx-on-android)
  - [Step 15: Enable Biometric Unlock](#step-15-enable-biometric-unlock)
  - [Step 16: Open personal_vault on Android](#step-16-open-personal_vaultkdbx-on-android)
- [Part 3: Personal Vault Setup](#part-3-personal-vault-setup)
  - [Step 17: Create the Personal Vault Folder](#step-17-create-the-personal_vault-folder-in-google-drive)
  - [Step 18: Create personal_vault.kdbx](#step-18-create-personal_vaultkdbx)
  - [Step 19: Pair Chrome Extension (Personal Profile)](#step-19-pair-chrome-extension-to-personal-vault)
  - [Step 20: Personal Vault on Mac 2 (Skipped)](#step-20-personal-vault-on-mac-2-skipped-by-design)
  - [Step 21: Migrate from Another Password Manager](#step-21-migrate-credentials-from-another-password-manager)
  - [Step 22: Enable Android Autofill](#step-22-enable-keepass2android-as-system-autofill)
  - [Step 23: Configure Browser Extension Settings](#step-23-configure-browser-extension-settings)
- [Setup Progress Checklist](#setup-progress-checklist)
- [Important Notes & Security Reminders](#important-notes--security-reminders)

---

## Part 1: Work Vault Setup

### Step 1: Install Google Drive for Desktop (Mac 1)

Download and install: [https://www.google.com/drive/download](https://www.google.com/drive/download)

Sign in with your **personal Gmail** account. After installation, the Drive folder appears in Finder:

```
Finder sidebar > Google Drive > My Drive
```

---

### Step 2: Create the Folder Structure

Inside personal Google Drive (via Finder or drive.google.com), create this hierarchy:

```
My Drive > Synchronous Backup > KeePass backup > Work_Vault
```

The nesting keeps vaults clearly organised under a backup directory, and isolates `Work_Vault` as the only thing ever shared externally.

---

### Step 3: Share Only the `Work_Vault` Folder with Work Gmail

1. Open [drive.google.com](https://drive.google.com) → navigate to `Work_Vault`
2. Right-click → **Share** → add your work Gmail address
3. Set permission: **Editor**

This is the security boundary. The work Gmail can read and write files inside `Work_Vault`, but cannot see anything above it — not `KeePass backup`, not `Synchronous Backup`, nothing else in your personal Drive root.

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

Set a strong Master Password. This is the only thing that unlocks the vault — KeePassXC does not store it anywhere.

**Create the group structure** — right-click root group in the left sidebar → **Add group** for each level:

1. `Infrastructure Logins` → then two subgroups inside:
   - `Passwords` — web logins, cloud consoles, admin portals
   - `Passkeys` — passkeys for work accounts
2. `SSH Identities` — at vault root level (not inside any other group)
3. `Secrets & Configs`

---

### Step 6: Harden Security Settings

`Cmd + ,` → configure:

| Setting | Value |
|---|---|
| Lock after inactivity | 300 seconds (5 minutes) |
| Lock on session lock | Enabled |
| Lock when lid is closed | Enabled |

This ensures the vault seals itself automatically if you step away or close the laptop.

---

### Step 7: Set Up Browser Integration on Mac 1

1. KeePassXC Preferences → **Browser Integration** tab → enable for Chrome
2. In Chrome (work profile), install **KeePassXC Browser** from the Chrome Web Store
3. Click the extension icon → **Connect**
4. Name the connection: `Mac1-Chrome`

This pairs the **work Gmail Chrome profile** with `work_vault`. The personal Chrome profile is paired separately in Step 19.

After pairing, the extension icon turns green when KeePassXC is open and unlocked.

---

### Step 8: Set Up Mac 2 — Install Google Drive for Desktop

Same installation: [https://www.google.com/drive/download](https://www.google.com/drive/download)

Sign in with the **work Gmail** account — that account has access to the shared `Work_Vault` folder.

---

### Step 9: Surface the Shared Folder in Finder (Mac 2)

Shared folders only appear under "Shared with me" in the browser, not in the Drive desktop app automatically.

**Fix:**

1. Open [drive.google.com](https://drive.google.com) on Mac 2 (logged in as work Gmail)
2. Click **Shared with me** in the left sidebar
3. Find `Work_Vault`
4. Right-click → **Organize** → **Add shortcut to My Drive** → place in **My Drive** root

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

Enter the same Master Password set in Step 5.

Apply the same security settings (Step 6). Enable Browser Integration and pair the Chrome extension as `Mac2-Chrome`.

---

### Step 11: Migrate Passwords from Google Password Manager

**Export from Chrome:**

1. Chrome → `Settings > Autofill > Password Manager > Saved passwords`
2. Three-dot menu → **Export passwords**
3. Chrome saves `Chrome Passwords.csv` to your desktop

**Import into KeePassXC:**

1. Top menu → **Database** → **Import** → **CSV file**
2. Select `Chrome Passwords.csv`
3. Map columns: `name` → Title, `url` → URL, `username` → Username, `password` → Password
4. Set destination group: `Infrastructure Logins/Passwords`

---

### Step 12: Clean Up and Verify

`Chrome Passwords.csv` is plain text — anyone who gets that file has all your passwords.

1. Delete `Chrome Passwords.csv` from desktop → **Empty Trash immediately**
2. Chrome → `Settings > Autofill > Password Manager` → turn off **Offer to save passwords**
3. Clear Chrome's existing saved passwords

**Verify:** Navigate to a login page and confirm the KeePassXC browser extension autofills credentials from the vault.

---

## Part 2: Android Setup

### Step 13: Install Keepass2Android

Google Play Store → **Keepass2Android Password Safe** (by Philipp Crocoll / Croco Apps).

> Install the regular version, **not** the "Offline" variant — the offline version cannot connect to cloud storage.

---

### Step 14: Connect `work_vault.kdbx` on Android

> There is a known bug with the native KeePass file browser on some Android versions. Use the system file picker instead.

1. Open Keepass2Android → **Open file...**
2. Select **System file picker** (native Android file icon — not the KeePass cloud icons)
3. Tap the menu icon (top-left three horizontal lines) → select the **personal Google account**
4. Navigate to: `Synchronous Backup` → `KeePass backup` → `Work_Vault`
5. Tap `work_vault.kdbx`
6. Google prompts for permission — tick the checkbox → **Continue**
7. Enter Master Password → **Unlock**

---

### Step 15: Enable Biometric Unlock

After the vault opens for the first time, Keepass2Android offers biometric unlock. Accept it — from now on unlocking only needs a fingerprint tap.

> The Master Password is still what actually protects the vault. Biometrics is a convenience shortcut only.

---

### Step 16: Open `personal_vault.kdbx` on Android

> **Note on order:** This step is done after the personal vault is created on Mac (Steps 17–18). It is documented here to keep all Android steps together.

1. Tap the back arrow in Keepass2Android → return to database list → **Open file...**
2. System file picker → personal Gmail account → navigate to `Personal_vault`
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

### Step 18: Create `personal_vault.kdbx`

1. KeePassXC → Top menu → **Database** → **New database**
2. Name: `personal_vault`
3. Leave encryption defaults (AES-256, PBKDF2) → click Continue
4. Set a Master Password — **different from `work_vault`**
5. Save to `Google Drive > My Drive > Synchronous Backup > KeePass backup > Personal_vault`

**Create group structure** — right-click root → **Add group**:

1. `Personal Logins` → subgroup `Login` → two subgroups inside:
   - `Websites` — personal web logins
   - `Passkeys` — passkeys for personal accounts
2. `SSH Identities` — at vault root level
3. `Secrets & Configs`

> Auto-lock settings (Step 6) apply at the application level — they cover all databases in the same KeePassXC install. No need to configure again.

---

### Step 19: Pair Chrome Extension to Personal Vault

Each Chrome profile is a separate environment and needs its own extension pairing.

1. Confirm `personal_vault.kdbx` is open and unlocked
2. Switch to the **personal Gmail Chrome profile**
3. Install **KeePassXC Browser** from the Chrome Web Store (separate install for this profile)
4. Click the extension → **Connect** → name the connection `Mac1-Chrome-Personal`

The personal Chrome profile now autofills from `personal_vault`. The work Chrome profile autofills from `work_vault`. Both use the same KeePassXC desktop app, paired as separate connections.

> KeePassXC matches credentials by URL — entries go to the right vault automatically based on which vault has the matching URL stored.

---

### Step 20: Personal Vault on Mac 2 (Skipped by Design)

Mac 2 is work-only. Personal credentials are not needed there. Only `work_vault.kdbx` is set up on Mac 2.

**If you ever add it in the future:**
- Drive desktop app menu bar icon → profile icon → **Add another account** → sign in with personal Gmail
- Then open `personal_vault.kdbx` from the newly mounted personal Drive folder in Finder

---

### Step 21: Migrate Credentials from Another Password Manager

**From Bitwarden (recommended: use JSON export, not CSV):**

1. Log into [vault.bitwarden.com](https://vault.bitwarden.com)
2. **Tools** → **Export vault** → **JSON** format → enter Master Password → download
3. KeePassXC → confirm `personal_vault` is the active tab
4. Top menu → **Database** → **Import** → **JSON file** → select the file
5. All entries land in `Personal Logins/Login/Websites`
6. **Delete the JSON file from Downloads immediately → empty Trash**

**From other managers:** use their CSV export and map columns during KeePassXC's import wizard. Bitwarden's CSV does not map cleanly — always use JSON from Bitwarden specifically.

---

### Step 22: Enable Keepass2Android as System Autofill

Android has a system-level Autofill Service that Keepass2Android plugs into — so it can autofill in Chrome and any other app, without a browser extension.

**Enable:**
1. Android **Settings** → search "Autofill"
2. **Autofill service** → select **Keepass2Android** → confirm

**How it works after enabling:**
- Unlock the vault in Keepass2Android first — it must be open in memory
- Tap a username or password field → Keepass2Android popup appears above the keyboard
- Authenticate with fingerprint → credentials fill automatically
- Works in Chrome, in apps, anywhere Android shows a login field
- Searches all currently unlocked vaults

**Disable Chrome's own password manager to avoid conflicts:**

Chrome → three-dot menu → **Settings** → **Password Manager** → turn off **Save passwords** and **Auto sign-in**

> **Banking sites:** Some banks block autofill on specific fields — most commonly the User ID field. This is a bank-side security decision, not a KeePass issue. Workaround: long-press the username field in Keepass2Android → **Copy username** → paste manually. The password field will still autofill normally.

---

### Step 23: Configure Browser Extension Settings

Right-click the KeePassXC extension icon → **Options** — configure per Chrome profile.

**Settings to apply in every profile:**

| Setting | Value | Why |
|---|---|---|
| Always ask where to save new credentials | ON | Prevents entries going to the wrong group |
| Set as default password manager | ON | Overrides Chrome's built-in |
| Enable passkeys | ON | Required for passkey support |
| Enable passkeys fallback | ON | Allows fallback to password if passkey fails |
| Auto-submit login forms | OFF | Security risk — leave disabled |
| Automatically fill single-credential entries | OFF | Security warning — leave disabled |

**Mac 1 — Work Chrome profile:**

| Setting | Value |
|---|---|
| Default group for new passwords | `Infrastructure Logins/Passwords` |
| Default group for new passkeys | `Infrastructure Logins/Passkeys` |

**Mac 1 — Personal Chrome profile:**

| Setting | Value |
|---|---|
| Default group for new passwords | `Personal Logins/Login/Websites` |
| Default group for new passkeys | `Personal Logins/Login/Passkeys` |

**Mac 2 — Work Chrome profile:**

| Setting | Value |
|---|---|
| Default group for new passwords | `Infrastructure Logins/Passwords` |
| Default group for new passkeys | `Infrastructure Logins/Passkeys` |

> **Cross-Origin iframe prompt:** If the extension warns about a cross-origin iframe on a login page, click Yes only for sites you fully trust and use regularly. KeePassXC remembers the choice per site.

---

## Setup Progress Checklist

Use this to track what is complete on your own setup.

**Work vault (`work_vault.kdbx`)**

| Item | Steps | Done? |
|---|---|---|
| Mac 1 — Chrome extension paired and configured | 7, 23 | |
| Mac 2 — Chrome extension paired and configured | 10, 23 | |
| Google Drive sync — both Macs reading the same file | 1–3, 8–9 | |
| Password migration — export file destroyed | 11, 12 | |
| Android — `work_vault` connected, biometrics enabled | 14, 15 | |
| SSH Identities group — keys loaded and verified | [SSH_AGENT.md](SSH_AGENT.md) | |
| TOTP — work accounts set up | [TOTP_AND_PASSKEYS.md](TOTP_AND_PASSKEYS.md) | |
| Secrets & Configs — tokens stored, Keychain configured | [SECRETS_AND_CONFIGS.md](SECRETS_AND_CONFIGS.md) | |

**Personal vault (`personal_vault.kdbx`)**

| Item | Steps | Done? |
|---|---|---|
| Folder and database created | 17, 18 | |
| Chrome extension paired — personal profile | 19, 23 | |
| Android — `personal_vault` connected, biometrics enabled | 16 | |
| Android autofill service enabled | 22 | |
| Previous password manager migrated, export destroyed | 21 | |
| Old password manager account deactivated | — | |
| SSH Identities group — keys loaded and verified | [SSH_AGENT.md](SSH_AGENT.md) | |
| TOTP — personal accounts set up | [TOTP_AND_PASSKEYS.md](TOTP_AND_PASSKEYS.md) | |
| Passkeys — set up for applicable accounts | [TOTP_AND_PASSKEYS.md](TOTP_AND_PASSKEYS.md) | |

---

## Important Notes & Security Reminders

**Master Password**

- Do not save the Master Password anywhere on your Mac — not in Notes, a text file, or another password manager on the same machine
- Do not store it inside KeePassXC itself — that would be like locking a key inside the box it opens
- The only safe places are your memory or a physical note stored somewhere secure (not at your desk)
- If you forget the Master Password, the vault **cannot be recovered** — there is no reset option

**The `.kdbx` Files**

- Both vault files are encrypted at rest — even if someone gets the file, they cannot open it without the Master Password
- `work_vault.kdbx` — keep it only inside `Work_Vault/`. Moving it breaks the work Gmail's access
- `personal_vault.kdbx` — keep it only inside `Personal_vault/`. Never share this folder

**Export Files (CSV or JSON)**

- Any exported credential file is completely plain text — no encryption at all
- Delete immediately after import and empty the Trash right away. Never leave it in Downloads

**Auto-lock**

- Set to 5 minutes, but lock the vault manually before screen sharing — press `Ctrl + L` inside KeePassXC

**Browser Extension**

- The extension only fills credentials when you actively trigger it — it does not read your browsing history or send data anywhere
- If the extension icon goes grey (disconnected), it means KeePassXC is not running or the vault is locked — open and unlock the app to reconnect

**Screen Capture Protection**

- KeePassXC has **screen capture protection** ON by default — the app window appears black in screen shares and recordings
- To disable temporarily for a demo: `Cmd + ,` → **Security** tab → uncheck → restart KeePassXC
- Always re-enable and restart after the session

**Simultaneous Editing**

- Avoid making changes on both Macs at the exact same time — if both save simultaneously, KeePassXC detects a conflict and prompts you to choose which version to keep
- Treat one Mac as the primary editing machine — let Drive sync before switching
