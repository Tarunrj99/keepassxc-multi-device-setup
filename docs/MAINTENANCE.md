# Maintenance — KeePassXC Multi-Device Setup

How to perform common maintenance tasks: rotating master passwords, renaming Drive directories, and adding new SSH keys.

[← Back to README](../README.md)

---

## Table of Contents

- [How to Change the Master Password](#how-to-change-the-master-password)
- [How to Rename the Google Drive Directory](#how-to-rename-the-google-drive-directory)
- [How to Add a New SSH Key in the Future](#how-to-add-a-new-ssh-key-in-the-future)
- [How to Rotate an API Token](#how-to-rotate-an-api-token)
- [How to Secure Credential Files](#how-to-secure-credential-files)
- [How to Encrypt a Whole Directory](#how-to-encrypt-a-whole-directory)

---

## How to Change the Master Password

Do this when you want to rotate either vault's password. The steps are the same for both vaults.

1. Open KeePassXC → unlock the vault you want to change
2. Top menu → **Database** → **Database settings**
3. Click the **Security** tab
4. Under **Password** → click **Change password**
5. Enter the current Master Password, then the new one twice
6. Click **OK** → `Ctrl + S`

**Impact on other devices:**

| Device / Component | What happens |
|---|---|
| Other Mac | Next vault open will prompt for the new password |
| Android (Keepass2Android) | Asks for the new password — biometric unlock resets and must be re-enabled |
| Old password | Stops working immediately after saving |
| Vault contents | Completely untouched — nothing inside changes |
| SSH keys loaded via agent | Unaffected — key files on disk are unchanged |
| macOS Keychain | Unaffected — Keychain entries are independent of the vault password |

> The Master Password is not stored anywhere in the file. Changing it re-encrypts the entire file with the new password. The old password becomes invalid immediately.

---

## How to Rename the Google Drive Directory

You may want to rename `Work_Vault` or the parent folders. Here is how to do it safely without breaking KeePassXC's connection to the vault file.

**Important:** KeePassXC remembers the path to the `.kdbx` file. If the folder name changes, the remembered path becomes stale — you need to re-open the file from the new location after renaming.

**Steps:**

1. **Close KeePassXC on both Macs** — make sure neither Mac has the vault open
2. **Rename on drive.google.com:**
   - Go to drive.google.com (personal Gmail account)
   - Right-click the folder → **Rename**
   - The rename syncs automatically to Finder on both Macs via Drive for Desktop
3. **Re-open on Mac 1:**
   - Open KeePassXC
   - Top menu → **Database** → **Open database**
   - Navigate to the renamed folder → select `work_vault.kdbx`
   - Enter Master Password
4. **Repeat on Mac 2** — same process
5. **On Android:**
   - Keepass2Android stores the Drive file reference by file ID, not folder name, so it may still work automatically
   - Open Keepass2Android and try to unlock — if it fails, use the system file picker to re-navigate to the file from the new folder path

> If you rename a parent folder (e.g. rename `KeePass backup`), the same steps apply — just navigate to the new path when re-opening.

---

## How to Add a New SSH Key in the Future

When you generate a new SSH key pair or receive a new key for a new server, add it to the vault following these steps. The process is the same as the initial [SSH Agent Setup](SSH_AGENT.md).

### Step 1 — Generate or place the key file

**Generate a new key:**

```bash
ssh-keygen -t ed25519 -C "description of what this key is for"
# When prompted, save to ~/.ssh/<descriptive-name>
# This creates a private key file and a .pub public key file
```

**Received a key file from someone:**

```bash
# Place it in ~/.ssh/ and set the correct permissions
chmod 600 ~/.ssh/<key-filename>
```

### Step 2 — Decide which vault it belongs to

| Key for | Vault |
|---|---|
| Work server, production, staging | `work_vault → SSH Identities` |
| Personal server, GitHub, home lab | `personal_vault → SSH Identities` |

### Step 3 — Create the vault entry

Follow the same process from [SSH_AGENT.md — How to create a vault entry](SSH_AGENT.md#how-to-create-a-vault-entry):

1. Open KeePassXC → click the correct vault tab
2. Left sidebar → click `SSH Identities`
3. Press `Ctrl + N`
4. **General tab:** title (`SSH Key - <server-name>`), username (SSH user)
5. **Notes tab:** key path (`/Users/<your-username>/.ssh/<key-filename>`), port, purpose
6. **SSH Agent tab:** External File → enter full key path → tick both checkboxes
7. Click **OK** → `Ctrl + S`

### Step 4 — Verify

Lock and unlock the vault, then run:

```bash
ssh-add -l
# The new key should appear in the list
```

### Step 5 — Copy to Mac 2 if needed

If the key needs to be on Mac 2 as well (use `scp` or USB — never email or Drive):

```bash
scp ~/.ssh/<key-filename> <mac2-username>@<mac2-ip>:~/.ssh/<key-filename>
ssh <mac2-username>@<mac2-ip> "chmod 600 ~/.ssh/<key-filename>"
```

Then create the same vault entry on Mac 2 following Steps 2–4 above — the `work_vault` file is shared, so the entry already exists on Mac 2's vault, but you still need the key file on disk for it to load.

---

## How to Rotate an API Token

When a service rotates your API token or you want to update it manually:

1. **Generate a new token** at the service (Cloudflare, GitHub, AWS, etc.)
2. **Update KeePassXC:**
   - Open the entry in `Secrets & Configs` → right-click → **Edit Entry**
   - Replace the Password field value with the new token
   - `Ctrl + S` to save — the updated entry syncs to both Macs via Drive
3. **Update macOS Keychain on each Mac** — run the store command from the entry's Notes field:
   ```bash
   printf 'Paste token (hidden): '; read -s t && echo && security add-generic-password -s "<keychain-service-name>" -a "<ENV_VAR_NAME>" -w "$t" -U && unset t && echo "Done — stored in Keychain"
   ```
   The `-U` flag updates the existing Keychain entry — it does not fail if it already exists.

4. **Revoke the old token** at the service — do this after confirming the new token works.

---

## How to Secure Credential Files

KeePassXC supports **file attachments** directly inside entries. Files are stored encrypted inside the `.kdbx` — synced to all devices via Google Drive automatically.

**Attach a file:**

1. KeePassXC → find or create the entry for that credential
2. Right-click → **Edit Entry**
3. Click the **Advanced** tab
4. Under **Attachments** → click **Add** → select the file
5. Click **OK** → `Ctrl + S`

**Use the file later:**

1. Right-click the entry → **Edit Entry** → **Advanced** tab
2. Select the attachment → click **Save** → choose a temporary location
3. Use the file
4. **Delete the exported copy when done** — the exported file is unencrypted on disk. The encrypted original stays safely inside KeePassXC.

**Good for:**
- GCP service account `.json` key files
- `.pem` / `.crt` certificates
- `.env` files for a project
- PDF documents containing credentials
- Any config file with sensitive content

---

## How to Encrypt a Whole Directory

For a larger collection of files, use an **encrypted disk image** via `hdiutil` — built into macOS, no extra tools needed.

**Create an encrypted disk image (one-time setup):**

```bash
# Creates a 500MB AES-256 encrypted .dmg — adjust -size as needed
hdiutil create -size 500m \
  -encryption AES-256 \
  -fs HFS+ \
  -volname "SecureFiles" \
  ~/Documents/secure-files.dmg
```

When prompted, enter a strong password. Store it in KeePassXC as a new entry (e.g. `Secret - Secure Files Disk Image`).

**Mount when you need to access the files:**

```bash
hdiutil attach ~/Documents/secure-files.dmg
# Enter password → mounts as /Volumes/SecureFiles/ in Finder
```

**Unmount when done:**

```bash
hdiutil detach /Volumes/SecureFiles
```

**Where to store the `.dmg`:**
- Keep it in your Google Drive folder — it syncs to both Macs automatically
- The file is fully encrypted — even if someone gets the `.dmg`, they cannot open it without the password
- Mount it on Mac 2 using the same `hdiutil attach` command

**When to use which approach:**

| Situation | Best approach |
|---|---|
| One or a few files tied to a specific account | KeePassXC attachment — simpler, already in your workflow |
| Many files or a whole project's secrets directory | Encrypted disk image (`.dmg`) |
| Files you access frequently | Disk image — can stay mounted while you work |
| Files you rarely need | KeePassXC attachment — export when needed, delete after use |
