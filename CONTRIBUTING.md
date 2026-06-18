# Contributing

Thank you for helping improve this guide. Contributions of all kinds are welcome — corrections, additions, clarifications, and improvements.

---

## What to contribute

- **Corrections** — steps that are outdated, inaccurate, or incorrect
- **Improvements** — clearer explanations, better examples, more complete coverage
- **New sections** — topics that are missing (e.g. other platforms, additional cloud setups)
- **Translations** — the guide in other languages

---

## How to contribute

1. **Fork the repository** on GitHub
2. **Create a branch** with a descriptive name:
   ```bash
   git checkout -b fix/totp-setup-steps
   git checkout -b docs/add-linux-section
   ```
3. **Make your changes** — see the [Style Guide](#style-guide) below
4. **Open a pull request** against `main` with a clear title and description of what was changed and why

For small corrections (typos, broken links, minor wording fixes), a direct PR is fine. For larger structural changes or new sections, open an issue first to discuss the approach.

---

## Style Guide

**Language and tone:**
- Write in plain, direct English — the same level of formality used throughout the guide
- Explain the "why" not just the "what" — the guide assumes the reader is technical but not familiar with these specific tools
- Avoid jargon without explanation

**Formatting:**
- Use `code blocks` for all commands, file paths, and code snippets
- Use tables for comparisons and structured information (settings, options, mappings)
- Use `---` horizontal rules to separate major sections
- Headings: `##` for top-level sections, `###` for steps, `####` for sub-steps
- Keep step headings as `### Step N: Description` — not `### Step N — Description`

**Privacy:**
- No real names, org names, account IDs, or identifiable information
- All examples must use generic placeholders: `<your-username>`, `<key-filename>`, `<your-org-id>.awsapps.com`, etc.
- Check that nothing in a diff would reveal a specific person's or organisation's setup

**Commands:**
- Test commands before adding them — all bash commands in this guide should be runnable as written (with placeholders replaced)
- Use `printf ... ; read -s` for sensitive input — never inline secrets in commands
- Prefer `ed25519` over `rsa` for new SSH key examples

**Links:**
- Use relative paths for internal document links: `[SSH_AGENT.md](SSH_AGENT.md)` from within `docs/`, `[docs/SSH_AGENT.md](docs/SSH_AGENT.md)` from root
- Link back to the README at the top of each `docs/` file: `[← Back to README](../README.md)`

---

## Reporting Issues

If you find something incorrect or unclear but do not want to submit a PR yourself, open a GitHub Issue with:
- Which file and section the issue is in
- What is wrong or missing
- What the correct information should be (if you know it)

---

## Code of Conduct

Be constructive and respectful. This project follows the [Contributor Covenant](https://www.contributor-covenant.org/) code of conduct.
