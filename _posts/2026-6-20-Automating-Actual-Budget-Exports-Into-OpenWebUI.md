---
title: Automating Actual Budget Exports into Open WebUI
date: 2026-06-20 17:45:00
categories: [open-webui,documentation,sync]
tags: [openwebui,oikb,nextcloud,actualbudget,docker,docker-compose,webdav,finance]
---

![budgeting](https://images.pexels.com/photos/4386373/pexels-photo-4386373.jpeg)

## Giving Open WebUI access to my budget

I wanted Open WebUI to be able to answer questions about my finances without manually exporting files from Actual Budget every time. The goal was to create a clean pipeline that automatically exports financial data from Actual Budget, places it somewhere durable, syncs it into an Open WebUI Knowledge Base, and keeps everything updated on a schedule.

The final setup uses:

- Actual Budget as the source of financial data
- A small Dockerized exporter container
- A Nextcloud WebDAV mount as the handoff location
- `oikb` to sync the exported files into Open WebUI
- A dedicated Open WebUI Knowledge Base for the Actual Budget exports

The end result is a repeatable pipeline:

```text
Actual Budget
   |
   | Actual Budget API
   v
actual-exporter container
   |
   | CSV / JSON / TXT exports
   v
Nextcloud WebDAV mount
   |
   | Local folder on Docker host
   v
oikb
   |
   | Incremental Knowledge Base sync
   v
Open WebUI
```

---

# Environment Summary

Actual Budget is running in Docker on the Debian host.

The running Actual Budget container is:

```text
actualbudget-actualbudget-1
```

The image is:

```text
actualbudget/actual-server:latest
```

Actual Budget is exposed on the host at:

```text
0.0.0.0:5006->5006/tcp
```

The public Actual Budget URL is:

```text
https://budget.cjs-cloud.com
```

The Nextcloud WebDAV folder used for exports is:

```text
ActualBudget Exports
```

The local mounted path on the Debian host is:

```text
/mnt/nextcloud-actualbudget-exports
```

The Open WebUI Knowledge Base UUID for the Actual Budget exports is:

```text
84256f82-f55d-4c82-9357-ddfc0eae873e
```

---

# Final Working Setup

## Nextcloud Export Folder

The Nextcloud folder is:

```text
ActualBudget Exports
```

The WebDAV base URL for the Nextcloud user is:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/files/csadmin
```

The full WebDAV path for the export folder is:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/files/csadmin/ActualBudget%20Exports/
```

On the Debian host, this folder is mounted at:

```text
/mnt/nextcloud-actualbudget-exports
```

The mount can be verified with:

```bash
mount | grep nextcloud-actualbudget-exports
```

Expected output should show something similar to:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/files/csadmin/ActualBudget Exports/ on /mnt/nextcloud-actualbudget-exports type fuse
```

---

# WebDAV Mount Configuration

## Required Package

The host uses `davfs2` for the WebDAV mount:

```bash
sudo apt install davfs2
```

## Mount Point

The local mount point is:

```bash
/mnt/nextcloud-actualbudget-exports
```

It was created with:

```bash
sudo mkdir -p /mnt/nextcloud-actualbudget-exports
```

---

# `/etc/fstab`

The working `/etc/fstab` entry is:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/files/csadmin/ActualBudget\040Exports/ /mnt/nextcloud-actualbudget-exports davfs rw,_netdev,uid=1000,gid=1000,file_mode=0660,dir_mode=0770 0 0
```

The important part is that the space in the folder name is represented as:

```text
\040
```

So this:

```text
ActualBudget Exports
```

becomes this in `/etc/fstab`:

```text
ActualBudget\040Exports
```

---

# `/etc/davfs2/secrets`

The WebDAV credentials are stored in:

```text
/etc/davfs2/secrets
```

The working format uses the mount point instead of the full URL:

```text
/mnt/nextcloud-actualbudget-exports csadmin "your-nextcloud-app-password"
```

The file is locked down with:

```bash
sudo chown root:root /etc/davfs2/secrets
sudo chmod 600 /etc/davfs2/secrets
```

---

# `/etc/davfs2/davfs2.conf`

WebDAV locks are disabled with:

```text
use_locks 0
```

This avoids lock-related warnings when mounting the Nextcloud WebDAV folder.

---

# Mount Commands

Reload systemd after editing `/etc/fstab`:

```bash
sudo systemctl daemon-reload
```

Mount the folder:

```bash
sudo mount /mnt/nextcloud-actualbudget-exports
```

Unmount the folder:

```bash
sudo umount /mnt/nextcloud-actualbudget-exports
```

Mount all configured filesystems:

```bash
sudo mount -a
```

---

# Actual Budget Exporter

## Host Config Directory

The exporter lives on the Docker host at:

```text
/opt/actual-exporter
```

The directory contains:

```text
/opt/actual-exporter/.env
/opt/actual-exporter/Dockerfile
/opt/actual-exporter/docker-compose.yml
/opt/actual-exporter/package.json
/opt/actual-exporter/export.js
```

Verify the files:

```bash
ls -la /opt/actual-exporter
```

Expected files:

```text
.env
Dockerfile
docker-compose.yml
package.json
export.js
```

---

# `package.json`

The exporter uses the official Actual Budget API package:

```json
{
  "name": "actual-budget-openwebui-exporter",
  "version": "1.0.0",
  "private": true,
  "type": "commonjs",
  "dependencies": {
    "@actual-app/api": "latest"
  }
}
```

---

# `Dockerfile`

The final working Dockerfile is:

```dockerfile
FROM node:20-bookworm-slim

WORKDIR /app

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    python3 \
    make \
    g++ \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

COPY package.json ./
RUN npm install --omit=dev

COPY export.js ./

CMD ["node", "/app/export.js"]
```

The build tools are included because the Actual Budget API dependency tree can require native Node modules to compile during install.

---

# `docker-compose.yml`

The exporter runs as its own small Docker Compose project.

The working Compose file is:

```yaml
services:
  actual-exporter:
    build: .
    env_file:
      - .env
    volumes:
      - actual_exporter_cache:/cache
      - /mnt/nextcloud-actualbudget-exports:/out:rw
    restart: "no"
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  actual_exporter_cache:
```

## What This Configuration Does

The exporter container writes output to:

```text
/out
```

That path is mapped to the host folder:

```text
/mnt/nextcloud-actualbudget-exports
```

Because that host folder is the Nextcloud WebDAV mount, any files written by the exporter appear in Nextcloud under:

```text
ActualBudget Exports
```

The exporter also uses a persistent cache volume:

```text
actual_exporter_cache
```

That cache is mounted inside the container at:

```text
/cache
```

---

# `.env`

The exporter environment file is:

```text
/opt/actual-exporter/.env
```

The working format is:

```env
ACTUAL_SERVER_URL=https://budget.cjs-cloud.com
ACTUAL_PASSWORD=your-actual-budget-server-password
ACTUAL_BUDGET_ID=80b43e52-baa2-47bf-b026-7253b7868d74
LOOKBACK_MONTHS=36
EXPORT_DIR=/out
ACTUAL_DATA_DIR=/cache
```

If the Actual Budget file itself is encrypted separately, this can also be added:

```env
ACTUAL_BUDGET_ENCRYPTION_PASSWORD=your-budget-file-encryption-password
```

## Important Variables

### `ACTUAL_SERVER_URL`

The Actual Budget server URL is:

```env
ACTUAL_SERVER_URL=https://budget.cjs-cloud.com
```

### `ACTUAL_PASSWORD`

This is the Actual Budget server password.

It is not:

- the Nextcloud password
- the Debian user password
- the OIDC password
- the Cloudflare password

### `ACTUAL_BUDGET_ID`

The Actual Budget ID is:

```env
ACTUAL_BUDGET_ID=80b43e52-baa2-47bf-b026-7253b7868d74
```

### `LOOKBACK_MONTHS`

The exporter currently exports the last 36 months:

```env
LOOKBACK_MONTHS=36
```

### `EXPORT_DIR`

The exporter writes to:

```env
EXPORT_DIR=/out
```

Inside the container, `/out` maps to:

```text
/mnt/nextcloud-actualbudget-exports
```

### `ACTUAL_DATA_DIR`

The Actual API cache lives at:

```env
ACTUAL_DATA_DIR=/cache
```

Inside Docker, `/cache` maps to the named volume:

```text
actual_exporter_cache
```

---

# Exported Files

A successful exporter run writes these files:

```text
actual-transactions.csv
actual-monthly-summary.csv
actual-accounts.csv
actual-categories.csv
actual-payees.csv
actual-export-metadata.json
README-for-openwebui.txt
```

These files are written to:

```text
/mnt/nextcloud-actualbudget-exports
```

And appear in Nextcloud under:

```text
ActualBudget Exports
```

## `actual-transactions.csv`

This contains individual transaction-level records.

It is intended for questions about:

- individual transactions
- merchants
- payees
- notes
- dates
- accounts
- transaction lookup

## `actual-monthly-summary.csv`

This contains deterministic monthly totals by category and account.

It is intended for questions about:

- monthly totals
- category totals
- income
- expenses
- spending trends
- account-level summaries

## `actual-accounts.csv`

This contains Actual Budget account metadata.

## `actual-categories.csv`

This contains Actual Budget category metadata.

## `actual-payees.csv`

This contains payee metadata if available from the Actual Budget API.

## `actual-export-metadata.json`

This contains export metadata, including:

- generated timestamp
- budget ID
- start date
- end date
- lookback window
- exported transaction count
- account count
- category count
- payee count

## `README-for-openwebui.txt`

This gives Open WebUI guidance on which files to use for different types of financial questions.

---

# Exporter Commands

Build the exporter:

```bash
cd /opt/actual-exporter
docker compose build
```

Run the exporter manually:

```bash
cd /opt/actual-exporter
docker compose run --rm actual-exporter
```

A successful run ends with output similar to:

```text
Export complete.
Transactions exported: 410
Files written to: /out
```

Verify the exported files:

```bash
ls -lah /mnt/nextcloud-actualbudget-exports
```

Preview transactions:

```bash
head -n 5 /mnt/nextcloud-actualbudget-exports/actual-transactions.csv
```

Preview monthly summaries:

```bash
head -n 5 /mnt/nextcloud-actualbudget-exports/actual-monthly-summary.csv
```

View export metadata:

```bash
cat /mnt/nextcloud-actualbudget-exports/actual-export-metadata.json
```

---

# oikb Knowledge Base Sync

`oikb` is already deployed as part of the Open WebUI stack.

The existing oikb config directory is:

```text
/opt/appdata/oikb
```

The important files are:

```text
/opt/appdata/oikb/.env
/opt/appdata/oikb/.oikb.yaml
```

`oikb` mirrors external content into Open WebUI Knowledge Bases and performs incremental sync instead of one-time uploads. It hashes files, compares them against the Knowledge Base state, uploads new or modified files, and removes deleted or stale files.

---

# Updated `.oikb.yaml`

The Actual Budget export source was added to `.oikb.yaml`.

The source entry is:

```yaml
  - name: actualbudget-exports
    source: /sources/actualbudget-exports
    kb-id: 84256f82-f55d-4c82-9357-ddfc0eae873e
    filter:
      include:
        - "*.csv"
        - "*.json"
        - "*.txt"
      exclude:
        - "*.tmp"
```

The full `.oikb.yaml` should keep the existing sources and include the new Actual Budget source.

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
    kb-id: ${KB_NEXTCLOUD_ID}

  - name: nextcloud-onyx
    source: nextcloud:/onyx
    kb-id: ${KB_NEXTCLOUD_ID}

  - name: actualbudget-exports
    source: /sources/actualbudget-exports
    kb-id: 84256f82-f55d-4c82-9357-ddfc0eae873e
    filter:
      include:
        - "*.csv"
        - "*.json"
        - "*.txt"
      exclude:
        - "*.tmp"
```

## Why the Source Path Is Different

On the Docker host, the export folder is:

```text
/mnt/nextcloud-actualbudget-exports
```

Inside the `oikb` container, it is mounted as:

```text
/sources/actualbudget-exports
```

So `.oikb.yaml` must use the container-visible path:

```yaml
source: /sources/actualbudget-exports
```

not the host path:

```yaml
source: /mnt/nextcloud-actualbudget-exports
```

---

# Updated oikb Docker Compose Service

The oikb container needs access to the mounted Nextcloud export folder.

The existing oikb service already mounts the config directory:

```yaml
- /opt/appdata/oikb:/config:ro
```

The Actual Budget export folder was added as a read-only bind mount:

```yaml
- /mnt/nextcloud-actualbudget-exports:/sources/actualbudget-exports:ro
```

The final oikb service format is:

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
      - KB_NEXTCLOUD_ID=your-nextcloud-kb-id
      - NEXTCLOUD_URL=https://nextcloud.cjs-cloud.com
      - NEXTCLOUD_USER=csadmin
      - NEXTCLOUD_PASSWORD=your-nextcloud-app-password
      - LOG_FORMAT=json
    volumes:
      - /opt/appdata/oikb:/config:ro
      - /mnt/nextcloud-actualbudget-exports:/sources/actualbudget-exports:ro
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

The oikb config directory is mounted into the container here:

```text
/config
```

The container starts in that directory:

```yaml
working_dir: /config
```

The daemon reads:

```text
/config/.oikb.yaml
```

The Actual Budget export folder is mounted read-only into the container here:

```text
/sources/actualbudget-exports
```

The `.oikb.yaml` source points to that same path:

```yaml
source: /sources/actualbudget-exports
```

This lets oikb sync the exported Actual Budget CSV, JSON, and TXT files into the dedicated Open WebUI Knowledge Base.

---

# Deploy / Redeploy Commands

After editing the oikb Docker Compose service:

```bash
docker rm -f oikb
docker compose up -d oikb
docker logs oikb -f
```

If using Portainer, redeploy the stack from Portainer after editing the stack YAML.

---

# Verify oikb Can See the Export Files

Check the running container mounts:

```bash
docker inspect oikb --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected mount entries should include:

```text
/opt/appdata/oikb -> /config
/mnt/nextcloud-actualbudget-exports -> /sources/actualbudget-exports
```

Check the files from inside the oikb container:

```bash
docker exec -it oikb ls -lah /sources/actualbudget-exports
```

Expected files:

```text
actual-transactions.csv
actual-monthly-summary.csv
actual-accounts.csv
actual-categories.csv
actual-payees.csv
actual-export-metadata.json
README-for-openwebui.txt
```

Validate the oikb config:

```bash
docker exec -it oikb oikb validate
```

Deep validation:

```bash
docker exec -it oikb oikb validate --deep
```

Run a dry run for the Actual Budget source:

```bash
docker exec -it oikb oikb sync --name actualbudget-exports --dry-run
```

Run a manual sync:

```bash
docker exec -it oikb oikb sync --name actualbudget-exports
```

List files in the Actual Budget Knowledge Base:

```bash
docker exec -it oikb oikb ls --kb-id 84256f82-f55d-4c82-9357-ddfc0eae873e
```

Check Knowledge Base status:

```bash
docker exec -it oikb oikb status --kb-id 84256f82-f55d-4c82-9357-ddfc0eae873e
```

---

# Scheduling

There are two scheduled components:

1. Actual Budget export
2. oikb sync into Open WebUI

## Actual Budget Export Schedule

The exporter can be scheduled with root cron:

```bash
sudo crontab -e
```

Daily export at 3:17 AM:

```cron
17 3 * * * mountpoint -q /mnt/nextcloud-actualbudget-exports && cd /opt/actual-exporter && /usr/bin/flock -n /tmp/actual-exporter.lock /usr/bin/docker compose run --rm actual-exporter >> /var/log/actual-exporter.log 2>&1
```

This does three things:

- confirms the Nextcloud WebDAV mount exists
- runs the exporter from `/opt/actual-exporter`
- prevents overlapping exporter runs with `flock`

Exporter logs are written to:

```text
/var/log/actual-exporter.log
```

View logs:

```bash
sudo tail -f /var/log/actual-exporter.log
```

## oikb Sync Schedule

The oikb daemon uses the interval in `.oikb.yaml`.

The default interval is:

```yaml
defaults:
  interval: 1h
```

This means all configured sources without a per-source override sync every hour.

A per-source override can be added to the Actual Budget source if desired:

```yaml
  - name: actualbudget-exports
    source: /sources/actualbudget-exports
    kb-id: 84256f82-f55d-4c82-9357-ddfc0eae873e
    interval: 1h
    filter:
      include:
        - "*.csv"
        - "*.json"
        - "*.txt"
      exclude:
        - "*.tmp"
```

Because the exporter currently runs daily at 3:17 AM, the hourly oikb daemon schedule is sufficient. After the exporter updates the files, oikb will pick up the modified files on its next scheduled sync.

---

# Open WebUI Knowledge Base

A dedicated Open WebUI Knowledge Base is used for the Actual Budget exports.

The Knowledge Base UUID is:

```text
84256f82-f55d-4c82-9357-ddfc0eae873e
```

The synced files are:

```text
actual-transactions.csv
actual-monthly-summary.csv
actual-accounts.csv
actual-categories.csv
actual-payees.csv
actual-export-metadata.json
README-for-openwebui.txt
```

This Knowledge Base should be attached to models or chats that need financial context.

---

# Recommended Open WebUI Instructions

Use these instructions for the model or Knowledge Base guidance:

```text
You have access to Actual Budget exports.

Use actual-monthly-summary.csv for totals, category spending, income, expenses, monthly summaries, trends, and comparisons.

Use actual-transactions.csv for individual transaction details, merchant/payee questions, account-specific questions, dates, notes, and transaction lookups.

Use actual-export-metadata.json to determine the available date range.

Amounts are decimal currency values. Negative transaction amounts are expenses or outflows. Positive transaction amounts are income, refunds, or inflows.

If the requested date range is outside the available export range, say that the data does not fully cover the requested period.
```

---

# Final Architecture

The final architecture looks like this:

```text
Actual Budget Docker container
   |
   | API access using @actual-app/api
   v
actual-exporter Docker Compose project
   |
   | Writes CSV, JSON, and TXT files
   v
/mnt/nextcloud-actualbudget-exports
   |
   | WebDAV-mounted Nextcloud folder
   v
Nextcloud / ActualBudget Exports
   |
   | Mounted read-only into oikb container
   v
/sources/actualbudget-exports
   |
   | oikb incremental sync
   v
Open WebUI Knowledge Base
   |
   | Attached to chat/model
   v
Financial questions in Open WebUI
```

---

# Useful Commands

## Actual Exporter

Run exporter manually:

```bash
cd /opt/actual-exporter
docker compose run --rm actual-exporter
```

Build exporter:

```bash
cd /opt/actual-exporter
docker compose build
```

View exported files:

```bash
ls -lah /mnt/nextcloud-actualbudget-exports
```

View exporter logs:

```bash
sudo tail -f /var/log/actual-exporter.log
```

## WebDAV Mount

Check mount:

```bash
mount | grep nextcloud-actualbudget-exports
```

Mount manually:

```bash
sudo mount /mnt/nextcloud-actualbudget-exports
```

Unmount manually:

```bash
sudo umount /mnt/nextcloud-actualbudget-exports
```

Mount everything from `/etc/fstab`:

```bash
sudo mount -a
```

## oikb

View oikb logs:

```bash
docker logs oikb -f
```

Validate config:

```bash
docker exec -it oikb oikb validate
```

Deep validate config:

```bash
docker exec -it oikb oikb validate --deep
```

Dry run Actual Budget sync:

```bash
docker exec -it oikb oikb sync --name actualbudget-exports --dry-run
```

Run Actual Budget sync:

```bash
docker exec -it oikb oikb sync --name actualbudget-exports
```

List files in the Actual Budget KB:

```bash
docker exec -it oikb oikb ls --kb-id 84256f82-f55d-4c82-9357-ddfc0eae873e
```

Check KB status:

```bash
docker exec -it oikb oikb status --kb-id 84256f82-f55d-4c82-9357-ddfc0eae873e
```

---

# Best Practices Going Forward

## Finances Use a Dedicated Knowledge Base

The Actual Budget exports should stay in their own dedicated Knowledge Base:

```text
Actual Budget Exports
```

This keeps financial data separate from general notes, documents, GitHub repositories, and manually uploaded files. This should also help with creating a model later that will be specializing in financial advice. 

## Prefer Summary Files for Totals

For financial totals, monthly summaries, and category analysis, prefer:

```text
actual-monthly-summary.csv
```

This avoids relying on the model to manually total hundreds or thousands of individual transaction rows.

For individual transaction lookup, use:

```text
actual-transactions.csv
```

## Keep the Export Folder Simple

The export folder should contain only files that should be synced into the Actual Budget Knowledge Base.

Current expected contents:

```text
actual-transactions.csv
actual-monthly-summary.csv
actual-accounts.csv
actual-categories.csv
actual-payees.csv
actual-export-metadata.json
README-for-openwebui.txt
```

Temporary files are excluded from oikb sync:

```yaml
exclude:
  - "*.tmp"
```

---

# Final Notes

The final setup successfully exports Actual Budget data into a Nextcloud-backed folder and syncs that folder into Open WebUI using oikb.

The working path on the Docker host is:

```text
/mnt/nextcloud-actualbudget-exports
```

The exporter writes to:

```text
/out
```

The oikb container reads from:

```text
/sources/actualbudget-exports
```

The target Open WebUI Knowledge Base is:

```text
84256f82-f55d-4c82-9357-ddfc0eae873e
```

The pipeline is now automated, repeatable, and ready for financial questions inside Open WebUI. We'll see if this actually curbs any negative spending habits...
