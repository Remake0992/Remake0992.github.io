---
title: Deploy Open Terminal on Windows with Tailscale and Open WebUI
date: 2026-07-28 00:00:00
categories: [self-hosting, windows]
tags: [open-terminal, tailscale, open-webui, powershell, automation]
---

## Overview

This article documents a Windows installer for Open Terminal that runs under a dedicated unprivileged local account, listens only on `127.0.0.1:8000`, and is published privately through Tailscale Serve for use with Open WebUI.

The installer was written to avoid several Windows-specific failure modes discovered during troubleshooting:

- Microsoft Store / per-user Python installations
- Broken virtual environments referencing another user profile
- PowerShell stderr handling that hid the real traceback
- Unicode console encoding failures from Open Terminal's banner output
- Winget output leaking into PowerShell return values

> **Note:** The instructions below are based on the Windows installer and troubleshooting steps developed during the conversation. The Proxmox article in the attached source is a separate Linux guide and is not the basis for this Windows deployment.

## Purpose

The goal of the installer is to provide a repeatable way to deploy Open Terminal for Open WebUI on Windows while automatically handling:

- machine-wide Python detection or installation
- creation and repair of the virtual environment
- creation of a dedicated `open-terminal` service account
- registration of a persistent Scheduled Task
- UTF-8 startup handling for Open Terminal's Unicode output
- local-only HTTP exposure on `127.0.0.1`
- Tailscale Serve configuration for private HTTPS access
- generation of Open WebUI connection information
- preservation or creation of the Open Terminal API key

## Architecture Overview

The intended architecture is:

1. Open Terminal runs locally on the Windows host.
2. The service binds to `127.0.0.1:8000`.
3. Tailscale Serve exposes that local endpoint privately over HTTPS.
4. Open WebUI connects to the Tailscale HTTPS URL using the Open Terminal API key.
5. Open Terminal runs under the dedicated `open-terminal` account rather than the Administrator account.

## Prerequisites

Before running the installer, confirm the following:

- Windows PowerShell 5.1 is available.
- You have local Administrator access.
- Tailscale is installed and connected.
- Open WebUI is available and will use the generated URL and API key.
- Python 3.10 or newer is installed machine-wide, or Winget is available to install it.

> **Important:** The installer is designed for a machine-wide Python installation such as `C:\Program Files\Python312\python.exe`. It should not rely on Microsoft Store Python or a user-scoped Python installation.

## Procedure

### Step 1: Save the installer

Save the unified PowerShell installer as:

```text
C:\Users\Administrator\Install-OpenTerminal-Windows.ps1
```

The script should include all of the following behaviors:

- machine-wide Python detection with a Winget fallback
- explicit `-PythonPath` support
- creation of the `open-terminal` local account
- `SeBatchLogonRight` assignment
- virtual environment repair logic
- UTF-8 startup wrapper
- Tailscale Serve configuration
- Open WebUI connection file generation

### Step 2: Run the installer

If Python is already installed machine-wide, run:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Users\Administrator\Install-OpenTerminal-Windows.ps1 -PythonPath "C:\Program Files\Python312\python.exe"
```

If you want the script to locate or install Python automatically, run:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Users\Administrator\Install-OpenTerminal-Windows.ps1
```

### Step 3: Review the generated connection information

After the installer completes successfully, review the generated credentials:

```powershell
Get-Content C:\ProgramData\OpenTerminal\credentials.txt
```

The same information is also written to:

```text
C:\ProgramData\OpenTerminal\open-webui-connection.json
```

### Step 4: Add the connection to Open WebUI

Use the following values in Open WebUI:

- **Name:** `Windows - <hostname>`
- **URL:** the exact HTTPS URL returned by `tailscale serve status`
- **API key:** the key written to `config.toml` and the credentials file

Copy the API key without extra quotes or spaces.

### Step 5: Validate the deployment

Confirm that Open Terminal is running and listening locally:

```powershell
Get-ScheduledTask -TaskName "Open Terminal" | Select-Object TaskName, State
Get-ScheduledTaskInfo -TaskName "Open Terminal" | Select-Object LastRunTime, LastTaskResult
Get-NetTCPConnection -LocalPort 8000 -State Listen
curl.exe -v http://127.0.0.1:8000/
```

A healthy result should show the task as running and the local HTTP endpoint responding.

Then verify Tailscale Serve:

```powershell
tailscale.exe serve status
```

## Validation

A successful installation should produce all of the following:

- a running Scheduled Task named `Open Terminal`
- Open Terminal listening on `127.0.0.1:8000`
- a valid local HTTP response
- a Tailscale HTTPS Serve URL
- protected connection files in `C:\ProgramData\OpenTerminal`
- Open WebUI able to reach the endpoint using the generated URL and API key

## Troubleshooting

### Winget installs Python but the installer still fails

The installer should now capture Winget output separately so it does not get treated as a PowerShell command. If this happens again, rerun the installer with an explicit `-PythonPath` value.

### Open Terminal exits with a UnicodeEncodeError

This means the startup wrapper is not forcing UTF-8 correctly. The wrapper must set:

```powershell
$env:PYTHONUTF8 = "1"
$env:PYTHONIOENCODING = "utf-8"
```

and set `Console.OutputEncoding` to UTF-8.

### The virtual environment still references a user-scoped Python

Delete and recreate the virtual environment using machine-wide Python. The script should detect `\Users\` and `\WindowsApps\` in `pyvenv.cfg` and rebuild the environment automatically.

### The task runs but Open Terminal is not reachable

Check the logs written by the startup wrapper:

```powershell
Get-Content C:\ProgramData\OpenTerminal\logs\startup.log -Tail 200
Get-Content C:\ProgramData\OpenTerminal\logs\open-terminal.stdout.log -Tail 200
Get-Content C:\ProgramData\OpenTerminal\logs\open-terminal.stderr.log -Tail 200
```

If the task is healthy but Open WebUI still fails, confirm that the Tailscale Serve URL matches the exact output of `tailscale serve status`.

## Security Considerations

- Keep Open Terminal bound to `127.0.0.1`.
- Use Tailscale Serve rather than exposing the service on the LAN.
- Keep API authentication enabled.
- Restrict the API key and connection files to Administrators.
- Do not run Open Terminal as root-equivalent or Administrator.
- Preserve the service account principle of least privilege.

## Related Resources
- [Open Terminal](https://github.com/open-webui/open-terminal)
- [Tailscale Serve](https://tailscale.com/kb/1247/serve)
- [Open WebUI documentation](https://docs.openwebui.com/)

