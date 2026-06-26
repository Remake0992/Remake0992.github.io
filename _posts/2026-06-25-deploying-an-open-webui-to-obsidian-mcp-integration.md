---
title: Deploying an Open WebUI to Obsidian MCP Integration
date: 2026-6-25 21:01:00
categories:
  - knowledge-base
  - automation
tags:
  - openwebui
  - obsidian
  - mcp
  - tailscale
  - markdown
  - automation
---
![AI Notebook](https://images.pexels.com/photos/5614124/pexels-photo-5614124.jpeg)
## The Pen V2

As I've been toying with AI in my personal life, I realized that I was doing a lot of copy and pasting-- way too much for my own good. For weeks, I've been plagued with this thought: "gosh, it'd be so easy if AI could write directly to my notebook!" At the time it was not possible. I was primarily using Nextcloud for notes. I know... I'm not sure why either. Nextcloud just makes organizing all of my documents in one place stupidly simple. I can chuck a markdown file, a docx file, and a spreadsheet in the same folder without thinking twice. 

This worked for a while until my needs developed. I needed something with proper tag support; I needed something with keyword searching and a fast, responsive interface; I needed something that could work offline in a pinch. I needed an actual notebook. Enter, Obsidian! I've been using Obsidian on and off for about five years now. It's a great piece of software, don't get me wrong, but every time I've dropped it, it was always because of how manual everything is. To build a proper "second-brain" style vault, you have to do a lot of work keeping up with formatting, tagging, and titling to ensure that all of your notes are cohesive. Don't even get me started on the nightmare of folder management. 

All of these issues are solved (or complicated) by AI, however. I can have AI format my notes for me. I can bang out a quick note on my phone, write something down on my e-ink tablet, and take some AI research chat history and combine them all into one document using a single prompt. Better yet, I can tell AI to put all of that together for me and write it into today's daily note. No more notes scattered in a million different places! 

This guide documents how to integrate Open WebUI with an Obsidian vault so AI can read from and write to notes through the Obsidian Local REST API with MCP plugin.
The deployment uses:
- Open WebUI running on a separate Linux host
- Obsidian running on a MacBook Pro
- Tailscale for private network connectivity
- The Obsidian Local REST API with MCP plugin for authenticated vault access
- Open WebUI External Tools configured as an MCP Streamable HTTP connection
This procedure was validated with a working remote connection from the Open WebUI host to the Obsidian host over Tailscale.

---
## Prerequisites

Before starting, confirm the following:
- Open WebUI is already deployed and reachable
- Open WebUI is version 0.6.31 or later
- Obsidian is installed and running on a desktop system
- The Local REST API with MCP plugin is installed and enabled in Obsidian
- Both systems are connected to the same Tailscale tailnet
- An Obsidian API key has been generated from the plugin settings
- You have administrative access to Open WebUI External Tools
Environment used in this deployment:
- Open WebUI host: `core-services`
- Obsidian host: `collens-macbook-pro`
- Obsidian Tailscale IP: `100.93.113.41`
- Obsidian HTTP API port: `27123`
- Open WebUI data path: `/appdata/openwebui`

> **Important:** This guide uses the plugin's HTTP listener on port `27123` over Tailscale to avoid dealing with self-signed HTTPS trust during initial setup. Not recommended for production. 

---
## Procedure
### Step 1: Install and enable the Obsidian plugin
Open Obsidian and install the **Local REST API with MCP** community plugin.
After installation:
1. Enable the plugin.
2. Open the plugin settings.
3. Copy the generated API key.
4. Enable the HTTP server.
5. Enable any advanced settings required to expose the bind/listen host option.
Expected result:
- Obsidian exposes a local API.
- An API key is available for Bearer authentication.
### Step 2: Configure the plugin to listen on a reachable interface
The initial issue encountered during deployment was that the API worked locally on the Mac but failed remotely from the Open WebUI host.
To fix this:
1. In the plugin settings, change the bind/listen host from `127.0.0.1` or `localhost` to either:
   - `0.0.0.0`, or
   - the Mac's Tailscale IP
2. Restart Obsidian completely.
Expected result:
- The API no longer listens only on loopback.
- Remote clients on the tailnet can connect.
### Step 3: Verify the API locally on the Obsidian host
Run the following commands on the Mac:
```bash
tailscale ip -4
lsof -nP -iTCP:27123 -sTCP:LISTEN
curl http://127.0.0.1:27123/
curl -H "Authorization: Bearer $OBSIDIAN_API_KEY" \
  http://127.0.0.1:27123/vault/
```
Expected result:
- Port `27123` is listening.
- The root endpoint returns plugin status.
- The authenticated `/vault/` request returns vault contents.
### Step 4: Verify remote connectivity from the Open WebUI host
On the Open WebUI host, export the API key and remote endpoint:
```bash
export OBSIDIAN_API_KEY='REPLACE_WITH_OBSIDIAN_API_KEY'
export OBSIDIAN_REMOTE='http://100.93.113.41:27123'
```
Test connectivity:
```bash
tailscale status
tailscale ping 100.93.113.41
ping -c 3 100.93.113.41
nc -vz 100.93.113.41 27123
curl http://100.93.113.41:27123/
curl -H "Authorization: Bearer $OBSIDIAN_API_KEY" \
  http://100.93.113.41:27123/vault/
```
Expected result:
- `tailscale ping` succeeds
- standard ICMP ping succeeds
- `nc` reports port `27123` as open
- the root endpoint returns API status
- the authenticated `/vault/` request returns folder and file listings
### Step 5: Confirm the working remote API response
A successful remote validation returned:
- plugin status `OK`
- plugin name `Local REST API with MCP`
- plugin version `4.1.3`
- Obsidian version `1.12.7`
- authenticated vault listing including:
  - `Inbox/`
  - `Recipe Book/`
  - `Tasks.md`
  - `cjs-cloud.com/`
This confirms that:
- the Mac is reachable over Tailscale
- the Obsidian API is listening on a remote-accessible interface
- the Bearer token is valid
- the vault can be queried remotely from the Open WebUI host
### Step 6: Prepare Open WebUI for MCP
Ensure the Open WebUI container includes a `WEBUI_SECRET_KEY` environment variable.
Example Docker Compose fragment:
```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    environment:
      - WEBUI_SECRET_KEY=replace-with-a-long-random-secret
    volumes:
      - /appdata/openwebui:/app/backend/data
    ports:
      - "9766:8080"
```
Redeploy and review logs:
```bash
docker compose up -d
docker logs --tail=100 openwebui
```
Expected result:
- Open WebUI starts normally with MCP support available.
### Step 7: Add the Obsidian MCP server in Open WebUI
In Open WebUI:
1. Go to **Admin Settings**
2. Open **External Tools**
3. Click **Add Server**
4. Configure the connection with these values:
| Field | Value |
|---|---|
| Type | `MCP (Streamable HTTP)` |
| URL | `http://100.93.113.41:27123/mcp/` |
| Auth | `Bearer` |
| Key | Obsidian API key |
Save the server and run the built-in verification.
Expected result:
- The server saves successfully.
- Open WebUI verifies the connection.
> **Important:** Do not add this integration as OpenAPI. Use `MCP (Streamable HTTP)`.
### Step 8: Enable the tool in chat
After the server is added:
1. Open a chat in Open WebUI.
2. Open **Integrations** or **Tools**.
3. Enable the Obsidian tool for the chat or model.
Recommended first test prompt:
```text
Create a note at Inbox/AI/openwebui-first-note.md with a title, a short summary of this chat, and three action items.
```
Recommended append test:
```text
Append a section called AI Notes to Inbox/AI/openwebui-first-note.md and add two bullet points.
```
Expected result:
- The model writes directly into the Obsidian vault using structured tool calls.

---
## Validation

Use the following checks to confirm the deployment is complete:
1. From the Open WebUI host, `nc -vz 100.93.113.41:27123` returns `open`.
2. `curl http://100.93.113.41:27123/` returns plugin status.
3. `curl -H "Authorization: Bearer $OBSIDIAN_API_KEY" http://100.93.113.41:27123/vault/` returns vault contents.
4. Open WebUI verifies the MCP server successfully.
5. A chat session can create or append a note in `Inbox/AI/`.

--- 
## Troubleshooting
### Symptom: Connection refused from the Open WebUI host
Common cause:
- The Obsidian plugin is listening only on `127.0.0.1`
Resolution:
1. Change the bind/listen host to `0.0.0.0` or the Tailscale IP.
2. Restart Obsidian.
3. Retest with `nc` and `curl`.
### Symptom: Local curl works on the Mac but remote curl fails
Common causes:
- Plugin bound only to loopback
- macOS firewall blocking inbound access
- Wrong target IP or port
Resolution:
1. Confirm the Tailscale IP with `tailscale ip -4`
2. Confirm the listener with:
```bash
lsof -nP -iTCP:27123 -sTCP:LISTEN
```
3. Confirm the firewall allows incoming connections to Obsidian.
4. Retest from the Open WebUI host.
### Symptom: Open WebUI verify fails
Common causes:
- Wrong connection type
- Wrong path
- Missing or invalid Bearer token
Resolution:
- Confirm:
  - Type is `MCP (Streamable HTTP)`
  - URL is `http://100.93.113.41:27123/mcp/`
  - Auth is `Bearer`
  - The API key is correct
### Symptom: Tool is added but does not act on notes
Common causes:
- Tool not enabled in the chat
- Model not allowed to use the tool
- Prompt does not clearly specify a file path or action
Resolution:
1. Enable the tool in the active chat.
2. Use explicit prompts with a full vault path.
3. Start with create or append actions in a dedicated test folder.

--- 
## Security Considerations
- Use Tailscale or another private network path for API access.
- Do not expose the Obsidian API directly to the public internet.
- Restrict initial AI write operations to a controlled folder such as `Inbox/AI/`.
- Rotate the Obsidian API key after setup if it has been exposed in terminal history, screenshots, or documentation drafts.
- Expand write scope only after validating model behavior.

> **Warning:** The Obsidian plugin can provide broad vault access. Avoid giving unrestricted write access to sensitive notes during initial rollout.

--- 
## Related Resources

- [Open WebUI MCP documentation](https://docs.openwebui.com/features/extensibility/mcp/)
- [Obsidian Local REST API with MCP plugin documentation](https://community.obsidian.md/plugins/obsidian-local-rest-api)
- Internal Open WebUI deployment documentation
- Internal AI knowledge and notebook automation standards by AI