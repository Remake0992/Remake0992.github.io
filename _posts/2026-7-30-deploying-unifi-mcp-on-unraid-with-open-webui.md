---
title: AI-Powered UniFi Network Management with MCP — Deploying on Unraid
date: 2026-7-30 09:20:00
categories:
  - homelab
  - self-hosting
  - ai
tags:
  - unifi
  - mcp
  - docker
  - unraid
  - open-webui
  - portainer
---

![Network](https://images.pexels.com/photos/8386440/pexels-photo-8386440.jpeg)

## My network has a new administrator

I manage a UniFi stack at home — a self-hosted controller running on `network-services`, a handful of switches, and a couple of access points. It's not a massive deployment, but it's enough that I don't want to click through the UniFi web UI every time I need to check which client is hogging bandwidth or audit a firewall rule. I wanted something smarter.

That's where [UniFi MCP](https://github.com/sirkirby/unifi-mcp) comes in. It's a collection of Model Context Protocol servers that expose your UniFi controller to AI assistants — 186 tools for Network alone, plus optional servers for Protect and Access. The idea is simple: ask your AI agent in plain English and it queries the controller, previews changes, and applies them with your approval.

This article walks through deploying all three UniFi MCP servers on Unraid via a Portainer stack and connecting them to Open WebUI.

## What UniFi MCP gives you

Each UniFi application gets its own MCP server:

| Server | Status | Tools | What it can do |
|--------|--------|-------|----------------|
| Network | Stable | 186 | Clients, devices, firewall rules, VLANs, VPNs, traffic flows, dashboard stats |
| Protect | Beta | 61 | Camera listings, motion events, smart detections, footage retrieval |
| Access | Beta | 36 | Door status, badge scans, visitor passes, access policies |

All mutations — firewall changes, device renames, client blocks — go through a **preview-then-confirm** flow. The AI shows you exactly what will change before anything touches your controller. Secrets like Wi-Fi passphrases and VPN keys are redacted by default, returning `***REDACTED***` in responses [1].

## Architecture

The result looks like this:

```text
Open WebUI
    │
    │  Streamable HTTP (POST /mcp)
    ▼
┌──────────────────────────────┐
│  unifi-network-mcp  :3000    │
│  unifi-protect-mcp  :3001    │    Docker containers on Unraid
│  unifi-access-mcp   :3002    │    (Portainer stack)
└──────────────┬───────────────┘
               │
               │  HTTPS :11443
               │  (local admin, no MFA)
               ▼
       ┌───────────────────┐
       │  UniFi Controller │
       │  network-services │
       │  192.168.0.105    │
       └───────────────────┘
```

The MCP servers run as Docker containers on Unraid. Open WebUI connects to them over the Docker network using Streamable HTTP. The containers authenticate to the UniFi controller with a dedicated local admin account — no SSO, no MFA.

## Prerequisites

Before deploying, you need:

- A UniFi controller reachable from your Docker host.
- A **local admin account** on the controller. Do not use a Ubiquiti SSO cloud account. Create one under *Settings → Admins → Add Admin*, give it Administrator role, and leave MFA/2FA disabled — the MCP servers cannot handle MFA challenges [1].
- Unraid with Docker and Portainer.
- Open WebUI 0.6.31 or later, with `WEBUI_SECRET_KEY` set on the container.

My controller lives at `192.168.0.105:11443` with the hostname `network-services`. It runs UniFi OS, which the MCP servers auto-detect on first connection.

## The service account

I created a dedicated local admin called `Unifi-MCP` on the controller. Giving it a distinct name makes it obvious in audit logs which actions were initiated by the AI versus a human admin.

The account has full Administrator rights because the MCP servers need broad read/write access to function properly. If you want to lock things down further, the project supports category-level tool permissions — you could disable high-risk tools like firewall mutation while keeping read-only queries available [1].

## Portainer stack

Here's the stack I deployed. It uses pre-built images from GitHub Container Registry, so no build step is required. Paste this into **Stacks → Add stack** in Portainer.

```yaml
version: "3.8"
services:
  # ── UniFi Network MCP ──────────────────────────────────────────
  unifi-network-mcp:
    image: ghcr.io/sirkirby/unifi-network-mcp:latest
    container_name: unifi-network-mcp
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - UNIFI_MCP_HTTP_ENABLED=true
      - UNIFI_MCP_HTTP_TRANSPORT=streamable-http
      - UNIFI_MCP_HOST=0.0.0.0
      - UNIFI_MCP_PORT=3000
      - UNIFI_MCP_ALLOWED_HOSTS=localhost,localhost:3000,127.0.0.1,127.0.0.1:3000,host.docker.internal,host.docker.internal:3000,network-services,network-services:3000
      - UNIFI_NETWORK_HOST=192.168.0.105
      - UNIFI_NETWORK_PORT=11443
      - UNIFI_NETWORK_USERNAME=Unifi-MCP
      - UNIFI_NETWORK_PASSWORD=<your-password>
      - UNIFI_NETWORK_VERIFY_SSL=false

  # ── UniFi Protect MCP ──────────────────────────────────────────
  unifi-protect-mcp:
    image: ghcr.io/sirkirby/unifi-protect-mcp:latest
    container_name: unifi-protect-mcp
    restart: unless-stopped
    ports:
      - "3001:3001"
    environment:
      - UNIFI_MCP_HTTP_ENABLED=true
      - UNIFI_MCP_HTTP_TRANSPORT=streamable-http
      - UNIFI_MCP_HOST=0.0.0.0
      - UNIFI_MCP_PORT=3001
      - UNIFI_MCP_ALLOWED_HOSTS=localhost,localhost:3001,127.0.0.1,127.0.0.1:3001,host.docker.internal,host.docker.internal:3001,network-services,network-services:3001
      - UNIFI_PROTECT_HOST=192.168.0.105
      - UNIFI_PROTECT_PORT=11443
      - UNIFI_PROTECT_USERNAME=Unifi-MCP
      - UNIFI_PROTECT_PASSWORD=<your-password>
      - UNIFI_PROTECT_VERIFY_SSL=false

  # ── UniFi Access MCP ───────────────────────────────────────────
  unifi-access-mcp:
    image: ghcr.io/sirkirby/unifi-access-mcp:latest
    container_name: unifi-access-mcp
    restart: unless-stopped
    ports:
      - "3002:3002"
    environment:
      - UNIFI_MCP_HTTP_ENABLED=true
      - UNIFI_MCP_HTTP_TRANSPORT=streamable-http
      - UNIFI_MCP_HOST=0.0.0.0
      - UNIFI_MCP_PORT=3002
      - UNIFI_MCP_ALLOWED_HOSTS=localhost,localhost:3002,127.0.0.1,127.0.0.1:3002,host.docker.internal,host.docker.internal:3002,network-services,network-services:3002
      - UNIFI_ACCESS_HOST=192.168.0.105
      - UNIFI_ACCESS_PORT=11443
      - UNIFI_ACCESS_USERNAME=Unifi-MCP
      - UNIFI_ACCESS_PASSWORD=<your-password>
      - UNIFI_ACCESS_VERIFY_SSL=false
```

A few things worth calling out:

- `UNIFI_MCP_HTTP_ENABLED=true` and `UNIFI_MCP_HTTP_TRANSPORT=streamable-http` are what make these servers talk to Open WebUI. Without them, the containers default to stdio mode, which is fine for Claude Desktop but useless over a network [1].
- `UNIFI_NETWORK_PORT=11443` accounts for my controller's non-standard HTTPS port.
- `UNIFI_NETWORK_VERIFY_SSL=false` is necessary because UniFi OS uses a self-signed certificate that the container won't trust otherwise.
- `UNIFI_MCP_ALLOWED_HOSTS` includes `network-services` so hostname-based requests don't get rejected by FastMCP's host validation.

If you don't use Protect or Access, just delete those service blocks. They'll fail to start if the controller doesn't have those applications installed.

## Deployment and first smoke test

After deploying the stack in Portainer, check the logs:

```bash
docker logs unifi-network-mcp --tail=30
```

A healthy startup looks like this:

```text
Attempting to connect to Unifi controller at 192.168.0.105...
Pre-login auto-detected controller type: UniFi OS (proxy)
Detected UniFi OS controller (proxy paths required)
Successfully connected to Unifi controller at 192.168.0.105 for site 'default'
Starting FastMCP Streamable HTTP server on 0.0.0.0:3000 ...
Uvicorn running on http://0.0.0.0:3000 (Press CTRL+C to quit)
```

The auto-detection of the controller type is a nice touch — it correctly identified my UniFi OS console and adjusted its API paths accordingly.

## Connecting to Open WebUI

This is where I hit the first gotcha.

In Open WebUI, go to **Admin Settings → External Tools → Add Server** and configure:

| Field | Value |
|-------|-------|
| Type | `MCP (Streamable HTTP)` |
| URL | `http://unifi-network-mcp:3000/mcp` |
| Auth | `None` |

The critical detail is the **`/mcp` path**. FastMCP serves the Streamable HTTP endpoint at `/mcp`, not at the root `/`. If you point Open WebUI at `http://unifi-network-mcp:3000` (without the `/mcp` suffix), it hits `POST /`, which returns `404 Not Found`. Took me a few minutes of staring at container logs full of `404` responses to catch that one.

If Open WebUI is on a different Docker network than the MCP containers, the container name `unifi-network-mcp` won't resolve. In that case, use the Unraid host IP:

```
http://<unraid-ip>:3000/mcp
```

Add Protect and Access at `:3001/mcp` and `:3002/mcp` if you deployed them. No authentication is required — the MCP servers don't implement their own auth layer. Access control is handled entirely by Open WebUI's built-in server scoping.

## What you can ask it to do

Once connected, ask your AI agent in plain English. Here are a few prompts I've found useful:

**Network diagnostics:**

> "Show me all clients on the network with their signal strength and data usage"
> "Which access points have the most client disconnections this week?"
> "What changed on my network in the last 24 hours?"

**Firewall management:**

> "Create a firewall rule that blocks IoT devices from reaching the internet between midnight and 6 AM"
> "Audit my firewall policies — are there any redundant or conflicting rules?"

**Traffic analysis:**

> "Show me the largest traffic flows from the last hour and summarize who talked to what"

Every mutation comes back with a preview first. You see the exact payload before anything is written, and you confirm or deny it. That's the safety model that makes this usable in production — the AI proposes, you dispose.

## Security posture

- **Secret redaction is on by default.** Wi-Fi passphrases, VPN private keys, SNMP community strings — all come back as `***REDACTED***`. If I genuinely need raw values, I can set `UNIFI_REDACT_SENSITIVE_FIELDS=false`, but I haven't had a reason to yet [1].
- **The service account is local-only.** `Unifi-MCP` is not a Ubiquiti SSO account, so there's no cloud authentication path that could be compromised.
- **Network exposure is minimal.** The MCP containers bind to `0.0.0.0:3000`–`3002`, but those ports are only reachable on the internal Docker network and the Unraid host itself. Nothing is forwarded at the edge.
- **No MCP-level auth.** The servers don't require bearer tokens or API keys. That's fine in my setup because access is gated by Open WebUI's admin-only MCP server configuration, but if I were exposing these to a broader set of users, I'd want to add authentication or use the project's Cloud Relay component.

## Final thoughts

The project itself is solid. Auto-detection of UniFi OS versus the older controller software worked first try. Lazy tool loading keeps the initial MCP handshake small, which matters when you have 186 tools registered.

What I like most is that this isn't just a query interface. The agent skills that ship with the Claude Code plugin (the Network Health Check, Firewall Manager, Firewall Auditor) are genuinely useful, and I'd love to see something similar become available for Open WebUI users through the MCP tools directly.

For now, I have an AI that can answer "what's on my network right now?" without me opening a single browser tab.

## Related Resources

- [UniFi MCP on GitHub](https://github.com/sirkirby/unifi-mcp)
- [UniFi MCP website](https://unifimcp.com/)
- [Open WebUI MCP documentation](https://docs.openwebui.com/features/extensibility/mcp/)
- [Model Context Protocol specification](https://modelcontextprotocol.io/)
