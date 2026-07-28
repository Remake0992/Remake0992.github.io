---
title: Deploy Open Terminal on macOS with Tailscale HTTPS
date: 2026-7-28 13:48:03
categories: [macos, self-hosting]
tags: [open-terminal, tailscale, open-webui, macos, launchd]
---

## Overview

This guide explains how to run Open WebUI's Open Terminal on a MacBook, publish it privately over HTTPS with Tailscale Serve, and configure it to start automatically with macOS.

Open Terminal provides AI agents with a dedicated environment for executing commands, managing files, and running code through an API. It is designed to be controlled by an AI assistant rather than used as a conventional browser-based terminal.

Tailscale Serve provides a Tailscale-managed HTTPS endpoint that is accessible only to devices on the same tailnet. It should not be confused with Tailscale Funnel, which publishes a service to the public internet.

## Prerequisites

Before beginning, ensure the following requirements are met:

- A MacBook with Tailscale installed and connected
- Access to the target Tailscale tailnet
- Open Terminal installed under the current macOS user
- A working Open Terminal configuration file
- Open WebUI, if Open Terminal will be used by an AI model
- Administrative access to Open WebUI's terminal connection settings
- A Tailscale HTTPS certificate-enabled tailnet name

Confirm the Open Terminal executable path:

```bash
command -v open-terminal
```

The deployment documented here used:

```text
/Users/collensaunders/.local/bin/open-terminal
```

> **Important:** If `command -v open-terminal` returns a different path, use that path in the LaunchAgent configuration.

## Architecture

The resulting connection path is:

```text
Open WebUI
    |
    | HTTPS over the Tailscale tailnet
    v
Tailscale Serve on the MacBook
    |
    | Local HTTP connection
    v
Open Terminal
```

TLS terminates at Tailscale Serve. Open Terminal can therefore listen on a local HTTP port while Tailscale supplies the trusted HTTPS endpoint.

## Procedure

### Step 1: Verify the Open Terminal configuration

The deployment used the following configuration path:

```text
/Users/collensaunders/.config/open-terminal/config.toml
```

Confirm that the file exists:

```bash
test -f /Users/collensaunders/.config/open-terminal/config.toml \
  && echo "Configuration found" \
  || echo "Configuration not found"
```

Protect the configuration file because it may contain authentication credentials:

```bash
chmod 600 /Users/collensaunders/.config/open-terminal/config.toml
```

### Step 2: Start Open Terminal manually

Start Open Terminal with the configured TOML file:

```bash
/Users/collensaunders/.local/bin/open-terminal run \
  --config /Users/collensaunders/.config/open-terminal/config.toml
```

Leave this terminal window open during the initial test.

**Expected result:** Open Terminal starts without an immediate configuration or port-binding error.

If the command fails, verify:

- The executable path
- The configuration path
- File permissions
- The configured listening port
- Whether another process already uses the port

Once manual operation is confirmed, stop the process with `Ctrl+C`.

### Step 3: Test the service locally

Restart Open Terminal and test its configured port from another Terminal window.

For example, if the service is listening on port `3000`:

```bash
curl -v http://127.0.0.1:3000/
```

An HTTP response confirms that the service is reachable locally. Depending on the endpoint and authentication settings, the response may be a success, authentication error, or API response.

> **Note:** A `401 Unauthorized` response can still confirm network connectivity. It means the service answered but requires valid authentication.

### Step 4: Publish the service with Tailscale Serve

For an Open Terminal service listening on local port `3000`, configure Tailscale Serve:

```bash
tailscale serve --bg 3000
```

The equivalent explicit command is:

```bash
tailscale serve --bg https / http://127.0.0.1:3000
```

The `--bg` option saves the Serve configuration so it remains active after the Terminal window is closed [1].

Review the resulting configuration:

```bash
tailscale serve status
```

Tailscale should display the HTTPS URL assigned to the MacBook and the local backend receiving proxied requests.

**Expected result:**

```text
https://macbook-name.tailnet-name.ts.net
    |
    +-- / proxy http://127.0.0.1:3000
```

The exact hostname depends on the MacBook's Tailscale machine name and tailnet DNS name.

### Step 5: Test the Tailscale HTTPS endpoint

From another device connected to the same tailnet, open the HTTPS URL displayed by:

```bash
tailscale serve status
```

The endpoint can also be tested with `curl`:

```bash
curl -v https://macbook-name.tailnet-name.ts.net/
```

Replace the example hostname with the URL returned by Tailscale Serve.

**Expected result:** The request reaches Open Terminal through a valid Tailscale-managed HTTPS connection.

> **Important:** Tailscale Serve is private to the tailnet by default. Do not enable Tailscale Funnel unless Open Terminal is intentionally meant to be reachable from the public internet [1].

### Step 6: Create the macOS LaunchAgent

Create the log directory:

```bash
mkdir -p /Users/collensaunders/Library/Logs/open-terminal
```

Create the LaunchAgent:

```bash
cat > /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openwebui.open-terminal</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/collensaunders/.local/bin/open-terminal</string>
        <string>run</string>
        <string>--config</string>
        <string>/Users/collensaunders/.config/open-terminal/config.toml</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>ThrottleInterval</key>
    <integer>10</integer>

    <key>StandardOutPath</key>
    <string>/Users/collensaunders/Library/Logs/open-terminal/stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/collensaunders/Library/Logs/open-terminal/stderr.log</string>
</dict>
</plist>
EOF
```

Validate the property list:

```bash
plutil -lint /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

**Expected result:**

```text
/Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist: OK
```

### Step 7: Load the LaunchAgent

Load the service for the current user:

```bash
launchctl bootstrap \
  gui/$(id -u) \
  /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

Enable and start it:

```bash
launchctl enable gui/$(id -u)/com.openwebui.open-terminal
launchctl kickstart -k gui/$(id -u)/com.openwebui.open-terminal
```

Check its status:

```bash
launchctl print gui/$(id -u)/com.openwebui.open-terminal
```

Confirm that the process is running:

```bash
pgrep -af open-terminal
```

Review the logs:

```bash
tail -n 100 /Users/collensaunders/Library/Logs/open-terminal/stdout.log
```

```bash
tail -n 100 /Users/collensaunders/Library/Logs/open-terminal/stderr.log
```

### Step 8: Configure Open WebUI

In Open WebUI:

1. Open the administrator or user connection settings.
2. Locate the Open Terminal integration.
3. Add the HTTPS URL displayed by `tailscale serve status`.
4. Enter the authentication credentials configured for Open Terminal.
5. Save the connection.
6. Select the terminal in a chat that supports Open Terminal.
7. Ensure the model uses native function calling.
8. Ask the model to perform a basic test:

```text
Use the selected terminal to execute: pwd
```

**Expected result:** The model invokes Open Terminal and returns the terminal's working directory.

## Validation

Complete the following checks after deployment.

### Confirm the LaunchAgent is running

```bash
launchctl print gui/$(id -u)/com.openwebui.open-terminal
```

### Confirm Open Terminal is listening

```bash
pgrep -af open-terminal
```

If the configured port is `3000`, check it with:

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

### Confirm Tailscale Serve is active

```bash
tailscale serve status
```

### Confirm HTTPS access from another tailnet device

```bash
curl -v https://macbook-name.tailnet-name.ts.net/
```

Use the actual hostname returned by `tailscale serve status`.

### Confirm Open WebUI can execute a command

Ask the model:

```text
Use the selected terminal to execute: whoami
```

The returned user should match the macOS account running the LaunchAgent.

### Confirm startup after a reboot

Restart the MacBook, sign in, and run:

```bash
launchctl print gui/$(id -u)/com.openwebui.open-terminal
```

Then retest the Tailscale HTTPS URL and execute another command from Open WebUI.

## Troubleshooting

### Open Terminal does not start

Check the error log:

```bash
tail -n 100 /Users/collensaunders/Library/Logs/open-terminal/stderr.log
```

Common causes include:

- An incorrect executable path
- A missing configuration file
- Invalid TOML syntax
- Incorrect file permissions
- A port already in use
- A dependency unavailable in the LaunchAgent environment

Run the exact command from `ProgramArguments` manually to expose startup errors:

```bash
/Users/collensaunders/.local/bin/open-terminal run \
  --config /Users/collensaunders/.config/open-terminal/config.toml
```

### LaunchAgent configuration changes are not applied

Unload the existing job:

```bash
launchctl bootout \
  gui/$(id -u) \
  /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

Validate the file:

```bash
plutil -lint /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

Load it again:

```bash
launchctl bootstrap \
  gui/$(id -u) \
  /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

### Tailscale HTTPS URL does not respond

Check the Serve configuration:

```bash
tailscale serve status
```

Test Open Terminal locally:

```bash
curl -v http://127.0.0.1:3000/
```

If the local request fails, troubleshoot Open Terminal before Tailscale. If the local request succeeds but the HTTPS request fails, check:

- Tailscale is connected on both devices
- Both devices are authorized on the tailnet
- Tailnet ACLs or grants allow access
- MagicDNS and HTTPS are enabled
- Tailscale Serve points to the correct local port

### Open WebUI reports that the terminal server was not found

If Open WebUI displays an error such as:

```text
Terminal server 'URL' not found
```

verify that the saved Open Terminal connection still exists and that the chat is not referencing a stale terminal-server record.

Delete and recreate the Open Terminal connection if necessary, then start a new chat and select the newly created terminal.

### File browsing works, but command execution fails

The Open WebUI browser interface may be able to reach Open Terminal even when the Open WebUI backend cannot. A successful browser-side connection is therefore not sufficient to prove end-to-end connectivity [1].

Test the Tailscale URL from the same host or container running the Open WebUI backend.

If Open WebUI runs in Docker:

1. Open a shell in the Open WebUI container.
2. Run `curl` against the Tailscale HTTPS URL.
3. Confirm DNS resolution, TLS negotiation, and HTTP connectivity.
4. Verify that the container can route traffic through the host's Tailscale connection.

The backend must be able to reach the configured terminal URL for model-driven command execution to work [1].

### Authentication fails

Verify that:

- Open WebUI uses the same credentials configured in Open Terminal.
- The API key or token contains no accidental spaces.
- The credential was not copied with surrounding quotation marks.
- Open Terminal was restarted after its configuration changed.
- The Open WebUI connection was saved after updating the credential.

Do not disable authentication merely to work around a connectivity problem.

## Service Management

### Restart Open Terminal

```bash
launchctl kickstart -k gui/$(id -u)/com.openwebui.open-terminal
```

### Stop Open Terminal

```bash
launchctl kill SIGTERM gui/$(id -u)/com.openwebui.open-terminal
```

Because `KeepAlive` is enabled, launchd may restart the process automatically.

### Disable automatic startup

```bash
launchctl disable gui/$(id -u)/com.openwebui.open-terminal
```

```bash
launchctl bootout \
  gui/$(id -u) \
  /Users/collensaunders/Library/LaunchAgents/com.openwebui.open-terminal.plist
```

### Remove the Tailscale Serve configuration

```bash
tailscale serve reset
```

> **Warning:** `tailscale serve reset` removes the device's complete Serve configuration, including unrelated Serve routes configured on the same MacBook.

## Security Considerations

- Use Tailscale Serve rather than Funnel to keep Open Terminal private to the tailnet.
- Require Open Terminal authentication even when access is restricted by Tailscale.
- Store configuration files with restrictive permissions such as `600`.
- Do not commit API keys or configuration files containing credentials to Git.
- Restrict access using Tailscale ACLs or grants.
- Remember that Open Terminal allows AI agents to execute commands and access files under the macOS user account running the service.
- Run the service under a dedicated, minimally privileged account when broader access is unnecessary.
- Review Open Terminal and Open WebUI logs for unexpected command execution.
- Do not run Open Terminal as `root` unless a documented administrative requirement exists.

## Related Resources

- [Open Terminal GitHub repository](https://github.com/open-webui/open-terminal)
- [Tailscale Serve documentation](https://tailscale.com/kb/1242/tailscale-serve)
- [Tailscale access-control documentation](https://tailscale.com/kb/1018/acls)
- [Open WebUI documentation](https://docs.openwebui.com/)
