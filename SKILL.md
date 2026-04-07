---
name: bitwarden-auth
description: Retrieve and store login credentials, plus fetch TOTP codes, from a Vaultwarden / Bitwarden vault via secure wrapper scripts. Use this skill whenever a task requires a stored password, API key, or 2FA TOTP code — for example authenticating to a service, filling in credentials during automated testing, creating a new account that must be saved for later use, or fetching a one-time code. Never attempt to call bw directly; always use the wrappers in this skill.
---

## Vaultwarden authentication skill

Use this skill when a task requires logging into a website or entering a time-based one-time password from the local Vaultwarden instance. This skill is for authentication tasks only. It should not be used for general Vaultwarden browsing, item discovery, bulk export, or debugging

## What this skill can do

This skill supports three actions:

1. Get login context for a known item:
  - login URL
  - username
  - password

2. Get a current TOTP code for a known item. TOTP codes expire every 30 seconds — retrieve them immediately before use.

3. Store login context for a newly created account:
  - item name
  - login URL
  - username
  - password
  - optional notes

## When to use this skill

Use this skill when a task involves authenticating to a website or service, or retrieveing a TOTP code stored in the local Vaultwarden instance.

Typical examples:
- Log into a website using a stored account
- Retrieve the username/password for a known login item
- Create an account during testing, then save the resulting credentials in the vault for reuse
- Retrieve a current TOTP code for a known login item

## When not to use this skill
- the task requires broad vault enumeration instead of a known item lookup
- the item name in the vault is unknown

## Security rules
- **Never log credentials** to files, stdout summaries, or commit messages.
- **Never store credentials in code or presist them** — always retrieve them at runtime.
- Never call raw `bw` commands directly.
- Never print more secret material than the task needs.
- Use the returned secret immediately, then continue the task.
- Never use generic shell variable names such as `USER`, `USERNAME`, `PASSWORD`, or `LOGIN` for retrieved secrets. Use service-prefixed names like `ITEM_NAME_LOGIN_EMAIL` instead.
- For login tasks, first check whether the repo provides a dedicated helper script for that service. If it does, use it. Do not recreate the flow inline unless the helper is missing or broken
- When storing newly created credentials, pass secret values through stdin or a narrowly scoped helper. Do not put raw passwords on the command line.
- Do not overwrite an existing vault item silently. Create a clearly named new item or stop and ask the user how the existing item should be handled.

## Automation guardrails

When browser automation is required:

1. Prefer a repo-owned wrapper script over ad hoc shell composition once the flow spans more than one command.
2. Keep credential-bearing browser commands quiet. If a CLI echoes filled values, redirect or suppress that output.
3. Before submit, verify that the value in an email field still matches the fetched login and passes client-side email validity checks.
4. After submit, verify success with page state such as URL, title, headings, or known error text rather than re-snapshotting a password field.
5. Treat shell-reserved or shell-populated variable names as banned for secret handling, even if they seem convenient.

## Approved commands

Use only these wrapper commands:

- `get-login "Item Name"`
  Returns `{"url":"https://github.com/login","username":"codex@login","password":"example-password"}`

- `get-totp "<item name>"`
  Returns only the current TOTP code, for example: 123456

- `printf '%s' '{"url":"https://example.com","username":"agent@example.com","password":"secret"}' | store-login "Item Name"`
  Creates a new login item and returns `{"id":"...","name":"Item Name","url":"https://example.com","username":"agent@example.com"}`

## Normal workflow

1. Identify the exact Vaultwarden item needed.
2. If the task needs login credentials, run:
   - `get-login "<item name>"`
3. If the task needs a 2FA code, run:
   - `get-totp "<item name>"`
4. If the task creates a new account whose credentials should be saved, pass a JSON object on stdin to:
   - `store-login "Item Name"`
5. Move the returned values into narrowly scoped, service-prefixed variables or pass them directly into a repo-owned helper script.
6. Use the credential or code only for the immediate task.
7. Do not repeat, log or persist secrets outside the vault.

## Examples

Get credentials for a site:
```bash
get-login "GitHub Agent Account"
```

Get a current TOTP code:
```bash
get-totp "GitHub Agent Account"
```

Store credentials for a newly created account:
```bash
printf '%s' '{"url":"https://example.com","username":"new-user@example.com","password":"generated-secret"}' \
  | store-login "Example staging - qa agent"
```

## Troubleshooting
- If get-login, get-totp, or store-login fails, confirm the item name is correct and the vault is unlocked successfully
- If the wrapper reports that the vault is locked or unavailable, stop and surface the wrapper error clearly.
- If the item name is unknown, ask the user for the exact item name rather than guessing.
- If store-login reports that the item already exists, stop and decide whether a new name or an explicit update workflow is needed.
