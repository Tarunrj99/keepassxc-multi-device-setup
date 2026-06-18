# SSH Agent Integration — KeePassXC

KeePassXC's built-in SSH Agent integration automatically loads your private SSH keys when you unlock the vault — `ssh user@server` in Terminal works without specifying a key path or typing a passphrase each time. Keys are unloaded when you lock the vault.

[← Back to README](../README.md)

---

## Table of Contents

- [How it works](#how-it-works)
- [Group placement — must be at vault root level](#group-placement)
- [How to create a vault entry for an SSH key](#how-to-create-a-vault-entry)
- [Mac 1 — SSH Agent Setup (with Prezto)](#mac-1--ssh-agent-setup-with-prezto)
- [Mac 2 — SSH Agent Setup (without Prezto)](#mac-2--ssh-agent-setup-without-prezto)
- [How to add a new SSH key in the future](#how-to-add-a-new-ssh-key-in-the-future)
- [SSH key security](#ssh-key-security)

---

## How it works

Both vaults have an `SSH Identities` group at the vault root:

- **`work_vault`** → work servers (cloud VMs, staging, production environments)
- **`personal_vault`** → personal servers, GitHub, home lab

**SSH keys are not stored inside the vault.** The SSH Agent tab uses **External File** — KeePassXC stores only the **path** to the private key file on disk. When the vault unlocks, it reads the key from that path and loads it into the SSH agent in memory.

**What this means for multi-machine setups:**

The same vault file is shared between Mac 1 and Mac 2, but the key files on disk are different per machine. If a key file path exists on Mac 1 but not Mac 2, the vault entry silently does nothing on Mac 2 — this is expected behaviour.

To make a key available on both Macs: copy the private key file from Mac 1 to Mac 2 via `scp` or USB. Once the file exists at the same path on Mac 2, the existing vault entry loads it automatically — no vault changes needed.

> **Never copy SSH private keys via email, Slack, Google Drive, or any messaging platform.** Use only `scp` on the same local network or a physical USB drive.

---

## Group Placement

The `SSH Identities` group must sit at the **vault root level** — directly under the vault name, not inside any other group.

**Correct structure:**

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

**To create the group at root level:**
- Right-click the vault root (topmost item in the left sidebar — the vault name itself) → **Add group** → name it `SSH Identities`

**If the group is inside another group by mistake:**
- Right-click `SSH Identities` → **Move group** → select the vault root as the destination

---

## How to Create a Vault Entry

This process is the same on both machines. Do this for each private key in `~/.ssh/`.

**First, check which keys exist:**

```bash
ls -la ~/.ssh/
```

Any file **without a `.pub` extension** is a private key. Skip `config`, `known_hosts`, and `known_hosts.old`.

**For each key:**

1. Open KeePassXC → click the correct vault tab at the top (`work_vault` for work keys, `personal_vault` for personal keys)
2. In the left sidebar, click the `SSH Identities` group to select it
3. Press `Ctrl + N` to open a new entry

**Fill in the General tab:**

| Field | Value |
|---|---|
| Title | `SSH Key - <description>` e.g. `SSH Key - GCP prod VM` or `SSH Key - GitHub` |
| Username | The SSH user for that server, e.g. `ubuntu`, `ec2-user`, `git` |
| Password | Leave blank (unless the key file has a passphrase) |
| URL | Server hostname or IP (optional) |

**Fill in the Notes tab:**

```
Key: /Users/<your-username>/.ssh/<key-filename>
Port: 22
Purpose: what this server or service is used for
```

**Fill in the SSH Agent tab:**

1. Under **Private key** → select **External File**
2. Enter the full path: `/Users/<your-username>/.ssh/<key-filename>`
3. KeePassXC auto-fills Fingerprint, Comment, and Public Key — this confirms the file was found and readable
4. Tick **Add key to agent when database is unlocked**
5. Tick **Remove key from agent when database is locked**

Click **OK** → `Ctrl + S` to save the vault.

---

## Mac 1 — SSH Agent Setup (with Prezto)

> This section applies when Mac 1 uses **Prezto** — a zsh framework that manages its own SSH agent. If your Mac 1 does not use Prezto, follow the [Mac 2 approach](#mac-2--ssh-agent-setup-without-prezto) instead.

**Issue:** KeePassXC's internal SSH agent socket has a known compatibility problem on macOS Sequoia — `ssh-add -l` returns "communication with agent failed" even when the vault is unlocked.

**Root cause:** KeePassXC's internal socket path at `$TMPDIR/keepassxc-*.socket` does not work reliably on Sequoia. The fix is to point KeePassXC at an already-running SSH agent socket — in this case, the Prezto-managed one.

---

### Step 1 — Identify the active agent socket

```bash
ssh-add -l
echo $SSH_AUTH_SOCK
cat ~/.zshrc | grep prezto
```

If Prezto is active, the socket is at:

```
/Users/<your-username>/.cache/prezto/ssh-agent.sock
```

---

### Step 2 — Enable SSH Agent in KeePassXC and set the override

1. `Cmd + ,` → click **SSH Agent** in the left sidebar
2. Toggle **Enable SSH Agent** ON
3. Set **SSH_AUTH_SOCK override** to:
   ```
   /Users/<your-username>/.cache/prezto/ssh-agent.sock
   ```
4. Leave all other fields at default
5. Click **OK** → `Cmd + Q` to quit KeePassXC → reopen → unlock vault

---

### Step 3 — Create entries in both vaults

- Work keys → `work_vault → SSH Identities` — follow [How to create a vault entry](#how-to-create-a-vault-entry)
- Personal keys → `personal_vault → SSH Identities`

`Ctrl + S` after all entries are added in each vault.

---

### Step 4 — Verify

```bash
# Both vaults locked
ssh-add -l
# The agent has no identities.

# work_vault unlocked
ssh-add -l
# (lists all work keys configured in SSH Identities)

# personal_vault also unlocked
ssh-add -l
# (lists all work keys + personal keys)

# Both vaults locked again
ssh-add -l
# The agent has no identities.
```

> **Why Prezto auto-loads some keys on startup:** Prezto's SSH module automatically loads keys with standard names (`id_ed25519`, `id_rsa`, etc.) when you open a new terminal. Custom-named keys are managed exclusively by KeePassXC. This is expected — locking both vaults and checking `ssh-add -l` may still show standard-named keys (loaded by Prezto), while custom-named work keys disappear.

---

## Mac 2 — SSH Agent Setup (without Prezto)

Mac 2 does not have Prezto — a persistent SSH agent with a fixed socket path is set up manually instead. The end result is the same as Mac 1: KeePassXC loads keys on vault unlock.

**Issue:** Same macOS Sequoia internal socket issue. Additionally, without Prezto, Mac 2 had an agent running with a randomly-named socket at `~/.ssh/agent/s.XXXXXX` — this changes on every reboot and cannot be used as a permanent override path.

**Fix:** Set up a persistent agent with a fixed socket path, configured in `.zshrc`.

---

### Step 1 — Check current state

```bash
ls -la ~/.ssh/
ssh-add -l
echo $SSH_AUTH_SOCK
cat ~/.zshrc | grep prezto
```

---

### Step 2 — Set up a persistent SSH agent with a fixed socket

```bash
mkdir -p ~/.ssh/agent

cat >> ~/.zshrc << 'EOF'

# SSH agent — fixed socket for KeePassXC integration
export SSH_AUTH_SOCK=~/.ssh/agent/ssh-agent.sock
if [ ! -S "$SSH_AUTH_SOCK" ]; then
    ssh-agent -a "$SSH_AUTH_SOCK" > /dev/null 2>&1
fi
EOF
```

```bash
source ~/.zshrc
```

**Verify the agent is running:**

```bash
echo $SSH_AUTH_SOCK
# /Users/<your-username>/.ssh/agent/ssh-agent.sock

ssh-add -l
# The agent has no identities.   ← socket works, no keys loaded yet — correct
```

---

### Step 3 — Set the override in KeePassXC on Mac 2

1. `Cmd + ,` → **SSH Agent**
2. Toggle **Enable SSH Agent** ON
3. Set **SSH_AUTH_SOCK override** to:
   ```
   /Users/<your-username>/.ssh/agent/ssh-agent.sock
   ```
4. Click **OK** → `Cmd + Q` → reopen KeePassXC → unlock vault

---

### Step 4 — Create entries and verify

Create entries in `work_vault → SSH Identities` — follow [How to create a vault entry](#how-to-create-a-vault-entry).

```bash
# Vault locked
ssh-add -l
# The agent has no identities.

# Vault unlocked
ssh-add -l
# (lists all keys configured in SSH Identities on Mac 2)

# Vault locked again
ssh-add -l
# The agent has no identities.
```

> If the key comment shows something unexpected (e.g. `user@old-machine`), this is metadata set when the key was originally generated — it does not affect how the key works.

---

## How to Add a New SSH Key in the Future

See [MAINTENANCE.md — How to Add a New SSH Key](MAINTENANCE.md#how-to-add-a-new-ssh-key-in-the-future) for the complete process.

---

## SSH Key Security

**Private keys — handle with care:**

- Private key files must have `600` permissions: `chmod 600 ~/.ssh/<key-filename>`
- Never share a private key file with anyone for any reason — not via email, Slack, Drive, or any messaging platform
- If someone needs access to the same server, they generate their own key pair and you add their **public key** to the server

**Public keys — safe to share:**

- Public key files (`.pub`) are safe to share, copy to servers, or paste into GitHub/GitLab
- To view: `cat ~/.ssh/<key-filename>.pub`
- To check a fingerprint (safe identifier that does not expose key content): `ssh-keygen -lf ~/.ssh/<key-filename>`

**Transferring a private key to another machine:**

```bash
# Option 1 — scp on the same local network
scp ~/.ssh/<key-filename> <username>@<machine-ip>:~/.ssh/<key-filename>
ssh <username>@<machine-ip> "chmod 600 ~/.ssh/<key-filename>"

# Option 2 — USB drive
# Copy file to USB, hand it over physically, recipient sets chmod 600

# Option 3 — GPG-encrypted transfer (if neither option above is possible)
gpg --symmetric --cipher-algo AES256 ~/.ssh/<key-filename>
# Share the .gpg file and the passphrase on separate channels
# Recipient decrypts:
# gpg --decrypt <key-filename>.gpg > ~/.ssh/<key-filename>
# chmod 600 ~/.ssh/<key-filename>
```
