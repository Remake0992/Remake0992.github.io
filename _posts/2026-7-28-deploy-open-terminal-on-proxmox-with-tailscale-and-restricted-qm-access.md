---
title: Deploy Open Terminal on Proxmox with Tailscale and Restricted VM Management
date: 2026-7-28 15:05:18
categories: [self-hosting, proxmox]
tags: [open-terminal, tailscale, proxmox, systemd, sudo]
---

## Overview

This guide deploys Open WebUI's Open Terminal directly on a Proxmox VE host, publishes it privately through Tailscale Serve, and gives its dedicated Linux account limited access to selected Proxmox troubleshooting and VM-management commands.

The final design is:

```text
Open WebUI
    |
    | HTTPS over the Tailscale tailnet
    v
Tailscale Serve on the Proxmox host
    |
    | Local HTTP
    v
Open Terminal on 127.0.0.1:8000
    |
    | Restricted sudo wrappers
    v
Proxmox qm and diagnostic commands
```

TLS terminates at Tailscale Serve. Open Terminal listens on local HTTP and is not exposed directly on the LAN. Open Terminal runs under the unprivileged `open-terminal` account instead of `root`.

The included installer creates:

- A Python virtual environment under `/opt/open-terminal/venv`
- An Open Terminal configuration at `/etc/open-terminal/config.toml`
- A persistent systemd service
- A unique API key
- A Tailscale Serve HTTPS endpoint
- Restricted `qmlist`, `qmstatus`, and `qmrestart` commands
- Quick health-check and comprehensive diagnostic commands
- A root-readable credentials file

> **Warning:** Open Terminal allows an AI agent to execute commands and access files available to its Linux account. Do not run it as root or grant unrestricted passwordless sudo unless there is a documented requirement.

## Prerequisites

Before starting, confirm the following:

- Proxmox VE is installed and operational.
- Tailscale is installed and connected on the Proxmox host.
- The host is authorized in the intended tailnet.
- Tailnet HTTPS and MagicDNS are enabled.
- Open WebUI is available.
- You have root access to the Proxmox host.
- You know which VMIDs Open Terminal should be allowed to reboot.

Confirm Tailscale connectivity:

```bash
tailscale status
tailscale ip -4
```

Optionally assign a recognizable Tailscale hostname:

```bash
tailscale set --hostname=proxbox1
```

## Procedure

### Step 1: Save the installer

Log in to the Proxmox host:

```bash
ssh root@proxbox1
```

Create the installer:

```bash
nano /root/install-open-terminal.sh
```

Paste the script from the **Complete installation script** section into the file, save it, and make it executable:

```bash
chmod 700 /root/install-open-terminal.sh
```

### Step 2: Choose the permitted VMIDs

The installer supports an explicit VM allowlist. For example, to permit graceful reboots of VMs `100`, `101`, and `105`:

```bash
OPEN_TERMINAL_ALLOWED_VMIDS="100,101,105" \
  /root/install-open-terminal.sh
```

To allow every numeric VMID:

```bash
OPEN_TERMINAL_ALLOWED_VMIDS="*" \
  /root/install-open-terminal.sh
```

> **Important:** An explicit allowlist is safer than `*`. The account can list and inspect all VMs, but `qmrestart` will only reboot VMIDs in the allowlist.

Optional installer variables are:

| Variable | Default | Purpose |
|---|---:|---|
| `OPEN_TERMINAL_PORT` | `8000` | Local Open Terminal port |
| `OPEN_TERMINAL_API_KEY` | Generated or preserved | Explicit API key |
| `OPEN_TERMINAL_WORKSPACE` | `/srv/open-terminal` | Working directory |
| `OPEN_TERMINAL_ALLOWED_VMIDS` | `*` | Comma-separated reboot allowlist |
| `OPEN_TERMINAL_CORS_ORIGINS` | `*` | Allowed browser origins |
| `CONFIGURE_TAILSCALE_SERVE` | `true` | Whether to configure Serve |

### Step 3: Retrieve the connection information

After installation completes, display the generated credentials:

```bash
cat /root/open-terminal-credentials.txt
```

The file contains the Tailscale URL and Open Terminal API key. It is protected with mode `600` and is readable only by root.

### Step 4: Add the connection to Open WebUI

In Open WebUI, open either:

```text
User Settings → Integrations → Open Terminal
```

or:

```text
Admin Settings → Integrations → Open Terminal
```

Enter:

```text
Name: Proxmox - proxbox1
URL: https://proxbox1.your-tailnet.ts.net
API key: sk-generated-key
```

Use the exact URL returned by `tailscale serve status`. Copy the API key without quotation marks or spaces.

### Step 5: Test command execution

From Open WebUI, ask the model to run:

```bash
whoami
hostname
pwd
pveversion
sudo /usr/local/sbin/qmlist
sudo /usr/local/sbin/open-terminal-healthcheck
```

The expected user is:

```text
open-terminal
```

## Complete installation script

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
umask 077

# Optional environment variables:
#   OPEN_TERMINAL_PORT=8000
#   OPEN_TERMINAL_API_KEY="sk-existing-key"
#   OPEN_TERMINAL_WORKSPACE="/srv/open-terminal"
#   OPEN_TERMINAL_ALLOWED_VMIDS="100,101"
#   OPEN_TERMINAL_CORS_ORIGINS="*"
#   CONFIGURE_TAILSCALE_SERVE=true

SERVICE_NAME="open-terminal"
SERVICE_USER="open-terminal"
SERVICE_GROUP="open-terminal"

INSTALL_DIR="/opt/open-terminal"
VENV_DIR="${INSTALL_DIR}/venv"
CONFIG_DIR="/etc/open-terminal"
CONFIG_FILE="${CONFIG_DIR}/config.toml"
DATA_DIR="/var/lib/open-terminal"
WORKSPACE="${OPEN_TERMINAL_WORKSPACE:-/srv/open-terminal}"
PORT="${OPEN_TERMINAL_PORT:-8000}"
LOCAL_URL="http://127.0.0.1:${PORT}"
CORS_ORIGINS="${OPEN_TERMINAL_CORS_ORIGINS:-*}"
ALLOWED_VMIDS="${OPEN_TERMINAL_ALLOWED_VMIDS:-*}"
CONFIGURE_TAILSCALE_SERVE="${CONFIGURE_TAILSCALE_SERVE:-true}"

SYSTEMD_FILE="/etc/systemd/system/open-terminal.service"
SUDOERS_FILE="/etc/sudoers.d/open-terminal-proxmox"
CREDENTIAL_FILE="/root/open-terminal-credentials.txt"

QMLIST_WRAPPER="/usr/local/sbin/qmlist"
QMSTATUS_WRAPPER="/usr/local/sbin/qmstatus"
QMRESTART_WRAPPER="/usr/local/sbin/qmrestart"
HEALTHCHECK_WRAPPER="/usr/local/sbin/open-terminal-healthcheck"
DIAGNOSTICS_WRAPPER="/usr/local/sbin/open-terminal-diagnostics"

log() {
    printf '\n\033[1;34m==>\033[0m %s\n' "$*"
}

success() {
    printf '\033[1;32m✔\033[0m %s\n' "$*"
}

warning() {
    printf '\033[1;33mWARNING:\033[0m %s\n' "$*" >&2
}

fatal() {
    printf '\033[1;31mERROR:\033[0m %s\n' "$*" >&2
    exit 1
}

show_failure_details() {
    warning "Installation encountered an error."
    systemctl status "${SERVICE_NAME}" --no-pager --full 2>/dev/null || true
    journalctl -u "${SERVICE_NAME}" -n 100 --no-pager --full 2>/dev/null || true
}

trap show_failure_details ERR

if [[ "${EUID}" -ne 0 ]]; then
    fatal "Run this installer as root."
fi

if ! [[ "${PORT}" =~ ^[0-9]+$ ]] || (( PORT < 1 || PORT > 65535 )); then
    fatal "OPEN_TERMINAL_PORT must be between 1 and 65535."
fi

if [[ ! "${WORKSPACE}" =~ ^/ ]]; then
    fatal "OPEN_TERMINAL_WORKSPACE must be an absolute path."
fi

if [[ ! "${ALLOWED_VMIDS}" =~ ^(\*|[0-9]+(,[0-9]+)*)$ ]]; then
    fatal "OPEN_TERMINAL_ALLOWED_VMIDS must be '*' or comma-separated VMIDs."
fi

if [[ "${WORKSPACE}" == *\"* || "${WORKSPACE}" == *$'\n'* ]]; then
    fatal "The workspace cannot contain quotation marks or newlines."
fi

if [[ "${CORS_ORIGINS}" == *\"* || "${CORS_ORIGINS}" == *$'\n'* ]]; then
    fatal "CORS origins cannot contain quotation marks or newlines."
fi

log "Installing prerequisites"
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y \
    python3 python3-venv python3-pip \
    openssl curl ca-certificates sudo jq \
    dnsutils iproute2 lsof netcat-openbsd procps

[[ -x /usr/sbin/qm ]] || fatal "/usr/sbin/qm was not found."
command -v tailscale >/dev/null 2>&1 || fatal "Tailscale is not installed."
systemctl enable --now tailscaled
tailscale status >/dev/null 2>&1 || fatal "Tailscale is not connected."

log "Creating the service account and directories"
getent group "${SERVICE_GROUP}" >/dev/null || groupadd --system "${SERVICE_GROUP}"

if ! id "${SERVICE_USER}" >/dev/null 2>&1; then
    useradd \
        --system \
        --gid "${SERVICE_GROUP}" \
        --create-home \
        --home-dir "${DATA_DIR}" \
        --shell /bin/bash \
        "${SERVICE_USER}"
fi

install -d -o "${SERVICE_USER}" -g "${SERVICE_GROUP}" -m 0755 "${INSTALL_DIR}"
install -d -o "${SERVICE_USER}" -g "${SERVICE_GROUP}" -m 0700 "${CONFIG_DIR}"
install -d -o "${SERVICE_USER}" -g "${SERVICE_GROUP}" -m 0700 "${DATA_DIR}"
install -d -o "${SERVICE_USER}" -g "${SERVICE_GROUP}" -m 0750 "${WORKSPACE}"

log "Installing Open Terminal"
if [[ ! -x "${VENV_DIR}/bin/python" ]]; then
    runuser -u "${SERVICE_USER}" -- python3 -m venv "${VENV_DIR}"
fi

runuser -u "${SERVICE_USER}" -- \
    "${VENV_DIR}/bin/python" -m pip install --upgrade pip setuptools wheel

runuser -u "${SERVICE_USER}" -- \
    "${VENV_DIR}/bin/python" -m pip install --upgrade open-terminal

[[ -x "${VENV_DIR}/bin/open-terminal" ]] || fatal "Open Terminal executable is missing."

log "Preparing the API key"
API_KEY="${OPEN_TERMINAL_API_KEY:-}"

if [[ -z "${API_KEY}" && -f "${CONFIG_FILE}" ]]; then
    API_KEY="$(
        sed -nE \
            's/^[[:space:]]*api_key[[:space:]]*=[[:space:]]*"([^"]+)".*$/\1/p' \
            "${CONFIG_FILE}" | head -n 1
    )"
fi

if [[ -z "${API_KEY}" ]]; then
    API_KEY="sk-$(openssl rand -hex 32)"
fi

if [[ "${API_KEY}" == *\"* || "${API_KEY}" == *$'\n'* ]]; then
    fatal "The API key cannot contain quotation marks or newlines."
fi

log "Writing Open Terminal configuration"
if [[ -f "${CONFIG_FILE}" ]]; then
    cp -a "${CONFIG_FILE}" \
        "${CONFIG_FILE}.backup.$(date +%Y%m%d-%H%M%S)"
fi

cat > "${CONFIG_FILE}" <<EOF
host = "127.0.0.1"
port = ${PORT}
api_key = "${API_KEY}"
cors_allowed_origins = "${CORS_ORIGINS}"
log_dir = "${DATA_DIR}"
file_browser_root = "${WORKSPACE}"
EOF

chown "${SERVICE_USER}:${SERVICE_GROUP}" "${CONFIG_FILE}"
chmod 0600 "${CONFIG_FILE}"

log "Installing qmlist"
cat > "${QMLIST_WRAPPER}" <<'EOF'
#!/bin/bash
set -euo pipefail
[[ $# -eq 0 ]] || { echo "Usage: sudo qmlist" >&2; exit 2; }
exec /usr/sbin/qm list
EOF

log "Installing qmstatus"
cat > "${QMSTATUS_WRAPPER}" <<'EOF'
#!/bin/bash
set -euo pipefail
[[ $# -eq 1 ]] || { echo "Usage: sudo qmstatus <VMID>" >&2; exit 2; }
VMID="$1"
[[ "${VMID}" =~ ^[0-9]+$ ]] || { echo "Invalid VMID" >&2; exit 2; }
exec /usr/sbin/qm status "${VMID}"
EOF

log "Installing qmrestart"
cat > "${QMRESTART_WRAPPER}" <<EOF
#!/bin/bash
set -euo pipefail
ALLOWED_VMIDS="${ALLOWED_VMIDS}"
[[ \$# -eq 1 ]] || { echo "Usage: sudo qmrestart <VMID>" >&2; exit 2; }
VMID="\$1"
[[ "\${VMID}" =~ ^[0-9]+\$ ]] || { echo "Invalid VMID" >&2; exit 2; }

if [[ "\${ALLOWED_VMIDS}" != "*" ]]; then
    AUTHORIZED=false
    IFS=',' read -ra VMID_LIST <<< "\${ALLOWED_VMIDS}"
    for ALLOWED_VMID in "\${VMID_LIST[@]}"; do
        [[ "\${VMID}" == "\${ALLOWED_VMID}" ]] && AUTHORIZED=true
    done
    [[ "\${AUTHORIZED}" == true ]] || {
        echo "VMID \${VMID} is not authorized for restart." >&2
        exit 1
    }
fi

/usr/sbin/qm status "\${VMID}" >/dev/null
logger -t open-terminal-qmrestart -- \
    "requester=\${SUDO_USER:-unknown} vmid=\${VMID} action=reboot"
exec /usr/sbin/qm reboot "\${VMID}"
EOF

log "Installing the health check"
cat > "${HEALTHCHECK_WRAPPER}" <<EOF
#!/bin/bash
set -u
PORT="${PORT}"
LOCAL_URL="http://127.0.0.1:${PORT}"
FAIL=0

check() {
    if eval "\$2"; then
        echo "[PASS] \$1"
    else
        echo "[FAIL] \$1"
        FAIL=1
    fi
}

check "open-terminal.service is active" \
    "systemctl is-active --quiet open-terminal"
check "TCP port \${PORT} is listening" \
    "ss -H -lnt 'sport = :\${PORT}' | grep -q ."
check "Local HTTP endpoint responds" \
    "curl -sS -o /dev/null --connect-timeout 3 --max-time 5 \${LOCAL_URL}/"
check "tailscaled.service is active" \
    "systemctl is-active --quiet tailscaled"
check "Tailscale is connected" \
    "tailscale status >/dev/null 2>&1"
check "Tailscale Serve uses port \${PORT}" \
    "tailscale serve status 2>/dev/null | grep -q '127.0.0.1:\${PORT}'"

exit "\${FAIL}"
EOF

log "Installing comprehensive diagnostics"
cat > "${DIAGNOSTICS_WRAPPER}" <<EOF
#!/bin/bash
set -u
PORT="${PORT}"

section() {
    echo
    echo "================================================================"
    echo "\$1"
    echo "================================================================"
}

run() {
    echo
    echo "--- \$1 ---"
    shift
    timeout 20 "\$@" 2>&1 || echo "[Command exited with status \$?]"
}

section "HOST"
run "Date" date
run "Hostname" hostnamectl
run "Uptime" uptime
run "Proxmox version" pveversion --verbose
run "Memory" free -h
run "Filesystems" df -hT
run "Inodes" df -ih
run "Block devices" lsblk
run "Failed units" systemctl --failed --no-pager --full

section "PROXMOX"
run "Storage" pvesm status
command -v pvecm >/dev/null && run "Cluster" pvecm status
run "Virtual machines" qm list
command -v pct >/dev/null && run "Containers" pct list

section "OPEN TERMINAL"
run "Service status" systemctl status open-terminal --no-pager --full
run "Recent logs" journalctl -u open-terminal -n 100 --no-pager --full
run "Listening port" bash -c "ss -lntp | grep ':${PORT}' || true"
run "Local HTTP" curl -v --connect-timeout 5 --max-time 10 \
    "http://127.0.0.1:${PORT}/"

section "NETWORK AND TAILSCALE"
run "Addresses" ip address show
run "Routes" ip route show
run "DNS" cat /etc/resolv.conf
run "Listening sockets" ss -lntp
run "Tailscale status" tailscale status
run "Tailscale IP" tailscale ip
run "Tailscale Serve" tailscale serve status

section "RECENT WARNINGS"
run "Kernel warnings" journalctl -k -p warning --since "-2 hours" --no-pager
run "System warnings" journalctl -p warning --since "-2 hours" --no-pager

section "CONFIGURATION WITHOUT API KEY"
if [[ -f /etc/open-terminal/config.toml ]]; then
    sed '/^[[:space:]]*api_key[[:space:]]*=/d' \
        /etc/open-terminal/config.toml
else
    echo "Configuration file is missing."
fi

section "HEALTH SUMMARY"
/usr/local/sbin/open-terminal-healthcheck || true
EOF

chown root:root \
    "${QMLIST_WRAPPER}" \
    "${QMSTATUS_WRAPPER}" \
    "${QMRESTART_WRAPPER}" \
    "${HEALTHCHECK_WRAPPER}" \
    "${DIAGNOSTICS_WRAPPER}"

chmod 0755 \
    "${QMLIST_WRAPPER}" \
    "${QMSTATUS_WRAPPER}" \
    "${QMRESTART_WRAPPER}" \
    "${HEALTHCHECK_WRAPPER}" \
    "${DIAGNOSTICS_WRAPPER}"

log "Installing restricted sudo rules"
cat > "${SUDOERS_FILE}" <<EOF
${SERVICE_USER} ALL=(root) NOPASSWD: ${QMLIST_WRAPPER}
${SERVICE_USER} ALL=(root) NOPASSWD: ${QMSTATUS_WRAPPER}
${SERVICE_USER} ALL=(root) NOPASSWD: ${QMRESTART_WRAPPER}
${SERVICE_USER} ALL=(root) NOPASSWD: ${HEALTHCHECK_WRAPPER}
${SERVICE_USER} ALL=(root) NOPASSWD: ${DIAGNOSTICS_WRAPPER}
EOF

chown root:root "${SUDOERS_FILE}"
chmod 0440 "${SUDOERS_FILE}"
visudo -cf /etc/sudoers

log "Installing the systemd service"
cat > "${SYSTEMD_FILE}" <<EOF
[Unit]
Description=Open WebUI Open Terminal
Documentation=https://github.com/open-webui/open-terminal
Wants=network-online.target
After=network-online.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=simple
User=${SERVICE_USER}
Group=${SERVICE_GROUP}
WorkingDirectory=${WORKSPACE}
Environment=HOME=${DATA_DIR}
Environment=PYTHONUNBUFFERED=1
ExecStart=${VENV_DIR}/bin/open-terminal run --config ${CONFIG_FILE} --cwd ${WORKSPACE}
Restart=on-failure
RestartSec=10
TimeoutStopSec=30
StandardOutput=journal
StandardError=journal
SyslogIdentifier=open-terminal

[Install]
WantedBy=multi-user.target
EOF

chown root:root "${SYSTEMD_FILE}"
chmod 0644 "${SYSTEMD_FILE}"
systemctl daemon-reload
systemctl enable --now "${SERVICE_NAME}"

log "Waiting for Open Terminal"
READY=false
HTTP_CODE=000

for attempt in $(seq 1 30); do
    HTTP_CODE="$(
        curl -sS -o /dev/null -w '%{http_code}' \
            --connect-timeout 1 --max-time 2 "${LOCAL_URL}/" || true
    )"

    if [[ -n "${HTTP_CODE}" && "${HTTP_CODE}" != "000" ]]; then
        READY=true
        break
    fi
    sleep 1
done

[[ "${READY}" == true ]] || fatal "Open Terminal did not answer on ${LOCAL_URL}."
success "Open Terminal answered with HTTP status ${HTTP_CODE}."

log "Checking the listener"
ss -lntp "sport = :${PORT}" || true

if ss -H -lnt "sport = :${PORT}" | grep -q '0.0.0.0'; then
    warning "Open Terminal is listening on 0.0.0.0 instead of loopback."
fi

if [[ "${CONFIGURE_TAILSCALE_SERVE}" == true ]]; then
    log "Existing Tailscale Serve configuration"
    tailscale serve status 2>/dev/null || true

    log "Configuring Tailscale Serve"
    tailscale serve --bg "${LOCAL_URL}"
fi

TAILSCALE_STATUS="$(tailscale serve status 2>/dev/null || true)"
TAILSCALE_URL="$(
    printf '%s\n' "${TAILSCALE_STATUS}" |
        grep -Eo 'https://[^[:space:]]+' | head -n 1 || true
)"

cat > "${CREDENTIAL_FILE}" <<EOF
Open Terminal connection information
====================================

Host: $(hostname --fqdn 2>/dev/null || hostname)
Local URL: ${LOCAL_URL}
Tailscale URL: ${TAILSCALE_URL:-Run tailscale serve status}
API key: ${API_KEY}
Configuration: ${CONFIG_FILE}
Workspace: ${WORKSPACE}
Allowed restart VMIDs: ${ALLOWED_VMIDS}

Restricted commands:
sudo /usr/local/sbin/qmlist
sudo /usr/local/sbin/qmstatus <VMID>
sudo /usr/local/sbin/qmrestart <VMID>
sudo /usr/local/sbin/open-terminal-healthcheck
sudo /usr/local/sbin/open-terminal-diagnostics
EOF

chown root:root "${CREDENTIAL_FILE}"
chmod 0600 "${CREDENTIAL_FILE}"

log "Testing restricted access"
sudo -u "${SERVICE_USER}" sudo -n "${QMLIST_WRAPPER}" >/dev/null
sudo -u "${SERVICE_USER}" sudo -n "${HEALTHCHECK_WRAPPER}" || true

trap - ERR

printf '\nInstallation completed.\n\n'
printf 'Connection details: cat %s\n' "${CREDENTIAL_FILE}"
printf 'Service status: systemctl status %s\n' "${SERVICE_NAME}"
printf 'Health check: sudo %s\n' "${HEALTHCHECK_WRAPPER}"
printf 'Diagnostics: sudo %s\n' "${DIAGNOSTICS_WRAPPER}"
printf '\n'
tailscale serve status || true
```

## Restricted Proxmox commands

The installer does not grant direct unrestricted access to `/usr/sbin/qm`. Instead, it creates root-owned wrappers and authorizes only those wrappers in `/etc/sudoers.d/open-terminal-proxmox`.

### List virtual machines

```bash
sudo /usr/local/sbin/qmlist
```

This runs:

```bash
/usr/sbin/qm list
```

### Check VM status

```bash
sudo /usr/local/sbin/qmstatus 100
```

### Gracefully reboot a VM

```bash
sudo /usr/local/sbin/qmrestart 100
```

The wrapper validates that the argument is numeric, verifies that the VM exists, checks the VMID allowlist, logs the request, and runs:

```bash
/usr/sbin/qm reboot 100
```

Review reboot requests with:

```bash
journalctl -t open-terminal-qmrestart
```

> **Note:** `qmrestart` is a custom wrapper name. Proxmox uses `qm reboot` for a graceful guest reboot. A hard reset would use `qm reset`, but that is intentionally not granted by this deployment.

## Validation

### Confirm the service

```bash
systemctl status open-terminal --no-pager -l
journalctl -u open-terminal -n 100 --no-pager -l
```

Expected result:

```text
Active: active (running)
Uvicorn running on http://127.0.0.1:8000
```

### Confirm the local listener

```bash
ss -lntp | grep ':8000'
curl -v http://127.0.0.1:8000/
```

An HTTP response confirms local connectivity. A `401 Unauthorized` response can be expected when the request does not provide the API key.

### Confirm Tailscale Serve

```bash
tailscale serve status
```

Expected configuration:

```text
https://proxbox1.your-tailnet.ts.net/
|-- proxy http://127.0.0.1:8000
```

Test from another tailnet device:

```bash
curl -v https://proxbox1.your-tailnet.ts.net/
```

### Confirm restricted sudo access

```bash
sudo -l -U open-terminal
sudo -u open-terminal sudo -n /usr/local/sbin/qmlist
sudo -u open-terminal sudo -n /usr/local/sbin/qmstatus 100
```

Only execute the reboot test when a VM can safely be restarted:

```bash
sudo -u open-terminal sudo -n /usr/local/sbin/qmrestart 100
```

### Run the health checks

```bash
sudo /usr/local/sbin/open-terminal-healthcheck
```

Run comprehensive diagnostics:

```bash
sudo /usr/local/sbin/open-terminal-diagnostics
```

Save the diagnostics to the Open Terminal workspace:

```bash
sudo /usr/local/sbin/open-terminal-diagnostics \
  > /srv/open-terminal/proxbox1-diagnostics.txt 2>&1
```

The file can then be reviewed or downloaded through Open Terminal.

## Troubleshooting

### Open Terminal reports connection refused

Check the actual listening port:

```bash
journalctl -u open-terminal -b -l --no-pager \
  | grep 'Uvicorn running' \
  | tail -1

ss -lntp | grep open-terminal
```

If the service listens on a different port, inspect both the configuration and systemd command:

```bash
cat /etc/open-terminal/config.toml
systemctl cat open-terminal
```

The service command must not contain accidental text such as `-->` after the configuration path.

After correcting it:

```bash
systemctl daemon-reload
systemctl restart open-terminal
sleep 5
curl -v http://127.0.0.1:8000/
```

### Tailscale proxies HTTPS to the local backend

An incorrect configuration may show:

```text
proxy https://127.0.0.1:8000
```

Open Terminal normally provides local HTTP while Tailscale terminates HTTPS. Correct it with:

```bash
tailscale serve --bg http://127.0.0.1:8000
tailscale serve status
```

Do not use `tailscale serve reset` casually. It removes the host's complete Serve configuration, including unrelated routes.

### Open WebUI reports “Server connection failed”

Validate each layer in order:

```bash
systemctl status open-terminal --no-pager -l
curl -v http://127.0.0.1:8000/
tailscale serve status
curl -v https://proxbox1.your-tailnet.ts.net/
```

If Open WebUI runs in Docker, test from its container:

```bash
docker exec -it open-webui curl -v \
  https://proxbox1.your-tailnet.ts.net/
```

Browser access alone does not prove that the Open WebUI backend can reach Open Terminal. The backend or container must be able to resolve and route to the Tailscale URL.

### Authentication fails

Display the configured key as root:

```bash
grep '^api_key' /etc/open-terminal/config.toml
```

Verify that:

- Open WebUI uses the same key.
- The value was copied without quotation marks.
- The key contains no accidental spaces.
- Open Terminal was restarted after changing the configuration.
- The Open WebUI connection was saved after updating the key.

Restart Open Terminal if needed:

```bash
systemctl restart open-terminal
```

### `qmrestart` reports that a VMID is unauthorized

Review the installed allowlist:

```bash
grep '^ALLOWED_VMIDS=' /usr/local/sbin/qmrestart
```

Rerun the installer with the desired list:

```bash
OPEN_TERMINAL_ALLOWED_VMIDS="100,101,105" \
  /root/install-open-terminal.sh
```

### Sudo requests a password

Validate sudoers:

```bash
visudo -cf /etc/sudoers
sudo -l -U open-terminal
```

Confirm permissions:

```bash
stat /etc/sudoers.d/open-terminal-proxmox
```

The file should be owned by `root:root` with mode `0440`.

### Collect a diagnostic bundle

```bash
sudo /usr/local/sbin/open-terminal-diagnostics \
  > /root/open-terminal-diagnostics.txt 2>&1
```

The diagnostic command removes the API key line before displaying the Open Terminal configuration.

## Updating Open Terminal

The installer is designed to be rerun. It preserves an existing API key, backs up the configuration, upgrades the Python package, refreshes the wrappers, and restarts the service.

```bash
OPEN_TERMINAL_ALLOWED_VMIDS="100,101,105" \
  /root/install-open-terminal.sh
```

A manual package-only update can be performed with:

```bash
systemctl stop open-terminal
runuser -u open-terminal -- \
  /opt/open-terminal/venv/bin/python -m pip install --upgrade open-terminal
systemctl start open-terminal
```

## Security Considerations

- Keep Open Terminal bound to `127.0.0.1`.
- Use Tailscale Serve rather than Funnel so the endpoint remains private.
- Keep API authentication enabled even on the tailnet.
- Use unique API keys on different Proxmox hosts.
- Protect `/etc/open-terminal/config.toml` with mode `600`.
- Restrict Tailscale access with ACLs or grants.
- Prefer an explicit VMID restart allowlist.
- Do not grant `open-terminal` unrestricted `NOPASSWD: ALL`.
- Review `journalctl -t open-terminal-qmrestart` for reboot activity.
- Review Open Terminal and Open WebUI logs for unexpected command execution.
- Remember that `file_browser_root` is a user-interface starting point, not a security boundary.

## Related Resources

- [Open Terminal GitHub repository](https://github.com/open-webui/open-terminal)
- [Tailscale Serve documentation](https://tailscale.com/kb/1242/tailscale-serve)
- [Tailscale access control documentation](https://tailscale.com/kb/1018/acls)
- [Proxmox VE qm manual](https://pve.proxmox.com/pve-docs/qm.1.html)
- [Deploy Open Terminal on macOS with Tailscale HTTPS](https://cjs-cloud.com/posts/deploy-open-terminal-on-macos-with-tailscale-https/)
