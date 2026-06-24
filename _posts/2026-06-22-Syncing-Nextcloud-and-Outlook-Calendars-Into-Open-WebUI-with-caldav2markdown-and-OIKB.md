---
title: Syncing Nextcloud and Outlook Calendars into Open WebUI with caldav2markdown and OIKB
date: 2026-06-24 10:49:00
categories: [open-webui, documentation, sync, calendar]
tags: [openwebui, oikb, calendar, nextcloud, outlook, ics, docker, portainer, knowledge-base]
---

![calendar](https://images.pexels.com/photos/9810172/pexels-photo-9810172.jpeg)

## Giving Open WebUI access to calendar data

Open WebUI is a fantastic product, but one issue I had when trying to incorporate it into my life is the lack of any calendar integrations. I could use Outlook or another connected service directly for some tasks, but that is not the workflow I wanted. I wanted calendar data to exist as searchable Knowledge Base content inside Open WebUI so it could be referenced naturally in chats.

This article documents the final working setup used to pull calendar data from Nextcloud and Outlook, convert that data into Markdown with `caldav2markdown`, and sync the resulting files into an Open WebUI Knowledge Base with `oikb`.

The final setup uses:

- `caldav2markdown` as the calendar-to-Markdown conversion tool
- Public ICS feeds for Nextcloud and Outlook calendars
- Three dedicated YAML config files, one per calendar source
- A host bind mount at `/opt/appdata/caldav2markdown`
- A shared output tree at `/opt/appdata/caldav2markdown/output`
- `oikb` to mirror generated Markdown files into Open WebUI
- A dedicated Open WebUI Knowledge Base for calendar data

The resulting pipeline looks like this:

```text
Nextcloud ICS / Outlook ICS
        |
        v
caldav2markdown container
        |
        | daily markdown files
        v
/opt/appdata/caldav2markdown/output
        |
        | read-only bind mount into oikb
        v
/sources/caldav2markdown
        |
        | incremental Knowledge Base sync
        v
Open WebUI Knowledge Base
```

---

# Environment Summary

The host path used for this project is:

```text
/opt/appdata/caldav2markdown
```

That host path is bind-mounted into the running container as:

```text
/data
```

The compiled binary lives at:

```text
/opt/appdata/caldav2markdown/bin/caldav2markdown
```

The generated Markdown output path is:

```text
/opt/appdata/caldav2markdown/output
```

The OIKB config directory already in use on the host is:

```text
/opt/appdata/oikb
```

The Open WebUI Knowledge Base UUID used for calendar sync is:

```text
25447265-97a9-4849-8b70-f748d320f4c3
```

---

# Final Working Calendar Sources

The final working setup uses three ICS feeds:

1. A public Nextcloud ICS calendar
2. A published Outlook work ICS calendar
3. A published Outlook school ICS calendar

Representative source formats are:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/public-calendars/Your_Feed_Here?export
https://outlook.office365.com/owa/calendar/<work-mailbox>/<feed-id>/calendar.ics
https://outlook.office365.com/owa/calendar/<school-mailbox>/<feed-id>/calendar.ics
```

The current observed sync results are:

- Nextcloud ICS feed: 9 events, 0 tasks, 7 daily files created
- Outlook Work ICS feed: 170 events, 0 tasks, 30 duplicates skipped, 140 daily files created
- Outlook School ICS feed: 0 events, 0 tasks, no failures

The school feed is still configured correctly, but it returned no events in the tested date window.

---

# Why ICS Was Used Instead of CalDAV

The original authenticated Nextcloud CalDAV discovery path was explored first, but for this environment the simplest reliable ingestion path ended up being published ICS feeds.

ICS was the better fit here because:

- Nextcloud public calendar exports were easy to consume
- Outlook calendar publishing exposed direct ICS URLs
- The goal was read-only ingestion into Open WebUI
- No bidirectional sync or write-back was needed

This pipeline is focused on durable read-only ingestion, not calendar management.

---

# Host Directory Layout

The working host layout is:

```text
/opt/appdata/caldav2markdown/
├── bin/
│   └── caldav2markdown
├── config-nextcloud.yaml
├── config-school.yaml
├── config-work.yaml
├── output/
│   ├── 0001/
│   ├── 2026/
│   └── 2027/
├── sync.sh
├── caldav2markdown.log
└── caldav2markdown.log.bak
```

The generated `output/` tree stores daily Markdown files in year/month/day format, for example:

```text
/opt/appdata/caldav2markdown/output/2026/06/2026-06-07.md
/opt/appdata/caldav2markdown/output/2026/07/2026-07-31.md
/opt/appdata/caldav2markdown/output/2026/09/2026-09-19.md
/opt/appdata/caldav2markdown/output/2027/06/2027-06-22.md
```

One malformed output file also appeared at:

```text
/opt/appdata/caldav2markdown/output/0001/01/0001-01-01.md
```

That file is excluded from OIKB sync.

---

# Final Config Model

The current setup does **not** use one giant YAML file for all calendars. It uses one YAML file per ICS source.

## `config-nextcloud.yaml`

```yaml
source_mode: ics

ics_url: "https://nextcloud.cjs-cloud.com/remote.php/dav/public-calendars/Your_Feed_Here?export"
ics_auth: none

output_dir: /data/output
use_frontmatter: true
use_hashtags: true
ignore_descriptions: true
exclude_calendars: "Day Planning"
```

## `config-work.yaml`

```yaml
source_mode: ics

ics_url: "https://outlook.office365.com/owa/calendar/<work-mailbox>/<feed-id>/calendar.ics"
ics_auth: none

output_dir: /data/output
use_frontmatter: true
use_hashtags: true
ignore_descriptions: true
```

## `config-school.yaml`

```yaml
source_mode: ics

ics_url: "https://outlook.office365.com/owa/calendar/<school-mailbox>/<feed-id>/calendar.ics"
ics_auth: none

output_dir: /data/output
use_frontmatter: true
use_hashtags: true
ignore_descriptions: true
```

## Important note about output

Although `output_dir: /data/output` is present in the YAML files, the deployed binary only wrote into the intended bind-mounted output tree reliably when `-output /data/output` was passed explicitly on the command line. Because of that, the final working setup enforces output location in `sync.sh`.

---

# Final `sync.sh`

The final calendar sync logic uses a host-side script that calculates a rolling date window, calls each source-specific config, and forces all output into the shared `/data/output` tree.

```bash
#!/bin/sh
START=$(date -d @$(($(date +%s) - 2592000)) +%Y-%m-%d)
END=$(date -d @$(($(date +%s) + 31536000)) +%Y-%m-%d)

LOG="/data/caldav2markdown.log"
OUT="/data/output"

echo "[$(date)] Starting calendar sync..." >> "$LOG"
echo "[$(date)] Date range: $START to $END" >> "$LOG"

echo "[$(date)] Syncing Nextcloud..." >> "$LOG"
/usr/local/bin/caldav2markdown \
  -config /data/config-nextcloud.yaml \
  -start "$START" \
  -end "$END" \
  -output "$OUT" >> "$LOG" 2>&1
RC1=$?

echo "[$(date)] Syncing Work (Outlook)..." >> "$LOG"
/usr/local/bin/caldav2markdown \
  -config /data/config-work.yaml \
  -start "$START" \
  -end "$END" \
  -output "$OUT" >> "$LOG" 2>&1
RC2=$?

echo "[$(date)] Syncing School (UNCG)..." >> "$LOG"
/usr/local/bin/caldav2markdown \
  -config /data/config-school.yaml \
  -start "$START" \
  -end "$END" \
  -output "$OUT" >> "$LOG" 2>&1
RC3=$?

if [ "$RC1" -eq 0 ] && [ "$RC2" -eq 0 ] && [ "$RC3" -eq 0 ]; then
  echo "[$(date)] Sync complete." >> "$LOG"
  exit 0
else
  echo "[$(date)] Sync failed. Exit codes: nextcloud=$RC1 work=$RC2 school=$RC3" >> "$LOG"
  exit 1
fi
```

## What this script does

This script calculates a rolling window of about 30 days back and 365 days forward using epoch math. That avoids BusyBox `date` issues with relative-date strings inside Alpine.

It then runs `caldav2markdown` three times, once per calendar source, and writes all generated Markdown into the same output tree. Because the tool merges into daily files, the final result is one combined date-based note stream containing personal, work, and school events.

---

# Final Portainer Stack

The final stable Portainer stack keeps the container lightweight and runs the sync script on a 30-minute cron schedule:

```yaml
version: "3.8"
services:
  caldav2markdown:
    image: alpine:3.21
    container_name: caldav2markdown
    restart: unless-stopped
    environment:
      - TZ=America/New_York
    volumes:
      - /opt/appdata/caldav2markdown:/data
      - /opt/appdata/caldav2markdown/bin/caldav2markdown:/usr/local/bin/caldav2markdown:ro
    entrypoint: ["sh", "-c"]
    command:
      - |
        apk add --no-cache ca-certificates tzdata > /dev/null 2>&1
        echo '*/30 * * * * /data/sync.sh' >> /etc/crontabs/root
        crond -f -l 2
```

This keeps the stack simple and makes the real logic live in `/data/sync.sh`, where it is easy to edit and survives restarts.

---

# Verifying the Running Container

Useful validation commands:

```bash
docker ps | grep caldav2markdown
docker exec -it caldav2markdown /bin/sh -c "/data/sync.sh"
docker exec -it caldav2markdown /bin/sh -c "ls -R /data/output"
find /opt/appdata/caldav2markdown/output -type f | head -n 20
tail -n 50 /opt/appdata/caldav2markdown/caldav2markdown.log
```

These verify:

- the container is running
- the sync script executes successfully
- markdown files are being written inside the container
- the same files are visible on the host via the bind mount
- the log shows per-source results

---

# Sync Results Observed During Testing

The currently validated run showed:

- Nextcloud: 9 events, 7 daily files created
- Work Outlook: 170 events, 30 duplicates skipped, 140 daily files created
- School Outlook: 0 events, no errors

Representative output structure after a successful sync looked like:

```text
/data/output/2026/05/2026-05-25.md
/data/output/2026/06/2026-06-24.md
/data/output/2026/12/2026-12-29.md
/data/output/2027/06/2027-06-22.md
```

This confirmed that the work calendar is the primary high-volume source, while the Nextcloud feed adds a smaller set of personal events into the same merged daily note stream.

---

# OIKB Configuration

OIKB is configured from `.oikb.yaml` in the OIKB config directory. The calendar source should be added under the top-level `sources:` key.

```yaml
sources:
  - name: caldav-calendar
    source: /sources/caldav2markdown
    kb-id: 25447265-97a9-4849-8b70-f748d320f4c3
    interval: 30m
    filter:
      include:
        - "**/*.md"
      exclude:
        - "0001/**"
```

## Why this source path works

On the Docker host, the Markdown files live at:

```text
/opt/appdata/caldav2markdown/output
```

Inside the OIKB container, that host path is mounted read-only as:

```text
/sources/caldav2markdown
```

The `.oikb.yaml` file must always reference the **container-visible** source path, not the host path.

---

# Updated OIKB Docker Compose Service

The OIKB service needs the calendar output mounted into the container:

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
    - LOG_FORMAT=json
  volumes:
    - /opt/appdata/oikb:/config:ro
    - /opt/appdata/caldav2markdown/output:/sources/caldav2markdown:ro
  working_dir: /config
  command: daemon
  ports:
    - "8035:8080"
  depends_on:
    - openwebui
```

The important part is the additional read-only source mount for the calendar Markdown output.

---

# Refresh Frequency

There are two schedules in this design:

1. `caldav2markdown` refreshes the source Markdown files every 30 minutes through Alpine cron.
2. `oikb` syncs the generated Markdown files into Open WebUI every 30 minutes through the per-source `interval: 30m` setting.

That keeps calendar data reasonably fresh without adding unnecessary complexity.

---

# Useful Commands

## Calendar Sync

Run the sync script manually:

```bash
docker exec caldav2markdown sh /data/sync.sh
```

List generated files:

```bash
find /opt/appdata/caldav2markdown/output -type f | sort | head -n 50
```

Watch the log:

```bash
tail -f /opt/appdata/caldav2markdown/caldav2markdown.log
```

Check the container-side output tree:

```bash
docker exec -it caldav2markdown /bin/sh -c "ls -R /data/output"
```

## OIKB

Restart OIKB after editing `.oikb.yaml`:

```bash
docker restart oikb
```

Check recent OIKB logs:

```bash
docker logs oikb --tail 50
```

Validate OIKB config:

```bash
docker exec -it oikb oikb validate
```

Dry-run the calendar sync source:

```bash
docker exec -it oikb oikb sync --name caldav-calendar --dry-run
```

Run a manual calendar sync:

```bash
docker exec -it oikb oikb sync --name caldav-calendar
```

List files in the calendar Knowledge Base:

```bash
docker exec -it oikb oikb ls --kb-id 25447265-97a9-4849-8b70-f748d320f4c3
```

---

# Troubleshooting Notes

## Problem: `output_dir` in YAML did not populate `/data/output`

Observed behavior:

The YAML configs loaded correctly, but files did not appear under `/data/output` until `-output /data/output` was added to the runtime command.

Fix:

Pass the output directory explicitly in `sync.sh`:

```bash
-output /data/output
```

## Problem: OIKB reported `0 files found`

Cause:

The initial OIKB filter used `*.md`, which only matched top-level markdown files and missed nested year/month output paths.

Fix:

Use a recursive include pattern:

```yaml
filter:
  include:
    - "**/*.md"
  exclude:
    - "0001/**"
```

## Problem: `0001/01/0001-01-01.md` appeared in output

Cause:

At least one generated item was written with an invalid or malformed date.

Fix:

Exclude that path from OIKB sync:

```yaml
exclude:
  - "0001/**"
```

## Problem: BusyBox `date` rejected GNU relative date syntax

Cause:

Alpine's BusyBox `date` does not behave like GNU `date` for relative-date strings such as `-30 days`.

Fix:

Use epoch math:

```bash
START=$(date -d @$(($(date +%s) - 2592000)) +%Y-%m-%d)
END=$(date -d @$(($(date +%s) + 31536000)) +%Y-%m-%d)
```

---

# Open WebUI Usage Notes

A dedicated Knowledge Base was used for the calendar pipeline. That keeps sync-managed content isolated from manually uploaded files and other unrelated knowledge sources.

Good prompt examples include:

```text
Based on my calendar, give me a brief for today and what is coming up this week.
```

```text
Summarize my work meetings and personal commitments for the next 7 days.
```

```text
What deadlines, appointments, or scheduling conflicts should I know about?
```

---

# Final Architecture

The final architecture looks like this:

```text
Nextcloud / Outlook ICS feeds
        |
        v
caldav2markdown
        |
        | writes daily markdown notes
        v
/opt/appdata/caldav2markdown/output
        |
        | mounted read-only into oikb
        v
/sources/caldav2markdown
        |
        | incremental sync every 30 minutes
        v
Open WebUI Knowledge Base
        |
        | attached to chats/models
        v
AI calendar briefings and schedule questions
```

The end result is a lightweight, durable, and repeatable calendar-to-Knowledge-Base pipeline that uses ICS feeds, merges them into daily Markdown files, and keeps them queryable inside Open WebUI.
