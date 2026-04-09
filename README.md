# bitwarden-auth

`bitwarden-auth` is a Codex skill for retrieving credentials and TOTP codes from a Vaultwarden or Bitwarden vault through the wrapper scripts in [`scripts/`](./scripts). The intended usage is:

- point Bitwarden CLI at your server
- unlock through the wrappers, not ad hoc `bw` commands
- fetch only the secret needed for the current task
- optionally store newly created credentials back into the vault

## Requirements
- Bitwarden CLI installed as `bw`
- `jq`
- `python3`


## Install the dependencies

### 1. Install Vaultwarden

Install [Vaultwarden](https://github.com/dani-garcia/vaultwarden)

### 2. Install Bitwarden CLI

Bitwarden CLI documentation can be found [here](https://bitwarden.com/help/cli/#download-and-install)



- If you are running Vaultwarden locally on `localhost`, Bitwarden CLI still expects your self-hosted server URL to use `https://`.
- In practice, that means you need some way to expose your local Vaultwarden instance over HTTPS before connecting `bw` to it.
- A reverse proxy such as Caddy is one option, but it is not required. Any approach that gives Bitwarden CLI a working HTTPS endpoint can work.
- Caddy documentation: <https://caddyserver.com/docs/>

## Start Vaultwarden locally

From this repo, create a local `compose.yaml` like this:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://your-vaultwarden-instance.example.com"
    volumes:
      - ./vw-data:/data
    ports:
      - "127.0.0.1:8000:80" # example only
```

Start it:

```bash
docker compose up -d
```

Check that the container is up:

```bash
docker compose ps
```

## Connect Bitwarden CLI to your Vaultwarden server

Bitwarden CLI must be configured to use your self-hosted server before login.

If your Vaultwarden instance is running locally, make sure you have already exposed it over HTTPS before this step. Bitwarden’s self-hosted configuration expects an `https://` server URL.

Start clean:

```bash
bw logout
```

Point the CLI at your local server:

```bash
bw config server https://your-vaultwarden-instance.example.com
```

Then log in interactively:

```bash
bw login
```

You can confirm the configured server with:

```bash
bw status
```

If you also use the desktop app or browser extension, choose `Self-hosted` on the login screen and enter:

```text
https://your-vaultwarden-instance.example.com
```

## For macOS users

The current wrapper scripts read your master password from macOS Keychain using:

```bash
security find-generic-password -a codex -s bw-master -w
```

To create that entry:

```bash
security add-generic-password -U -a codex -s bw-master -w 'YOUR_MASTER_PASSWORD'
```

**Note**
If you are not on macOS, replace that `security find-generic-password ...` step in the wrapper scripts with the secret-loading approach that fits your OS and workflow.

## Make the wrapper commands available

The skill documentation refers to `get-login`, `get-totp`, and `store-login` as direct commands. Add this repo’s `scripts/` directory to your shell `PATH`:

```bash
export PATH="$PWD/scripts:$PATH"
```

To keep that available in future shells, add the equivalent line to your shell profile after replacing `$PWD` with the absolute repo path.

Example:

```bash
export PATH="/Users/you/path/to/bitwarden-auth/scripts:$PATH"
```

## Smoke test the setup

Once you have:

- a running Vaultwarden instance
- an HTTPS URL for that Vaultwarden instance if you are self-hosting locally
- Bitwarden CLI logged in against that server
- a macOS Keychain item named by account `codex` and service `bw-master`
- at least one login item in your vault

you can test the wrappers.

Fetch a login:

```bash
get-login "My Test Login"
```

Fetch a TOTP code:

```bash
get-totp "My Test Login"
```

Store a new login:

```bash
printf '%s' '{"url":"https://example.com","username":"new-user@example.com","password":"generated-secret"}' \
  | store-login "Example staging - qa agent"
```

## Using the skill in Codex

Once the local setup is working, invoke the skill explicitly:

```text
$bitwarden-auth
```

Typical prompts:

- `Use $bitwarden-auth to get the login for "GitHub Agent Account".`
- `Use $bitwarden-auth to fetch the TOTP code for "GitHub Agent Account".`
- `Use $bitwarden-auth to store credentials for the new staging user I just created.`

Important usage rule from this repo:

- do not call raw `bw` commands from tasks that use the skill
- use the wrapper scripts instead

## Troubleshooting

### `bw login` or `bw config server` fails

- Confirm your Vaultwarden HTTPS endpoint is reachable.
- Confirm Vaultwarden itself is running and reachable behind whatever HTTPS setup you chose.
- Confirm the server URL includes `https://`.

### The wrappers say the vault is locked or unavailable

- Confirm `bw login` already succeeded against the same server.
- Confirm the macOS Keychain entry exists under account `codex` and service `bw-master`.
- Confirm the Keychain item contains the correct master password.

### TLS or certificate errors from `bw`

- Confirm the HTTPS endpoint you configured for Vaultwarden is reachable and trusted by your machine.
- Remember that the wrappers rely on system trust via `NODE_USE_SYSTEM_CA=1`.

### The scripts are present but commands are not found

- Re-check your `PATH`.
- From the repo root, run `echo $PATH`.
- Confirm `scripts/` is included.


## Notes on support boundaries

- Vaultwarden is an unofficial Bitwarden-compatible server, not the official Bitwarden server.
- If you hit Vaultwarden-specific issues, check Vaultwarden docs and discussions first rather than official Bitwarden support channels.
