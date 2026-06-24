---
title: Syncing Nextcloud and Outlook Calendars into Open WebUI with caldav2markdown and OIKB
date: 2026-06-22 16:45:00
categories: [open-webui,documentation,sync,calendar]
tags: [openwebui,oikb,calendar,nextcloud,outlook,ics,docker,portainer,knowledge-base]
---

![calendar](https://images.pexels.com/photos/9810172/pexels-photo-9810172.jpeg)

## Giving Open WebUI access to calendar data

Open WebUI is a fantastic product, but one issue I had when trying to incorporate it into my life is the lack of any calendar integrations. I _could_ use Ollama to access my calendars via Outlook, but that's not what I am looking for. I want to be able to see events coming up in my chats directly, so that it can be used as data in queries. Additionally, I want to be able to access all of my calendars this way. I use a mix of Apple Mail for home-use and Outlook for work and school. I can't be vendor-locked to a specific platform or product.

This article documents the final working setup used to pull calendar data from Nextcloud and Outlook, convert that data into Markdown with `caldav2markdown`, and sync the resulting files into an Open WebUI Knowledge Base with `oikb`. The overall design follows the same durable handoff pattern used in the existing Actual Budget and OIKB documentation: a scheduled source sync writes files to a predictable host path, and `oikb` incrementally mirrors those files into a dedicated Knowledge Base.

The final setup uses:

- `caldav2markdown` as the calendar-to-Markdown conversion tool
- A lightweight Alpine container in Portainer to run scheduled sync jobs
- Public ICS feeds for Nextcloud and Outlook calendars
- A host bind mount at `/opt/appdata/caldav2markdown`
- `oikb` to mirror generated Markdown files into Open WebUI
- A dedicated Open WebUI Knowledge Base for calendar data

The resulting pipeline looks like this:

```text
Nextcloud calendar (ICS)
        |
        | public ICS URL
        v
caldav2markdown container
        |
        | Markdown daily files
        v
/opt/appdata/caldav2markdown/output
        |
        | read-only bind mount into oikb
        v
oikb
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

The compiled binary lives at:

```text
/opt/appdata/caldav2markdown/bin/caldav2markdown
```

The verified compiled binary output was:

```text
-rwxr-xr-x 1 root root 7.9M ... /opt/appdata/caldav2markdown/bin/caldav2markdown
ELF 64-bit LSB executable, x86-64, statically linked
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

This setup follows the same host-mounted config and source-path pattern documented for the earlier [OIKB deployment](https://cjs-cloud.com/posts/Deploying-Open-OIKB-for-OpenWebUI/) and [Actual Budget export workflow](https://cjs-cloud.com/posts/automating-actual-budget-exports-into-openwebui/).

---

# Final Working Calendar Sources

The Nextcloud calendar source that produced usable events was a public ICS calendar, not the authenticated CalDAV base path. The working URL was:

```text
https://nextcloud.cjs-cloud.com/remote.php/dav/public-calendars/Your_Feed_Here
```

The work Outlook calendar was added through its published Office 365 ICS feed:

```text
https://outlook.office365.com/owa/calendar/811be2767ec74a8f961726645de533fb@hoffman-hoffman.com/Your_Feed_Here/calendar.ics
```

The school Outlook calendar was added through its published Office 365 ICS feed:

```text
https://outlook.office365.com/owa/calendar/fbf73567276b43718300ec160c0f5329@uncg.edu/Your_Feed_Here/calendar.ics
```

During testing, the Nextcloud ICS source produced 20 events across 4 daily files for a full-year window, while the rolling-window sync later produced 4 events in the near-term range. The Outlook work feed produced 139 events with 29 duplicates skipped and generated 109 daily files, while the UNCG feed authenticated successfully but returned 0 events at the time of testing.

---

# Why ICS Was Used Instead of CalDAV

The original authenticated Nextcloud CalDAV discovery call succeeded and listed five calendars: Personal, Financial, Home, Contact birthdays, and Work. However, the authenticated CalDAV pull returned no useful events in the requested time window, while the public ICS export immediately returned calendar data and produced Markdown output.

For this environment, ICS was the simplest and most reliable approach because:

- `caldav2markdown` supports ICS directly through `-source-mode ics`
- Outlook calendar publishing provided ICS URLs without needing Microsoft Graph or OAuth configuration
- The public Nextcloud ICS feed already exposed the events that needed to be ingested
- The final use case was read-only calendar ingestion into a Knowledge Base, not bidirectional calendar management

That made ICS a better fit than continuing to troubleshoot CalDAV behavior for this pipeline.

---

# Host Directory Layout

The working host layout is:

```text
/opt/appdata/caldav2markdown/
├── bin/
│   └── caldav2markdown
├── output/
│   ├── 0001/
│   └── 2026/
├── sync.sh
└── caldav2markdown.log
```

The generated `output/` tree stores daily Markdown files in year/month/day format, for example:

```text
/opt/appdata/caldav2markdown/output/2026/06/2026-06-07.md
/opt/appdata/caldav2markdown/output/2026/07/2026-07-31.md
/opt/appdata/caldav2markdown/output/2026/09/2026-09-19.md
/opt/appdata/caldav2markdown/output/2026/12/2026-12-06.md
```

One malformed output file also appeared at:

```text
/opt/appdata/caldav2markdown/output/0001/01/0001-01-01.md
```

That file likely corresponds to an event with an invalid or missing date and was excluded from OIKB sync.

---

# Building the Binary

The binary was built outside Portainer because building directly in Portainer was unreliable, and the project required a newer Go version than the original container image provided. A successful build required a Go image new enough for the repository's `go.mod`, and testing showed the repository required Go 1.24.7 or newer.

A prebuilt binary mounted into a lightweight Alpine runtime container proved to be the simplest stable deployment model. This is operationally similar to the pattern used in the attached documentation, where artifacts are prepared in a durable host path and then consumed by a smaller runtime service.
A representative build command is:

```bash
docker run --rm \
  -e GOPATH=/tmp/gopath \
  -e PATH="/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
  -v /opt/appdata/caldav2markdown/bin:/out \
  golang:1.24-alpine \
  sh -c '
    apk add --no-cache git ca-certificates &&
    git clone https://github.com/ksonney/caldav2markdown.git /src &&
    cd /src &&
    go build -ldflags="-w -s" -o /out/caldav2markdown ./cmd/caldav2markdown &&
    ls -lah /out/caldav2markdown
  '
```

After building, the binary should be verified with:

```bash
ls -lah /opt/appdata/caldav2markdown/bin/caldav2markdown
file /opt/appdata/caldav2markdown/bin/caldav2markdown
```

---

# Manual Validation Commands

The binary was first validated inside an Alpine container with a bind mount:

```bash
docker run --rm \
  -v /opt/appdata/caldav2markdown:/data \
  -v /opt/appdata/caldav2markdown/bin/caldav2markdown:/usr/local/bin/caldav2markdown:ro \
  alpine:3.21 \
  sh -c 'apk add --no-cache ca-certificates > /dev/null 2>&1 && /usr/local/bin/caldav2markdown --help'
```

This confirmed the binary executed correctly inside the same minimal runtime used later in the Portainer stack.

A manual ICS test for the Nextcloud feed looked like this:

```bash
docker run --rm \
  -v /opt/appdata/caldav2markdown:/data \
  -v /opt/appdata/caldav2markdown/bin/caldav2markdown:/usr/local/bin/caldav2markdown:ro \
  alpine:3.21 \
  sh -c 'apk add --no-cache ca-certificates > /dev/null 2>&1 && \
  /usr/local/bin/caldav2markdown \
    -source-mode ics \
    -ics-url "https://nextcloud.cjs-cloud.com/remote.php/dav/public-calendars/8L7ALsispt4PxGsB?export" \
    -output /data/output \
    -start 2026-01-01 \
    -end 2026-12-31 \
    -frontmatter \
    -hashtags'
```

That test proved the conversion path worked before the scheduler and OIKB were added.

---

# Final `sync.sh`

The final calendar sync logic was moved to a host-side script to avoid YAML escaping problems and container crash loops. Storing the sync script on the host made the Portainer stack simpler and made the logic survive container restarts.

The working script is:

```bash
#!/bin/sh
START=$(date -d @$(($(date +%s) - 2592000)) +%Y-%m-%d)
END=$(date -d @$(($(date +%s) + 31536000)) +%Y-%m-%d)

echo "[$(date)] Starting calendar sync..." >> /data/caldav2markdown.log

# Nextcloud personal calendar
echo "[$(date)] Syncing Nextcloud..." >> /data/caldav2markdown.log
/usr/local/bin/caldav2markdown \
  -source-mode ics \
  -ics-url "https://nextcloud.cjs-cloud.com/remote.php/dav/public-calendars/Your_Feed_Here" \
  -output /data/output \
  -frontmatter \
  -hashtags \
  -start "$START" \
  -end "$END" >> /data/caldav2markdown.log 2>&1

# Outlook Work calendar
echo "[$(date)] Syncing Work (Outlook)..." >> /data/caldav2markdown.log
/usr/local/bin/caldav2markdown \
  -source-mode ics \
  -ics-url "https://outlook.office365.com/owa/calendar/Your_Feed_Here/calendar.ics" \
  -output /data/output \
  -frontmatter \
  -hashtags \
  -start "$START" \
  -end "$END" >> /data/caldav2markdown.log 2>&1

# Outlook School calendar
echo "[$(date)] Syncing School (UNCG)..." >> /data/caldav2markdown.log
/usr/local/bin/caldav2markdown \
  -source-mode ics \
  -ics-url "https://outlook.office365.com/owa/calendar/Your_Feed_Here/calendar.ics" \
  -output /data/output \
  -frontmatter \
  -hashtags \
  -start "$START" \
  -end "$END" >> /data/caldav2markdown.log 2>&1

echo "[$(date)] Sync complete." >> /data/caldav2markdown.log
```

## What This Script Does

This script calculates a rolling window of approximately 30 days back and 365 days forward using epoch math compatible with the Alpine/BusyBox environment. That avoided the earlier issue where BusyBox `date` rejected GNU-style expressions like `date -d "-30 days"`.

The script then runs `caldav2markdown` three times against three ICS feeds and writes all resulting Markdown into the same output tree. Because `caldav2markdown` merges into existing daily files, the final output becomes one combined daily note stream containing personal, work, and school calendar events.

---

# Final Portainer Stack

The final stable Portainer stack uses the prebuilt binary and host-side sync script:

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

## Why This Stack Works

This stack keeps the container logic minimal. It installs the runtime certificates and timezone data, appends the calendar sync job into Alpine's root crontab, and then runs `crond` in the foreground so the container stays alive.

This pattern is intentionally simpler than embedding a large heredoc in the Compose file. Moving the real logic into `/data/sync.sh` avoided quoting problems, restart loops, and Portainer YAML parsing issues.

---

# Verifying the Running Container

After deployment, the running container was verified with:

```bash
docker ps | grep caldav2markdown
```

The crontab was verified with:

```bash
docker exec caldav2markdown crontab -l
```

The expected output included:

```text
*/30 * * * * /data/sync.sh
```

A manual test run was executed with:

```bash
docker exec caldav2markdown sh /data/sync.sh
```

This produced successful output and confirmed that the calendar conversion pipeline worked end-to-end before relying on cron.

---

# Sync Results Observed During Testing

The final logged test run showed:

- Nextcloud ICS feed: 4 events, 0 tasks, successful merge into existing daily files
- Outlook Work ICS feed: 139 events, 0 tasks, 29 duplicates skipped, 109 daily files created
- Outlook School ICS feed: 0 events, 0 tasks, no failures

Representative log output looked like this:

```text
[Mon Jun 22 16:35:17 EDT 2026] Starting calendar sync...
[Mon Jun 22 16:35:17 EDT 2026] Syncing Nextcloud...
...
Successfully created 4 daily files for 4 events and 0 tasks
[Mon Jun 22 16:35:18 EDT 2026] Syncing Work (Outlook)...
...
Found 139 events, 0 tasks (29 duplicates removed)
Successfully created 109 daily files for 139 events and 0 tasks
[Mon Jun 22 16:35:36 EDT 2026] Syncing School (UNCG)...
...
Found 0 events, 0 tasks (0 duplicates removed)
No events or tasks found.
[Mon Jun 22 16:35:37 EDT 2026] Sync complete.
```

This confirmed that the pipeline was stable and that the work calendar was the primary high-volume source.

---

# OIKB Configuration

The OIKB deployment was already based on the working pattern documented in the earlier Open WebUI setup: mount `/opt/appdata/oikb` to `/config`, set `working_dir: /config`, and run `command: daemon` so OIKB can find `.oikb.yaml` reliably.

The calendar source added to `/opt/appdata/oikb/.oikb.yaml` is:

```yaml
  - name: caldav-calendar
    source: /sources/caldav2markdown
    kb-id: 25447265-97a9-4849-8b70-f748d320f4c3
    interval: 30m
    filter:
      include:
        - "*.md"
      exclude:
        - "0001/**"
```

## Why This Source Path Works

On the Docker host, the Markdown files live at:

```text
/opt/appdata/caldav2markdown/output
```

Inside the OIKB container, that path is mounted read-only as:

```text
/sources/caldav2markdown
```

As with the Actual Budget export setup, `.oikb.yaml` must always reference the container-visible path, not the host path.

---

# Updated OIKB Docker Compose Service

The OIKB container needed one additional bind mount for the calendar output folder. The final pattern is:

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
      - NEXTCLOUD_URL=https://nextcloud.cjs-cloud.com
      - NEXTCLOUD_USER=your-nextcloud-user
      - NEXTCLOUD_PASSWORD=your-nextcloud-app-password
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

This is the same working OIKB pattern already proven in the earlier deployment documentation, with one extra source mount added for the calendar output directory.

---

# Refresh Frequency

There are two schedules in this design:

1. `caldav2markdown` refreshes calendar files every 30 minutes through Alpine cron.
2. `oikb` syncs the generated Markdown files into Open WebUI every 30 minutes through the per-source `interval: 30m` setting.

This means the worst-case lag between a calendar update becoming visible in Open WebUI is roughly 30 minutes, plus any small Open WebUI indexing delay after OIKB uploads the changed files. The OIKB documentation already notes that the daemon supports simple interval strings such as `30m` and per-source schedule overrides, which is exactly what was used here.

---

# Useful Commands

## Calendar Sync

Run the sync script manually:

```bash
docker exec caldav2markdown sh /data/sync.sh
```

Check the generated files:

```bash
find /opt/appdata/caldav2markdown/output -type f | sort
```

Watch the log:

```bash
tail -f /opt/appdata/caldav2markdown/caldav2markdown.log
```

Verify the cron schedule inside the container:

```bash
docker exec caldav2markdown crontab -l
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

Run a manual calendar sync into the Knowledge Base:

```bash
docker exec -it oikb oikb sync --name caldav-calendar
```

List files in the calendar Knowledge Base:

```bash
docker exec -it oikb oikb ls --kb-id 25447265-97a9-4849-8b70-f748d320f4c3
```

---

# Troubleshooting Notes

## Problem: Portainer build failed on `go build`

Early deployment attempts failed while building directly in Portainer because the Go version in use was too old for the repository and because inline build behavior was inconsistent.

Fix:

- Build the binary separately with a new enough Go container.
- Store the binary on the host.
- Mount the binary into a simple runtime container.

## Problem: `go.mod requires go >= 1.24.7`

Cause:

The repository required a newer Go version than the original 1.22 image.

Fix:

Use a Go 1.24 image during build.

## Problem: SQLite error with `-use-database`

Observed error:

```text
Error opening database: failed to enable WAL mode: Binary was compiled with 'CGO_ENABLED=0', go-sqlite3 requires cgo to work. This is a stub
```

Cause:

The statically compiled binary did not support the SQLite backend used by `-use-database`.

Fix:

Remove `-use-database` and `-database-path` from the runtime workflow.

This pipeline does not require the database feature to be useful because it is primarily writing merged daily Markdown files from ICS feeds.

## Problem: BusyBox `date` rejected `-30 days`

Cause:

Alpine's BusyBox `date` does not behave like GNU `date` for relative-date strings.

Fix:

Use epoch math:

```bash
START=$(date -d @$(($(date +%s) - 2592000)) +%Y-%m-%d)
END=$(date -d @$(($(date +%s) + 31536000)) +%Y-%m-%d)
```

## Problem: Portainer stack heredoc caused crash loops

Cause:

The large inline heredoc in the Compose `command:` block was brittle and caused restart loops.

Fix:

Store sync logic in `/opt/appdata/caldav2markdown/sync.sh` on the host and call that from cron.

---

# Open WebUI Usage Notes

A dedicated Knowledge Base was used for the calendar pipeline. This follows the same best practice used in the earlier OIKB and Actual Budget documentation: keep sync-managed content isolated in its own Knowledge Base instead of mixing it with manually uploaded files or unrelated sources.

Recommended prompt examples include:

```text
Based on my calendar, give me a brief for today and what is coming up this week.
```

```text
Summarize my work meetings and personal commitments for the next 7 days.
```

```text
What deadlines, appointments, or scheduling conflicts should I know about?
```

This structure makes the calendar data easy to query while keeping the ingestion side automated.

---

# Final Architecture

The final architecture looks like this:

```text
Nextcloud public ICS
        |
        v
caldav2markdown
        |
        | writes Markdown daily notes
        v
/opt/appdata/caldav2markdown/output
        |
        | mounted read-only into oikb container
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

Work Outlook adds a second major ICS feed into the same output tree, and school Outlook can be added the same way once the correct feed contains events. The end result is a lightweight, durable, and repeatable calendar-to-Knowledge-Base pipeline built on the same operational approach already used for other Open WebUI data ingestion workflows.

---

# caldav2markdown Commands

This table lists all command‑line flags that the tool recognises, grouped by functional area for quick reference.  
All options are prefixed with a single hyphen (`-`).  Arguments may be supplied on a single line or split across multiple lines; either form is accepted.

---

## CalDAV‑specific options

| Flag | Description |
|------|-------------|
| `-test` | Test the connection only; does **not** fetch events. |
| `-server-side-filtering` | Use CalDAV server‑side filtering for faster syncs (pre‑filter events on the server). |
| `-discover-calendars` | Discover and process all calendars present on the server. |
| `-include-calendars` | Comma‑separated list of calendar names to include (e.g., `Work,Personal`). |
| `-exclude-calendars` | Comma‑separated list of calendar names to exclude from processing. |
| `-list-calendars` | List available calendars on the server and exit without syncing. |

---

## Proxy options (applies to all modes)

| Flag | Description |
|------|-------------|
| `-proxy-url` | URL of an HTTP/HTTPS proxy (e.g., `http://proxy.example.com:8080`). |
| `-proxy-username` | Username for proxy authentication. |
| `-proxy-password` | Password for proxy authentication. |

---

## ics mode options

When running in “ICS” mode you must specify a source file or URL, plus any required authentication.

### Source (pick one)

| Flag | Description |
|------|-------------|
| `-ics-path` | Path to a local `.ics` file. |
| `-ics-url`  | Direct URL pointing to an online `.ics` file. |

### Authentication for remote ics

| Flag | Description |
|------|-------------|
| `-ics-auth` | Authentication method: `none`, `basic`, `bearer`, or `header`. Default is `none`. |
| `-ics-username` | Username used when `-ics-auth basic`. |
| `-ics-password` | Password used when `-ics-auth basic`. |
| `-ics-token`   | Bearer token used when `-ics-auth bearer`. |

---

## Database options

These flags enable an optional SQLite database for deduplication and offline use.

| Flag | Description |
|------|-------------|
| `-use-database` | Enable the SQLite database. |
| `-database-path` | Path to the SQLite file (defaults to a file next to the config). |
| `-from-database` | Generate markdown directly from the existing database instead of fetching from CalDAV/ICS. |
| `-db-stats` | Show database statistics and exit. |
| `-db-clear` | Delete all data in the database and exit. |

---

## Common options

These flags are available regardless of mode.

| Flag | Description |
|------|-------------|
| `-output` | Directory where markdown files will be written (`./events` by default). |
| `-config` | Path to a YAML configuration file (defaults to `/home/csadmin/.config/caldav2markdown/config.yaml`). |
| `-start` | Start date for events in `YYYY‑MM‑DD` format (default: `2000‑01‑01`). |
| `-end` | End date for events in `YYYY‑MM‑DD` format (default: two years from now). |
| `-output-format` | Output format: `markdown`, `org`, `org-diary`, or `diary`. Default is `markdown`. |
| `-emoji` | Prefix due dates with an emoji (disabled by default; can be enabled via this flag). |
| `-hashtags` | Append `#event` and/or `#task` hashtags to entries. |
| `-frontmatter` | Prepend a YAML front‑matter block to markdown files. |
| `-ignore-descriptions` | Skip event descriptions in the output. |
| `-event-checkboxes` | Render events as task‑style checkboxes (`[ ]`). |
| `-obsidian-tasks` | Enable the Obsidian preset: checkboxes, ignore descriptions, frontmatter, emojis, and hashtags. |
| `-single-file` | Generate a single markdown file instead of one per day. |
| `-single-file-name` | Filename to use when `-single-file` is set (default: `calendar.md` or `calendar.org`). |
| `-weekly-file` | Create weekly files (`Monday‑Sunday`, ISO week numbers) rather than daily ones. |

---

### Quick example usage

```bash
# Test connectivity only
caldav2markdown sync -test

# Full sync with server‑side filtering and calendar discovery
caldav2markdown sync \
  -server-side-filtering \
  -discover-calendars \
  --output ./my_events \
  --config ~/.config/caldav2markdown/config.yaml

# Sync a specific calendar, exclude another, using a proxy
caldav2markdown sync \
  -include-calendars "Work" \
  -exclude-calendars "Spam" \
  -proxy-url http://proxy.example.com:8080 \
  -output ./work_events

# Load events from an online .ics file with bearer auth
caldav2markdown sync \
  -ics-url https://example.com/agenda.ics \
  -ics-auth bearer \
  -ics-token "eyJhbGciOiJI..."
