---
title: Open WebUI oikb — Nextcloud and GitHub Knowledge Base Sync Setup
date: 2026-06-19 16:46:00
categories: [open-webui,documentation,sync]
tags: [openwebui,oikb,nextcloud,github,docker,docker-compose]
---

# Open WebUI oikb — Nextcloud and GitHub Knowledge Base Sync Setup

![syncing](https://images.pexels.com/photos/13328054/pexels-photo-13328054.jpeg)

## Overview

This article documents the setup to add **oikb** Knowledge Base Sync to an existing Open WebUI Docker Compose deployment.

`oikb` is the official companion tool for mirroring external content into Open WebUI Knowledge Bases. It can sync sources such as local folders, GitHub repositories, Confluence spaces, S3 buckets, Nextcloud folders, and many other supported sources into Open WebUI Knowledge Bases [1].

Unlike one-time file uploads, `oikb` performs incremental sync. It hashes files with SHA-256, compares them against the Knowledge Base state, uploads only new or modified files, and removes deleted/stale files [1].

## Environment Summary

Existing services:

- `openwebui`
- `mcpo`
- newly added `oikb`

Open WebUI is externally available at:

```text
https://ai.cjs-cloud.com
```

Inside Docker Compose, `oikb` communicates with Open WebUI using the service name:

```text
http://openwebui:8080
```

This is preferred inside the same Compose stack because containers can reach each other by service name on the Docker network.

---

# Requirements

`oikb` requires Open WebUI version **0.9.6+**, because it uses the incremental sync endpoints that were added in Open WebUI 0.9.6 [1].

Required credentials and IDs:

- Open WebUI API key
- Open WebUI Knowledge Base UUID
- Nextcloud base URL
- Nextcloud username
- Nextcloud app password
- Optional GitHub token for GitHub repo sync

The Knowledge Base ID is the UUID at the end of the Knowledge Base URL in Open WebUI Workspace:

```text
.../knowledge/<kb-id>
```

The Open WebUI docs state that the KB ID is the UUID in the Workspace Knowledge Base URL [1].

---

# Security Notice

During setup, several live credentials were used or exposed in plain text.

Rotate the following credentials after setup:

- Open WebUI API key
- Nextcloud app password
- OIKB API key
- OpenAI API key
- OAuth client secret
- Open WebUI secret key

`oikb` supports environment variable configuration, and `.oikb.yaml` supports variable interpolation such as `${VAR}` and `${VAR:-default}` [2].

---

# Final Working Setup

## Host Config Directory

The working config directory on the Docker host was:

```text
/opt/appdata/oikb
```

It contains:

```text
/opt/appdata/oikb/.env
/opt/appdata/oikb/.oikb.yaml
```

Verify the files:

```bash
ls -la /opt/appdata/oikb/
cat /opt/appdata/oikb/.oikb.yaml
```

Expected files:

```text
.env
.oikb.yaml
```

---

# `.oikb.yaml`

The working `.oikb.yaml` file:

```yaml
defaults:
  interval: 1h
  concurrency: 4
  filter:
    max-size: 50mb

sources:
  - name: nextcloud-documents
    source: nextcloud:/Documents
    kb-id: ${KB_NEXTCLOUD_ID}

  - name: nextcloud-onyx
    source: nextcloud:/onyx
    kb-id: ${KB_NEXTCLOUD_ID}
```

## What This Configuration Does

This syncs two Nextcloud folders:

```text
/Documents
/onyx
```

into this Open WebUI Knowledge Base:

```text
16c501cb-e11b-4fe5-af56-80adfdf890fe
```

The default schedule is:

```yaml
interval: 1h
```

So each source is scheduled to sync every hour.

`oikb daemon` reads `.oikb.yaml` and syncs each configured source on a schedule [2].

In case you're curious, Documents is my computer-written notes folder along with pdfs and such, the onyx folder is what I automatically upload my Boox handwritten notes to.

---

# `.env`

A `.env` file can be used to store secrets, although variables can also be placed directly in the Docker Compose `environment:` section.

Example `.env` format:

```env
OPEN_WEBUI_API_KEY=your-openwebui-api-key
OIKB_API_KEY=your-oikb-daemon-api-key
KB_NEXTCLOUD_ID=your-KB-id

NEXTCLOUD_URL=https://nextcloud.cjs-cloud.com
NEXTCLOUD_USER=username
NEXTCLOUD_PASSWORD=your-nextcloud-app-password
```

Lock down permissions:

```bash
chmod 600 /opt/appdata/oikb/.env
```

---

# Final Working Docker Compose Service

The key fix was adding:

```yaml
working_dir: /config
```

Without this, the container mounted `/opt/appdata/oikb` correctly, but `oikb` still could not find `.oikb.yaml` because it was not starting from the directory where the config file existed.

Final working `oikb` service:

```yaml
  oikb:
    image: ghcr.io/open-webui/oikb:latest
    container_name: oikb
    restart: unless-stopped
    environment:
      - TZ=America/New_York
      - OPEN_WEBUI_URL=http://openwebui:8080
      - OPEN_WEBUI_API_KEY=your-openwebui-api-key
      - OIKB_API_KEY=your-oikb-daemon-api-key
      - KB_NEXTCLOUD_ID=your_KB_id
      - NEXTCLOUD_URL=https://nextcloud.cjs-cloud.com
      - NEXTCLOUD_USER=csadmin
      - NEXTCLOUD_PASSWORD=your-nextcloud-app-password
      - LOG_FORMAT=json
    volumes:
      - /opt/appdata/oikb:/config:ro
    working_dir: /config
    command: daemon
    ports:
      - "8035:8080"
    depends_on:
      - openwebui
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 5s
```

## Why This Works

The successful manual Docker test was:

```bash
docker run --rm --entrypoint sh \
  -v /opt/appdata/oikb:/config:ro \
  -w /config \
  ghcr.io/open-webui/oikb:latest \
  -c "pwd; echo '---'; ls -la; echo '--- .oikb.yaml ---'; cat .oikb.yaml"
```

This proved that Docker could see the config file when:

- `/opt/appdata/oikb` was mounted to `/config`
- the container working directory was `/config`

The working Compose service mirrors that with:

```yaml
volumes:
  - /opt/appdata/oikb:/config:ro
working_dir: /config
command: daemon
```

---

# Deploy / Redeploy Commands

After editing the Compose file:

```bash
docker rm -f oikb
docker compose up -d oikb
docker logs oikb -f
```

If using Portainer, redeploy the stack from Portainer after editing the stack YAML.

## Verify Running Container Configuration

Check the running command and working directory:

```bash
docker inspect oikb --format 'WorkingDir={{.Config.WorkingDir}} Cmd={{json .Config.Cmd}}'
```

Expected:

```text
WorkingDir=/config Cmd=["daemon"]
```

Check mounts:

```bash
docker inspect oikb --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected:

```text
/opt/appdata/oikb -> /config
```

If the output does not match, the updated Compose stack was not applied or the wrong Compose file/Portainer stack was edited.

---

# Scheduling

The current `.oikb.yaml` contains:

```yaml
defaults:
  interval: 1h
```

This means all sources without an override are synced every hour.

`oikb` daemon supports scheduled sync using simple intervals such as `30m` or `1h`, and also cron expressions such as `0 6 * * 1-5` [2].

## Per-Source Schedule Override

Example:

```yaml
sources:
  - name: nextcloud-documents
    source: nextcloud:/Documents
    kb-id: ${KB_NEXTCLOUD_ID}
    interval: 30m

  - name: nextcloud-onyx
    source: nextcloud:/onyx
    kb-id: ${KB_NEXTCLOUD_ID}
    interval: 6h
```

This would sync:

- `/Documents` every 30 minutes
- `/onyx` every 6 hours

## Cron Example

```yaml
sources:
  - name: nextcloud-documents
    source: nextcloud:/Documents
    kb-id: ${KB_NEXTCLOUD_ID}
    interval: "0 3 * * *"
```

This would run the sync daily at 3:00 AM.

---

# Incremental Sync Behavior

`oikb` does not re-upload everything every time.

It works like this:

1. Scans the source.
2. Computes a SHA-256 checksum for every file.
3. Sends a manifest to Open WebUI `/sync/diff`.
4. Open WebUI returns what was added, modified, deleted, and unchanged.
5. `oikb` deletes stale files, creates missing directories, and uploads only new or modified files [1].

This means:

- First run uploads everything.
- Later runs only process changes.
- Unchanged files are not re-uploaded or re-embedded.
- Deleted source files are removed from the KB.

Example from the docs:

```text
Diff: +3, ~1, -2, 1198 unchanged
```

This means:

- 3 files added
- 1 file modified
- 2 files deleted
- 1198 files unchanged [1]

## Important Caution About Existing KB Files

Because `oikb` treats the configured source as the source of truth, files that exist in the Knowledge Base but do not exist in the configured sync source may be removed during sync.

This means manually uploaded KB files can disappear if the KB is later managed by `oikb`.

Recommended practice:

- Use a dedicated KB for `oikb`-managed content.
- Use a separate KB for manually uploaded files.
- Avoid mixing manually uploaded files and sync-managed files in the same KB unless you understand the deletion behavior.

---

# What Happened During the First Sync

The logs showed:

```text
Synced nextcloud:/Documents -> 16c501cb-e11b-4fe5-af56-80adfdf890fe: 33 added, 15 dirs created
```

This means:

- 33 files were uploaded.
- 15 directories were created.

A later run showed:

```text
Synced nextcloud:/Documents -> 16c501cb-e11b-4fe5-af56-80adfdf890fe: 33 unchanged
```

This means:

- The files were already synced.
- Nothing changed.
- No files needed to be uploaded again.

## Why `/onyx` Was Skipped Initially

The logs showed:

```text
Skipping nextcloud:/onyx — sync already running for 16c501cb-e11b-4fe5-af56-80adfdf890fe
```

This happened because both `/Documents` and `/onyx` were pointed at the same KB ID.

The daemon runs as one process, and scheduling/per-KB locks live in that single process. The docs state that the daemon is meant to run as one replica and that additional sources should be added to one daemon rather than running several copies [1].

If two sources target the same KB at the same time, one may be skipped until a later scheduled run.

## Recommendation

If `/Documents` and `/onyx` should remain separate, create a second Knowledge Base and assign `/onyx` to it.

Example:

```yaml
defaults:
  interval: 1h
  concurrency: 4
  filter:
    max-size: 50mb

sources:
  - name: nextcloud-documents
    source: nextcloud:/Documents
    kb-id: 16c501cb-e11b-4fe5-af56-80adfdf890fe

  - name: nextcloud-onyx
    source: nextcloud:/onyx
    kb-id: YOUR-SECOND-KB-ID-HERE
```

Using separate KBs avoids confusion and makes each source easier to manage.

---

# Adding GitHub Repository Sync

`oikb` supports GitHub repositories with the source format:

```text
github:owner/repo
```

The Open WebUI docs show that GitHub repos can be synced without a local clone [1].

## One-Shot GitHub Sync

Example:

```bash
oikb sync github:owner/repo --kb-id your-kb-id
```

## Add GitHub to `.oikb.yaml`

Create a new Knowledge Base in Open WebUI for the GitHub repo, then copy its KB UUID.

Add this to the Docker Compose environment:

```yaml
- GITHUB_TOKEN=your-github-token
- KB_GITHUB_ID=your-github-kb-id
```

Then update `.oikb.yaml`:

```yaml
defaults:
  interval: 1h
  concurrency: 4
  filter:
    max-size: 50mb

sources:
  - name: nextcloud-documents
    source: nextcloud:/Documents
    kb-id: ${KB_NEXTCLOUD_ID}

  - name: nextcloud-onyx
    source: nextcloud:/onyx
    kb-id: ${KB_NEXTCLOUD_ID}

  - name: github-repo
    source: github:owner/repo
    kb-id: ${KB_GITHUB_ID}
```

Replace:

```text
owner/repo
```

with the real GitHub owner and repository name.

## GitHub Token

For GitHub:

- Public repos may not require a token, but a token is recommended to avoid rate limits.
- Private repos require a GitHub token.
- The Open WebUI docs list `GITHUB_TOKEN` as the credential variable for private GitHub repositories [1].

## Optional GitHub Filters

To sync only documentation:

```yaml
  - name: github-repo
    source: github:owner/repo
    kb-id: ${KB_GITHUB_ID}
    filter:
      include: ["docs/**/*.md", "*.md", "*.txt"]
      exclude: ["drafts/**", "node_modules/**", "**/*.lock"]
      max-size: 50mb
```

`oikb` supports include/exclude globs and max file size filtering [1].

## Optional Split GitHub Repo Into Multiple KBs

Example:

```yaml
sources:
  - name: wiki-docs
    source: github:owner/repo
    kb-id: abc123
    filter:
      include: ["docs/**/*.md"]

  - name: wiki-code
    source: github:owner/repo
    kb-id: def456
    filter:
      include: ["src/**"]
```

The official docs show using filters to route different parts of one source into different Knowledge Bases [1].

---

# Useful Commands

## Validate Config

```bash
docker exec -it oikb oikb validate
```

Deep validation:

```bash
docker exec -it oikb oikb validate --deep
```

`validate --deep` verifies that Open WebUI is reachable, the API key is valid, and every configured KB ID exists [1].

## Sync One Named Entry

```bash
docker exec -it oikb oikb sync --name nextcloud-documents
```

```bash
docker exec -it oikb oikb sync --name nextcloud-onyx
```

## Dry Run

Preview changes without uploading:

```bash
docker exec -it oikb oikb sync --name nextcloud-documents --dry-run
```

The docs describe `--dry-run` as a way to preview added, modified, and deleted files without uploading [1].

## View History

```bash
docker exec -it oikb oikb history
```

JSON output:

```bash
docker exec -it oikb oikb history --json
```

Failed syncs only:

```bash
docker exec -it oikb oikb history --errors
```

## List Files in a KB

```bash
docker exec -it oikb oikb ls --kb-id 16c501cb-e11b-4fe5-af56-80adfdf890fe
```

## Show KB Status

```bash
docker exec -it oikb oikb status --kb-id 16c501cb-e11b-4fe5-af56-80adfdf890fe
```

## Reset a KB

Use extreme caution:

```bash
docker exec -it oikb oikb reset --kb-id 16c501cb-e11b-4fe5-af56-80adfdf890fe
```

`oikb reset` deletes all files in the target Knowledge Base, not just files uploaded by `oikb` [1].

---

# oikb Daemon HTTP API

The daemon listens on port `8080` inside the container and was mapped to:

```text
8035:8080
```

The logs showed endpoints such as:

```text
GET  http://localhost:8080/health
GET  http://localhost:8080/metrics
GET  http://localhost:8080/history
POST http://localhost:8080/sync/{kb_id}
OpenAPI spec: http://localhost:8080/openapi.json
```

The GitHub README documents daemon features including health checks, Prometheus metrics, sync history, on-demand sync, API key auth, and OpenAPI tool server support [2].

From the Docker host, these are accessible through:

```text
http://localhost:8035
```

Example health check:

```bash
curl http://localhost:8035/health
```

Example metrics:

```bash
curl http://localhost:8035/metrics
```

---

# Troubleshooting Notes From Setup

## Problem: `No sync entries found. Create a .oikb.yaml file.`

Cause:

`oikb` could not find `.oikb.yaml` in its current working directory.

Fix:

Mount the config folder and set the working directory:

```yaml
volumes:
  - /opt/appdata/oikb:/config:ro
working_dir: /config
command: daemon
```

The docs note that this error happens when `oikb sync` is run with no source and there is no `.oikb.yaml` in the current directory [1].

## Problem: `env file /opt/appdata/oikb/.env not found`

Cause:

Docker Compose or Portainer could not find the `.env` file at the host path referenced by `env_file`.

Fix options:

1. Create the file:

```bash
mkdir -p /opt/appdata/oikb
nano /opt/appdata/oikb/.env
chmod 600 /opt/appdata/oikb/.env
```

2. Or remove `env_file:` and place variables directly in the Compose `environment:` block.

## Problem: `docker compose run ...` returned `no configuration file provided: not found`

Cause:

The command was run from a directory that did not contain a Compose file.

Fix:

Run Docker Compose from the directory containing the compose file, or specify it directly:

```bash
docker compose -f /path/to/docker-compose.yml run --rm ...
```

## Problem: Host files exist but container cannot find them

Diagnostic test:

```bash
docker run --rm --entrypoint sh \
  -v /opt/appdata/oikb:/config:ro \
  -w /config \
  ghcr.io/open-webui/oikb:latest \
  -c "pwd; echo '---'; ls -la; echo '--- .oikb.yaml ---'; cat .oikb.yaml"
```

If this works, Docker can see the files. The issue is the Compose service definition.

Final fix:

```yaml
volumes:
  - /opt/appdata/oikb:/config:ro
working_dir: /config
```

---

# Best Practices Going Forward

## Use Dedicated KBs for Sync Sources

Recommended layout:

- KB: `Nextcloud Documents`
- KB: `Nextcloud Onyx`
- KB: `GitHub Repo`
- KB: `Manual Uploads`

This avoids accidental cleanup of manually uploaded files and reduces confusion about which source owns which KB.

## Keep Secrets Out of Markdown and Git

Use placeholder values in documentation:

```text
your-openwebui-api-key
your-nextcloud-app-password
your-github-token
```

## Use Filters for Large or Noisy Sources

Example:

```yaml
filter:
  include: ["**/*.md", "**/*.txt", "**/*.pdf", "**/*.docx"]
  exclude: ["node_modules/**", ".git/**", "**/*.tmp"]
  max-size: 50mb
```

Large syncs can be improved by using filters, max-size limits, and concurrency [1].

## Remember Server-Side Indexing Delay

`oikb` uploads files quickly, but Open WebUI extracts and embeds each new file asynchronously. A just-synced file is in the Knowledge Base, but it may take a moment before it is queryable [1].

---

# Final Notes

The working configuration was successful once the container was started with:

```yaml
working_dir: /config
```

and the config directory was mounted with:

```yaml
volumes:
  - /opt/appdata/oikb:/config:ro
```

The Nextcloud `/Documents` source synced successfully and later reported unchanged files, confirming incremental sync was working.

GitHub sync can be added by adding a `github:owner/repo` source entry, a GitHub token if needed, and a target Knowledge Base ID.

---

# References

- Open WebUI Knowledge Base Sync documentation [1]
- open-webui/oikb GitHub repository [2]
