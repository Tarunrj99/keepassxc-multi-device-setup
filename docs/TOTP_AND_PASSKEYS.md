# TOTP & Passkeys — KeePassXC

How to set up and use TOTP (2FA codes) and passkeys stored directly in KeePassXC — no separate authenticator app needed.

[← Back to README](../README.md)

---

## Table of Contents

- [TOTP](#totp)
  - [How to add TOTP to any account](#how-to-add-totp-to-any-account)
  - [How to use TOTP when logging in](#how-to-use-totp-when-logging-in)
  - [Common accounts to set up TOTP for](#common-accounts-to-set-up-totp-for)
  - [Known limitation — proprietary MFA apps](#known-limitation--proprietary-mfa-apps)
- [Passkeys](#passkeys)
  - [How passkeys work with KeePassXC](#how-passkeys-work-with-keepassxc)
  - [How to create a passkey](#how-to-create-a-passkey)
  - [How to use a passkey to log in](#how-to-use-a-passkey-to-log-in)
  - [Passkey limitation on Android](#passkey-limitation-on-android)

---

## TOTP

KeePassXC generates 6-digit TOTP codes for accounts that have 2FA enabled — keeping the password and the 2FA code in the same vault entry. No Google Authenticator, no Authy, no separate app.

> **Important:** Once both the password and TOTP are in the vault, the Master Password protects both factors. This makes the setup highly convenient but means **Master Password security becomes even more critical** — losing the vault means losing access to your 2FA codes as well.

---

### How to Add TOTP to Any Account

#### Part 1 — Get the TOTP secret key from the website

1. Log into the account on the website
2. Go to **Settings → Security → Two-Factor Authentication** (exact path varies per site)
3. Select **Authenticator App** as the 2FA method
4. The site shows a QR code — look for a link below or beside it:
   - "Can't scan QR code?"
   - "Enter key manually"
   - "Manual setup key"
5. Click it — a Base32 text key appears (letters and numbers only, e.g. `JBSWY3DPEHPK3PXP`)
6. Copy that key — do not close the page yet

#### Part 2 — Find the matching entry in KeePassXC

1. Open KeePassXC → unlock the correct vault
2. Navigate to the group where the login entry is stored:
   - Work accounts → `Infrastructure Logins → Passwords`
   - Personal accounts → `Personal Logins → Login → Websites`
3. Find the entry for that account
4. **Right-click the entry** → **TOTP → Set up TOTP**

#### Part 3 — Configure TOTP in KeePassXC

1. A small window opens with a **Secret key** input field
2. Paste the Base32 key copied from the website
3. Select **Default TOTP Settings (RFC 6238)** — do not change Period or Digits unless the site specifically requires non-standard values
4. Click **OK**
5. Press `Ctrl + S` to save the vault

#### Part 4 — Verify it works

1. Go back to the website — it will ask you to enter the 6-digit code to confirm setup
2. In KeePassXC, right-click the same entry → **TOTP → Copy TOTP**
3. Paste the 6-digit code into the website's verification field (you have 30 seconds)
4. Site confirms → TOTP is now active

---

### How to Use TOTP When Logging In

1. Go to the login page — the browser extension fills username and password automatically
2. The site asks for the 2FA code
3. In KeePassXC, navigate to the entry:
   - Work accounts → `Infrastructure Logins → Passwords`
   - Personal accounts → `Personal Logins → Login → Websites`
4. Right-click the entry → **TOTP → Copy TOTP**
5. Paste the 6-digit code into the site

> The code rotates every 30 seconds. Paste it before it changes. KeePassXC shows a countdown timer on the TOTP copy dialog.

---

### Common Accounts to Set Up TOTP For

These are common services that support standard TOTP authenticator apps. The exact path to find the setup varies — look under Settings → Security → Two-Factor Authentication → Authenticator App on each.

| Service | Setup path | Vault to use |
|---|---|---|
| **GitHub** | Settings → Password and authentication → Two-factor authentication → Authenticator app | `personal_vault` |
| **GitLab** | Preferences → Account → Two-Factor Authentication | `personal_vault` |
| **Cloudflare** | Profile → Access Management → Authentication → Two-Factor Authentication | `work_vault` |
| **AWS SSO** | Security → Authenticator apps → Register device | `work_vault` |
| **MongoDB Atlas** | Account → Profile → Security → Two-Factor Authentication | `work_vault` |
| **Azure / Microsoft 365** | mysignins.microsoft.com → Security info → Add sign-in method | `work_vault` |
| **Google** | myaccount.google.com → Security → 2-Step Verification → Authenticator | either vault |
| **Stripe** | Dashboard → Settings → Security → Two-step authentication | `work_vault` |
| **Datadog** | Account Settings → Security → Two-Factor Authentication | `work_vault` |

> **AWS SSO supports multiple registered authenticator devices** — you can register a new device specifically for KeePassXC without removing existing ones. Each device gets its own setup QR code.

> **Azure / Microsoft 365:** The full login URL contains session tokens that change every time. Store only the base URL `https://login.microsoftonline.com` in the entry so the browser extension can match it reliably.

---

### Known Limitation — Proprietary MFA Apps

Some services do not support standard TOTP authenticators. They only allow their own proprietary app (e.g. OneLogin Protect, Duo Mobile) or hardware keys (YubiKey).

**What this means in practice for such services:**
- Username and password → stored and autofilled by KeePassXC as normal
- MFA step → handled by the proprietary app on your phone
- When you log in, KeePassXC autofills the credentials, then the site asks for MFA — open the proprietary app, copy the code or tap Approve

This is not a KeePassXC limitation — it is a company or admin policy decision on the service side.

---

## Passkeys

Passkeys are a newer authentication standard replacing passwords on supported sites. KeePassXC added passkey support in version 2.7.7+.

**What a passkey is:** A cryptographic key pair generated per site. The private key stays encrypted in your vault. The site stores the public key. No password to type — no password to steal.

**Check your KeePassXC version:** `KeePassXC menu → About KeePassXC` — must be 2.7.7 or later.

---

### How Passkeys Work with KeePassXC

1. A site offers "Create a passkey" during signup or in security settings
2. The **KeePassXC browser extension intercepts the prompt** and asks which database to save the passkey in
3. The passkey is stored as an entry in the vault:
   - Work passkeys → `work_vault → Infrastructure Logins → Passkeys`
   - Personal passkeys → `personal_vault → Personal Logins → Login → Passkeys`
4. On next login, the site asks to use the passkey — the extension retrieves it from the vault automatically

---

### How to Create a Passkey

**Example — GitHub:**

1. GitHub → **Settings** → **Password and authentication** → **Passkeys** → **Add a passkey**
2. GitHub prompts your browser for a passkey
3. The KeePassXC browser extension intercepts the prompt and opens a dialog asking where to save it
4. Select `personal_vault → Personal Logins → Login → Passkeys`
5. The passkey is created and saved automatically

The process is the same on other sites — look for "Add a passkey" or "Register a passkey" in account security settings.

---

### How to Use a Passkey to Log In

1. Go to the site's login page
2. The site shows a "Sign in with passkey" option (or automatically triggers the passkey prompt)
3. The KeePassXC browser extension detects the prompt and retrieves the passkey from the vault
4. Authentication completes — no username or password needed

---

### Passkey Limitation on Android

Passkeys stored in KeePassXC only work on desktop (Mac) via the browser extension. **Keepass2Android does not currently support passkeys.**

For sites that use passkeys on Mac, use username + password from the vault on Android instead — both can coexist. Most sites that offer passkeys still keep password login as a fallback.
