---
title: Deploying Open Terminal for OpenWebUI
date: 2026-6-16 14:45:00
categories: [homelab,selfhosted,openwebui]
tags: [openwebui,open-terminal,linux,systemd,sudo,docker,selfhosted]
---

## Giving OpenWebUI hands

OpenWebUI is great as a chat interface, but by default it does not magically have access to administer the host systems around it. This becomes especially obvious when OpenWebUI is running in Docker. Giving a container access to `sudo` inside the container does not mean it can administer the baremetal host. Container permissions and host permissions are separate things.

What I set up was a very stupid, but very workable approach: **Open Terminal running baremetal on the target machine**.

Instead of trying to make the OpenWebUI container itself privileged, I installed `open-terminal` directly on the Linux host. OpenWebUI can then connect to that Open Terminal service over HTTP using an API key. Open Terminal runs commands locally on the host, under a real Linux user, with a controlled sudoers allowlist. This is really handy for Operating Systems like Proxmox.

The result is that OpenWebUI can perform useful admin tasks on the machine without needing the OpenWebUI container itself to be fully privileged.

## Final architecture

The setup looks like this:

```text
OpenWebUI
   |
   | HTTP API request with API key
   v
Open Terminal service
   |
   | Runs commands as csadmin
   v
Linux host
   |
   | Optional passwordless sudo for allowlisted commands
   v
Privileged host operations
```

On the host, Open Terminal is configured as a normal `systemd` service:

```text
/etc/systemd/system/open-terminal.service
```

The API key is stored in:

```text
/etc/open-terminal.env
```

The broad sudo allowlist is stored in:

```text
/etc/sudoers.d/open-terminal-broad
```

By default, the Open Terminal service listens on:

```text
0.0.0.0:8054
```

That means it accepts connections on all network interfaces. In OpenWebUI, the endpoint should be added as either the hostname or IP of the target machine:

```text
http://network-services:8054
```

or:

```text
http://192.168.0.105:8054
```

## Why baremetal matters

OpenWebUI may be running in Docker, but the terminal capability I wanted was host-level command execution. If OpenWebUI is containerized, there are a few ways to approach that problem:

- give the container more privileges,
- mount the Docker socket,
- SSH from the container to the host,
- or run a host-side command execution service.

The repeatable option here was the last one: **run Open Terminal directly on the host**.

That keeps OpenWebUI itself from needing privileged container access. OpenWebUI only needs to know the Open Terminal URL and API key.

## How the service is configured

The deployment script creates a `systemd` service similar to this:

```ini
[Unit]
Description=Open Terminal for OpenWebUI
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=username
WorkingDirectory=/home/username
Environment=PATH=/home/username/.local/bin:/usr/local/bin:/usr/bin:/bin
EnvironmentFile=/etc/open-terminal.env
ExecStart=/home/username/.local/bin/open-terminal run --host 0.0.0.0 --port 8054 --api-key ${OPEN_TERMINAL_API_KEY}
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The important pieces are:

### Service user

Open Terminal runs as:

```text
username
```

This means normal commands execute with the permissions of `username`.

### Working directory

The service uses the user's home directory:

```text
/home/username
```

### Open Terminal executable

The Open Terminal executable is installed into the user's local binary path:

```text
/home/username/.local/bin/open-terminal
```

### Listen address

The service binds to:

```text
0.0.0.0
```

This allows OpenWebUI to reach the service from another host or from inside a container.

### Listen port

The default port is:

```text
8054
```

### API key

The API key is generated during deployment and stored here:

```text
/etc/open-terminal.env
```

The file contains:

```bash
OPEN_TERMINAL_API_KEY=p_generatedkeyhere
```

The API key is then passed into the Open Terminal service when it starts.

## How sudo access is configured

The setup intentionally does **not** use full unrestricted sudo like this:

```bash
username ALL=(ALL) NOPASSWD: ALL
```

That would effectively give OpenWebUI full root access to the host.

Instead, the script creates a broad but allowlisted sudoers file:

```text
/etc/sudoers.d/open-terminal-broad
```

The allowlist includes command groups for common administrative tasks:

- Docker management
- Journal log access
- selected `systemctl` commands
- APT package management
- reading files under `/var/log`
- network and firewall inspection/changes
- basic system maintenance commands

This is still powerful, but it is safer than blanket `NOPASSWD: ALL`.

The sudoers file is validated during deployment with:

```bash
visudo -cf /etc/sudoers
visudo -cf /etc/sudoers.d/open-terminal-broad
```

That step is important because a broken sudoers file can lock out administrative access.

## What the sudo allowlist permits

The broad allowlist created by the script includes these command aliases:

```text
OT_DOCKER
OT_JOURNAL
OT_SYSTEMCTL_READ
OT_SYSTEMCTL_SERVICES
OT_APT
OT_LOGS
OT_NETWORK
OT_MAINT
```

In practical terms, that means Open Terminal can run commands such as:

```bash
sudo docker ps
```

```bash
sudo journalctl -n 100
```

```bash
sudo systemctl status docker
```

```bash
sudo apt update
```

```bash
sudo tail -n 100 /var/log/syslog
```

```bash
sudo ss -ltnp
```

```bash
sudo df -h
```

It does not grant unrestricted access to every command on the system.

## Why this setup is useful

This gives OpenWebUI a repeatable way to manage other machines in the environment.

For example, after deploying Open Terminal to a server, OpenWebUI can be configured to connect to that server and perform allowlisted actions such as:

```bash
docker ps
```

```bash
journalctl -u some-service
```

```bash
systemctl status docker
```

```bash
apt update
```

Because Open Terminal is installed baremetal, these commands run against the real host, not inside the OpenWebUI container.

## Deployment script

Copy and paste the following script into the terminal on the target Linux host.

This script is designed to be tolerant of broken third-party APT repositories. If `apt-get update` fails because of a repository signing issue, the script continues as long as the required tools are already installed.

```bash
sudo bash -s <<'SCRIPT'
set -euo pipefail

# ============================================================
# Open Terminal baremetal replication script for OpenWebUI
# APT-repo-error-tolerant version
# ============================================================

TERMINAL_USER="${TERMINAL_USER:-${SUDO_USER:-REPLACE_WITH_YOUR_DESIRED_USERNAME}}"
OPEN_TERMINAL_HOST="${OPEN_TERMINAL_HOST:-0.0.0.0}"
OPEN_TERMINAL_PORT="${OPEN_TERMINAL_PORT:-8054}"
INSTALL_SUDOERS="${INSTALL_SUDOERS:-1}"
ADD_DOCKER_GROUP="${ADD_DOCKER_GROUP:-1}"

echo "==> Target Open Terminal user: ${TERMINAL_USER}"
echo "==> Listen address: ${OPEN_TERMINAL_HOST}:${OPEN_TERMINAL_PORT}"

if [[ -z "${OPEN_TERMINAL_API_KEY:-}" ]]; then
  OPEN_TERMINAL_API_KEY="$(python3 - <<'PY'
import secrets, string
alphabet = string.ascii_letters + string.digits
print('p_' + ''.join(secrets.choice(alphabet) for _ in range(32)))
PY
)"
fi

echo "==> Checking prerequisites"

if command -v apt-get >/dev/null 2>&1; then
  echo "==> Running apt-get update, but continuing if third-party repos fail"
  apt-get update || echo "WARNING: apt-get update failed. Continuing anyway."

  echo "==> Installing prerequisites, best effort"
  DEBIAN_FRONTEND=noninteractive apt-get install -y \
    curl \
    ca-certificates \
    sudo \
    openssl \
    python3 \
    python3-venv \
    python3-pip \
    iproute2 || echo "WARNING: apt-get install had problems. Continuing if required tools already exist."
fi

echo "==> Verifying required commands"

if ! command -v python3 >/dev/null 2>&1; then
  echo "ERROR: python3 is required but not installed."
  exit 1
fi

if ! command -v curl >/dev/null 2>&1; then
  echo "ERROR: curl is required but not installed."
  echo "Fix APT or install curl manually, then rerun."
  exit 1
fi

if ! command -v sudo >/dev/null 2>&1; then
  echo "ERROR: sudo is required but not installed."
  exit 1
fi

echo "==> Ensuring user exists: ${TERMINAL_USER}"

if ! id "${TERMINAL_USER}" >/dev/null 2>&1; then
  useradd --create-home --shell /bin/bash "${TERMINAL_USER}"
fi

USER_HOME="$(getent passwd "${TERMINAL_USER}" | cut -d: -f6)"

if [[ -z "${USER_HOME}" || ! -d "${USER_HOME}" ]]; then
  echo "ERROR: Could not determine home directory for ${TERMINAL_USER}"
  exit 1
fi

echo "==> User home: ${USER_HOME}"

echo "==> Adding ${TERMINAL_USER} to sudo group"
usermod -aG sudo "${TERMINAL_USER}" || true

if [[ "${ADD_DOCKER_GROUP}" == "1" ]] && getent group docker >/dev/null 2>&1; then
  echo "==> Adding ${TERMINAL_USER} to docker group"
  usermod -aG docker "${TERMINAL_USER}" || true
fi

echo "==> Installing uv for ${TERMINAL_USER}, if needed"

sudo -H -u "${TERMINAL_USER}" bash -lc '
set -e
export PATH="$HOME/.local/bin:$PATH"
if ! command -v uv >/dev/null 2>&1 && [[ ! -x "$HOME/.local/bin/uv" ]]; then
  curl -LsSf https://astral.sh/uv/install.sh | sh
fi
'

echo "==> Installing/upgrading open-terminal for ${TERMINAL_USER}"

sudo -H -u "${TERMINAL_USER}" bash -lc '
set -e
export PATH="$HOME/.local/bin:$PATH"
uv tool install --upgrade open-terminal
'

if [[ ! -x "${USER_HOME}/.local/bin/open-terminal" ]]; then
  echo "ERROR: open-terminal was not installed at ${USER_HOME}/.local/bin/open-terminal"
  exit 1
fi

echo "==> open-terminal installed at ${USER_HOME}/.local/bin/open-terminal"

if [[ "${INSTALL_SUDOERS}" == "1" ]]; then
  echo "==> Installing broad allowlisted sudoers policy"

  cat > /etc/sudoers.d/open-terminal-broad <<EOF
# Broad allowlisted sudo access for Open Terminal / OpenWebUI.
# This is intentionally not full NOPASSWD: ALL.
# Review and reduce for production.

Cmnd_Alias OT_DOCKER = /usr/bin/docker *
Cmnd_Alias OT_JOURNAL = /usr/bin/journalctl *
Cmnd_Alias OT_SYSTEMCTL_READ = /usr/bin/systemctl status *, /usr/bin/systemctl list-units *, /usr/bin/systemctl list-unit-files *, /usr/bin/systemctl is-active *, /usr/bin/systemctl is-enabled *
Cmnd_Alias OT_SYSTEMCTL_SERVICES = /usr/bin/systemctl start docker, /usr/bin/systemctl stop docker, /usr/bin/systemctl restart docker, /usr/bin/systemctl reload docker, /usr/bin/systemctl start nginx, /usr/bin/systemctl stop nginx, /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
Cmnd_Alias OT_APT = /usr/bin/apt update, /usr/bin/apt upgrade *, /usr/bin/apt install *, /usr/bin/apt remove *, /usr/bin/apt purge *, /usr/bin/apt autoremove *
Cmnd_Alias OT_LOGS = /usr/bin/tail /var/log/*, /usr/bin/tail -n * /var/log/*, /usr/bin/head /var/log/*, /usr/bin/head -n * /var/log/*, /usr/bin/cat /var/log/*
Cmnd_Alias OT_NETWORK = /usr/sbin/ufw status *, /usr/sbin/ufw allow *, /usr/sbin/ufw deny *, /usr/sbin/ufw delete *, /usr/sbin/ufw reload, /usr/bin/ss *, /usr/bin/netstat *, /usr/sbin/ip *
Cmnd_Alias OT_MAINT = /usr/bin/df *, /usr/bin/du *, /usr/bin/free *, /usr/bin/ps *, /usr/bin/top *, /usr/bin/htop *, /usr/bin/kill *, /usr/bin/pkill *

${TERMINAL_USER} ALL=(root) NOPASSWD: OT_DOCKER, OT_JOURNAL, OT_SYSTEMCTL_READ, OT_SYSTEMCTL_SERVICES, OT_APT, OT_LOGS, OT_NETWORK, OT_MAINT
EOF

  chmod 0440 /etc/sudoers.d/open-terminal-broad

  echo "==> Validating sudoers policy"
  visudo -cf /etc/sudoers
  visudo -cf /etc/sudoers.d/open-terminal-broad
fi

echo "==> Writing API key environment file"

cat > /etc/open-terminal.env <<EOF
OPEN_TERMINAL_API_KEY=${OPEN_TERMINAL_API_KEY}
EOF

chmod 0600 /etc/open-terminal.env

echo "==> Writing systemd service"

cat > /etc/systemd/system/open-terminal.service <<EOF
[Unit]
Description=Open Terminal for OpenWebUI
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=${TERMINAL_USER}
WorkingDirectory=${USER_HOME}
Environment=PATH=${USER_HOME}/.local/bin:/usr/local/bin:/usr/bin:/bin
EnvironmentFile=/etc/open-terminal.env
ExecStart=${USER_HOME}/.local/bin/open-terminal run --host ${OPEN_TERMINAL_HOST} --port ${OPEN_TERMINAL_PORT} --api-key \${OPEN_TERMINAL_API_KEY}
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

chmod 0644 /etc/systemd/system/open-terminal.service

echo "==> Enabling and starting open-terminal.service"

systemctl daemon-reload
systemctl enable open-terminal.service
systemctl restart open-terminal.service

sleep 3

echo "==> Service status"
systemctl --no-pager --full status open-terminal.service || true

echo
echo "==> Listening socket check"
ss -ltnp | grep ":${OPEN_TERMINAL_PORT}" || true

echo
echo "==> Testing sudo allowlist"

if [[ "${INSTALL_SUDOERS}" == "1" ]]; then
  sudo -u "${TERMINAL_USER}" sudo -n /usr/bin/systemctl status open-terminal.service >/dev/null && \
    echo "sudo systemctl status test: OK" || \
    echo "sudo systemctl status test: FAILED"

  if command -v docker >/dev/null 2>&1; then
    sudo -u "${TERMINAL_USER}" sudo -n /usr/bin/docker ps >/dev/null && \
      echo "sudo docker ps test: OK" || \
      echo "sudo docker ps test: FAILED"
  fi
fi

echo
echo "============================================================"
echo "Open Terminal deployment complete."
echo
echo "Service user: ${TERMINAL_USER}"
echo "Listen URL:   http://<TARGET_MACHINE_IP>:${OPEN_TERMINAL_PORT}"
echo "Bind host:    ${OPEN_TERMINAL_HOST}"
echo "Port:         ${OPEN_TERMINAL_PORT}"
echo "API key:      ${OPEN_TERMINAL_API_KEY}"
echo
echo "Add this to OpenWebUI using:"
echo "  URL:     http://<TARGET_MACHINE_IP>:${OPEN_TERMINAL_PORT}"
echo "  API key: ${OPEN_TERMINAL_API_KEY}"
echo
echo "Useful commands:"
echo "  sudo systemctl status open-terminal.service"
echo "  sudo journalctl -u open-terminal.service -f"
echo "  sudo -l -U ${TERMINAL_USER}"
echo "============================================================"
SCRIPT
```

## What the script creates

After the script completes, the target host has a few important files and services.

### Open Terminal executable

For the default user, Open Terminal is installed here:

```bash
/home/username/.local/bin/open-terminal
```

The script installs it with `uv`:

```bash
uv tool install --upgrade open-terminal
```

### API key environment file

The generated API key is stored here:

```bash
/etc/open-terminal.env
```

The file contains a value like this:

```bash
OPEN_TERMINAL_API_KEY=p_generatedapikeyhere
```

The file permissions are locked down to:

```bash
0600
```

### systemd service

The Open Terminal service is created here:

```bash
/etc/systemd/system/open-terminal.service
```

The service is enabled at boot with:

```bash
systemctl enable open-terminal.service
```

It is started or restarted with:

```bash
systemctl restart open-terminal.service
```

### sudoers allowlist

The sudoers allowlist is written here:

```bash
/etc/sudoers.d/open-terminal-broad
```

It is validated with:

```bash
visudo -cf /etc/sudoers.d/open-terminal-broad
```

The permissions are set to:

```bash
0440
```

## Expected deployment output

A successful deployment should end with output similar to this:

```text
Open Terminal deployment complete.

Service user: username
Listen URL:   http://<TARGET_MACHINE_IP>:8054
Bind host:    0.0.0.0
Port:         8054
API key:      p_generatedapikeyhere

Add this to OpenWebUI using:
  URL:     http://<TARGET_MACHINE_IP>:8054
  API key: p_generatedapikeyhere
```

The service status should show:

```text
Active: active (running)
```

The listening socket check should show something similar to:

```text
LISTEN 0 2048 0.0.0.0:8054 0.0.0.0:*
```

The sudo allowlist checks should show:

```text
sudo systemctl status test: OK
sudo docker ps test: OK
```

The Docker test only appears if Docker is installed.

## Verifying the installation

After the script finishes, check the service:

```bash
sudo systemctl status open-terminal.service
```

A healthy service should show:

```text
Active: active (running)
```

Then check that the port is listening:

```bash
ss -ltnp | grep 8054
```

Expected output should include something like:

```text
LISTEN 0 2048 0.0.0.0:8054 0.0.0.0:*
```

## Verifying sudo access

Check the sudo rules for the service user:

```bash
sudo -l -U username
```

Test a systemd command without an interactive password prompt:

```bash
sudo -u username sudo -n /usr/bin/systemctl status open-terminal.service
```

If Docker is installed, test Docker access:

```bash
sudo -u username sudo -n /usr/bin/docker ps
```

If the commands work without asking for a password, the sudo allowlist is functioning.

## Adding it to OpenWebUI

Once Open Terminal is running, add it to OpenWebUI using the URL and API key printed by the script.

Example URL by hostname:

```text
http://network-services:8054
```

Example URL by IP:

```text
http://192.168.0.105:8054
```

The script prints the generated key at the end:

```text
API key: p_generatedapikeyhere
```

Use that key in OpenWebUI’s Open Terminal configuration.

## Changing the default user, port, or API key

The script supports environment variable overrides.

To run Open Terminal as a different user:

```bash
sudo TERMINAL_USER=myuser bash -s < deploy-open-terminal.sh
```

To use a different port:

```bash
sudo OPEN_TERMINAL_PORT=8055 bash -s < deploy-open-terminal.sh
```

To provide your own API key:

```bash
sudo OPEN_TERMINAL_API_KEY='p_mycustomapikey' bash -s < deploy-open-terminal.sh
```

To skip installing the sudoers allowlist:

```bash
sudo INSTALL_SUDOERS=0 bash -s < deploy-open-terminal.sh
```

To skip adding the user to the Docker group:

```bash
sudo ADD_DOCKER_GROUP=0 bash -s < deploy-open-terminal.sh
```

If using the copy/paste heredoc version, place the environment variable before the command, like this:

```bash
sudo OPEN_TERMINAL_PORT=8055 bash -s <<'SCRIPT'
# paste the deployment script here
SCRIPT
```

## Useful commands

Restart Open Terminal:

```bash
sudo systemctl restart open-terminal.service
```

Stop Open Terminal:

```bash
sudo systemctl stop open-terminal.service
```

Start Open Terminal:

```bash
sudo systemctl start open-terminal.service
```

View service logs:

```bash
sudo journalctl -u open-terminal.service -f
```

Check service status:

```bash
sudo systemctl status open-terminal.service
```

Check listening port:

```bash
ss -ltnp | grep 8054
```

Check sudo permissions:

```bash
sudo -l -U csadmin
```

View the API key file:

```bash
sudo cat /etc/open-terminal.env
```

Validate the sudoers file:

```bash
sudo visudo -cf /etc/sudoers.d/open-terminal-broad
```

## Troubleshooting

### APT update fails

One issue we encountered was a broken third-party APT repository.

The error looked like this:

```text
E: The repository 'http://www.deb-multimedia.org trixie InRelease' is not signed.
```

The script is designed to continue past this because third-party repo failures should not necessarily block Open Terminal deployment.

That said, the repository should still be fixed later by either installing the proper keyring, updating the repo configuration, or removing the repository if it is no longer needed.

### Open Terminal is not running

Check the service status:

```bash
sudo systemctl status open-terminal.service
```

Then check logs:

```bash
sudo journalctl -u open-terminal.service -n 100 --no-pager
```

If needed, restart the service:

```bash
sudo systemctl restart open-terminal.service
```

### Port 8054 is already in use

Check what is using the port:

```bash
ss -ltnp | grep 8054
```

If something else is already using it, rerun the deployment with a different port:

```bash
sudo OPEN_TERMINAL_PORT=8055 bash -s <<'SCRIPT'
# paste the deployment script here
SCRIPT
```

Then configure OpenWebUI to use the new port.

### Sudo still asks for a password

Validate the sudoers file:

```bash
sudo visudo -cf /etc/sudoers.d/open-terminal-broad
```

Check the user’s sudo rules:

```bash
sudo -l -U username
```

The allowed commands should show `NOPASSWD`.

Also check the sudoers file permissions:

```bash
ls -l /etc/sudoers.d/open-terminal-broad
```

The expected mode is:

```text
0440
```

### Docker commands fail

If Docker commands fail, confirm Docker is installed:

```bash
docker --version
```

Confirm the Docker service is running:

```bash
sudo systemctl status docker
```

Confirm the service user is in the Docker group:

```bash
id username
```

If the user was just added to the Docker group, the user may need to log out and back in for group membership to fully apply in interactive shells. The sudo allowlist should still allow `sudo docker` commands if configured correctly.

### OpenWebUI cannot connect

First verify that Open Terminal is listening:

```bash
ss -ltnp | grep 8054
```

Then test from the OpenWebUI host:

```bash
curl http://network-services:8054
```

If the hostname fails, try the IP address:

```bash
curl http://192.168.0.105:8054
```

If the IP works but the hostname does not, the problem is DNS or local name resolution.

### API key does not work

Confirm the API key stored on the host:

```bash
sudo cat /etc/open-terminal.env
```

Restart the service after changing the key:

```bash
sudo systemctl restart open-terminal.service
```

Then update the key in OpenWebUI.

## Security considerations

This setup is convenient, but it should be treated carefully.

The Open Terminal endpoint can run commands on the host. The API key should be protected like a password. The service should only be reachable from trusted systems, such as the OpenWebUI host or a private management network.

The sudoers policy is not full root access, but it is still broad. Review the allowed command aliases before deploying this widely.

Good follow-up improvements would be:

- restrict the Open Terminal listener with firewall rules,
- limit access to a management VLAN or VPN,
- rotate API keys periodically,
- reduce the sudoers allowlist per machine,
- avoid exposing port `8054` publicly,
- and document which OpenWebUI users are allowed to use terminal tools.

One important warning from the service output is that Open Terminal may allow all CORS origins by default:

```text
CORS is set to '*' (allow all origins)
```

For a private LAN or VPN-only deployment, this may be acceptable. For anything more exposed, restrict access with firewall rules or configure allowed origins if supported by the Open Terminal version in use.

## Final thoughts

The important part of this setup is that Open Terminal runs **baremetal** on the host. That gives OpenWebUI a clean way to interact with real host services without making the OpenWebUI container itself privileged.

To replicate this on another machine:

1. SSH into the target host.
2. Paste and run the deployment script.
3. Save the generated API key.
4. Add the endpoint to OpenWebUI.
5. Verify command execution.
6. Review and tighten the sudoers allowlist if needed.

This gives OpenWebUI enough access to be useful for administration while avoiding the most dangerous option: giving it blanket unrestricted root access.
