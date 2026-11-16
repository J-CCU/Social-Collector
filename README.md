# Social Collector

A single-file Python desktop application (`Social_Collector_V65.py`) that collects public account and content metadata from **YouTube**, **Instagram**, and **TikTok** into three separate local SQLite databases. A Tkinter GUI runs one worker subprocess per platform, each with its own cooldown, retry, and completion state.

It is for operators who track a fixed list of public accounts over time and want the raw metadata in local databases for their own analysis.

The collector stores metadata only. It does not download media files and does not bypass DRM, CAPTCHAs, login walls, or other access controls. It performs no IP, cookie, or account rotation.

## Quick start

1. Install Python 3.9+ with Tkinter, then the packages:
   `python3 -m pip install -U requests "yt-dlp[default]" instaloader pyyaml`
2. Next to `Social_Collector_V65.py`, create `social_universe.csv` listing your targets (see [social_universe.csv](#social_universecsv)). Instagram handles in this file on the first run are approved automatically; handles added later need `--v65-approve-instagram @handle`.
3. Optionally add API credentials in `config.yaml` (see [config.yaml](#configyaml)).
4. Run `python3 Social_Collector_V65.py` (Windows: `py Social_Collector_V65.py`) and click **Start**.
5. Results go to `youtube_maximum.db`, `instagram.db`, `tiktok.db`, and `exports/<platform>/*.csv` next to the script (see [Output](#output-databases-and-files)).

## Contents

- [Quick start](#quick-start)
- [Features by platform](#features-by-platform)
- [Requirements](#requirements)
- [Folder layout](#folder-layout)
- [Running](#running)
- [Command-line options](#command-line-options)
- [Configuration](#configuration)
- [Output: databases and files](#output-databases-and-files)
- [Scheduling, rate limits, and backups](#scheduling-rate-limits-and-backups)
- [Troubleshooting](#troubleshooting)
- [Code layout](#code-layout)

## Features by platform

Common to all three platforms:

- Accepts handles, profile/channel URLs, or video/post URLs, one per line.
- Database-first incremental sync: complete items are skipped or get a light metric refresh, incomplete items resume, new items are fully collected.
- Historical snapshots on every pass (account/channel snapshots and video/post snapshots).
- Raw HTML/JSON payloads are kept gzip-compressed in a `raw_data` table, so data can be re-parsed later.
- Persisted rate-limit cooldowns and a pending-item pointer, so an interrupted run resumes where it stopped.
- Optional comment collection, controlled by one shared toggle (off by default).
- Optional "extended content surfaces" (Maximum mode), stored in `surface_items` / `surface_snapshots`.
- Per-run CSV export.

### YouTube

- Resolves handles and URLs to a `UC...` channel ID.
- Uses yt-dlp for video discovery and per-video metadata, plus direct InnerTube requests for additional raw channel/video data.
- Optional YouTube Data API key as a fast path for metadata and comments, with a daily quota budget and a reserve for comments.
- Maximum mode adds caption-track metadata (at most one track per video), available format info, and comments.
- A durable work queue in SQLite (`youtube_work_queue`), with parallel yt-dlp enrichment workers and a separate comment lane.

### Instagram

- Instaloader is the public profile/post collector. An authenticated Instaloader session can be loaded if one is configured.
- Optional Meta Graph API support:
  - Business Discovery for public **professional** accounts (profile plus up to 100 recent media per request).
  - Graph token hot-reload from `config.yaml`, and an optional long-lived token exchange when app credentials are present.
- yt-dlp is an optional per-post enrichment source.
- Stores sidecar (carousel) items, captions, formats, and comments.
- **Account approval gate (V65):** a JSON registry decides which handles may be collected. Handles already in `social_universe.csv` on the first run are approved automatically. Any handle added later stays on hold until you approve it with `--v65-approve-instagram`. Handles classified as confirmed ineligible or nonexistent are skipped, but their history is kept.
- Optional single static HTTP/HTTPS proxy, used only for Instagram public-fallback traffic (never for Graph, YouTube, or TikTok).

### TikTok

- Uses yt-dlp's TikTok extractor and embedded page JSON for profile and video discovery.
- Optional TikTok Research API (needs approved access) for richer account, video, and comment data.
- Optional TikTok API for Business comment collection, for accounts you own only.
- If `curl_cffi` is installed, browser impersonation is available. Rows marked `COMPLETE_LIMITED` are retried once it is present.
- Third-party comments that can't be reached do not block an otherwise complete public payload.

## Requirements

- Python 3.9 or newer (the code uses `zoneinfo`) with Tkinter for the GUI.
- Required: `requests`.
- Strongly recommended (the platform collectors depend on them for normal operation):

| Package | Used for |
|---|---|
| `yt-dlp[default]` | YouTube and TikTok discovery and metadata; Instagram post enrichment |
| `instaloader` | Instagram profile and post collection |
| `pyyaml` | Reading `config.yaml` (it is ignored with a warning if PyYAML is missing) |

- Optional:

| Package / tool | Used for |
|---|---|
| `yt-dlp-ejs` plus Deno, or Node.js 22+ | yt-dlp YouTube JavaScript challenge solving. Deno is preferred; Node 22+ is used automatically as a fallback. Neither runtime is downloaded by the program. If Deno is found but `yt-dlp-ejs` is not installed, yt-dlp is allowed to fetch the EJS solver script from npm. |
| `curl_cffi` | TikTok browser impersonation |
| `psycopg2-binary` | Optional local PostgreSQL backend (see `DATABASE_URL` below) |

Example install:

```bash
# macOS / Linux
python3 -m pip install -U requests "yt-dlp[default]" instaloader pyyaml
python3 -m pip install -U yt-dlp-ejs curl_cffi        # optional

# Windows
py -m pip install -U requests "yt-dlp[default]" instaloader pyyaml
py -m pip install -U yt-dlp-ejs curl_cffi             # optional
```

On macOS, a Homebrew Python may need the separate `python-tk` formula for Tkinter.

## Folder layout

The script resolves every path relative to its own folder (`BASE_DIR`). A typical deployment:

```text
<collector folder>/
  Social_Collector_V65.py
  config.yaml                 # credentials (optional, never commit)
  social_universe.csv         # list of accounts/channels to collect
  local_data_path.txt         # optional: points the data root elsewhere, e.g. "content"
  content/                    # data root (when configured as above)
    youtube_maximum.db
    instagram.db
    tiktok.db
    instagram_qualification_v65.json
    backups/
    exports/
    audit_reports/
```

By default the databases sit next to the script. To keep them in a subfolder such as `content/`, use either:

- environment variable `SOCIAL_COLLECTOR_DATA_DIR=content`, or
- a `local_data_path.txt` file next to the script that contains the folder path. `--migrate-local-data` writes this file for you.

A relative path is resolved against the script folder. The environment variable takes precedence over the pointer file.

## Running

Start the GUI from the collector folder:

```bash
# macOS / Linux
python3 Social_Collector_V65.py

# Windows
py Social_Collector_V65.py
```

On startup the GUI:

1. Takes an exclusive lock (`.social_collector.lock` in the data root). A second GUI instance pointed at the same data root refuses to start.
2. Loads `social_universe.csv` into the per-platform input boxes.
3. Runs a non-blocking startup audit of database completeness.

Main GUI controls:

| Control | Default | Effect |
|---|---|---|
| Mode | `Maximum` | `Fast`, `Full`, or `Maximum`. Maximum adds captions, formats, comments, and extended surfaces. |
| Content limit | `All available` | `Latest 30`, `Latest 100`, `Latest 300`, `Latest 1000`, or `All available` |
| Full historical catalog + refresh | off | Walks the full catalog instead of only the recent window |
| Continuous rotation | on | Keep cycling the platforms instead of stopping after one pass |
| FAST / Attended | on | Checked = high-throughput profile (YouTube targets 600 video starts/hour and 8 yt-dlp workers); unchecked = SAFE / overnight pacing |
| Extended surfaces (Maximum) | on | Collect additional content surfaces in Maximum mode |
| Collect comments | off | Off = fast core collection; on = resume/backfill comments on all platforms where an authorized source exists. Existing comments are never deleted. |
| Start / End (finish current item) / Force Stop | | Start workers; stop gracefully after the current item; terminate immediately |
| Open Data Folder | | Open the data root in the file manager |
| Run Self-Test | | Run the built-in regression self-test |
| API Credentials | | Paste a new Instagram access token or YouTube Data API key. **Save Instagram token** / **Save YouTube key** write it to `config.yaml`; **Reload config** re-reads the file; **Show** reveals the typed value. |
| Advanced proxy… | hidden | Shows the optional single static proxy panel for Instagram public fallback. The password is kept in memory for the session only and never written to disk. **Test Instagram Connection** in this panel fetches the first Instagram handle in the input box through the proxy. If the box is empty it asks you to enter a handle. |
| Platform Status | | Per-platform state (running/cooldown) and completion counts; **Show database/details** adds stored row counts |

## Command-line options

The script has no argparse interface. It recognizes these flags (pass one at a time):

| Flag | Action |
|---|---|
| *(none)* | Launch the GUI |
| `--print-universe` | Print the enabled entries from `social_universe.csv` as JSON |
| `--backup-now` | Make a verified backup of all three databases now |
| `--migrate-local-data [TARGET]` | Copy the three databases to `TARGET` (default `~/SocialCollector/data`, or `%SystemDrive%\SocialCollector\data` on Windows), verify them, then write `local_data_path.txt`. Originals are not deleted. Also copies `instagram_qualification_v65.json` if present, so approvals and on-hold handles carry over. Refuses to overwrite a non-empty destination database or registry file. |
| `--v65-audit` | Write a conservative eligibility/coverage audit CSV per database to `<data root>/audit_reports/` |
| `--v65-instagram-status` | Print the Instagram qualification registry summary as JSON |
| `--v65-approve-instagram @handle` | Approve a new Instagram handle for collection |
| `--v65-classify-instagram @handle STATUS "evidence"` | Set a handle's qualification status (see below) |
| `--production-self-test` | Run the consolidated production self-test |
| `--vNN-self-test` | Run a version-specific regression test (for example `--v65-self-test`, `--v65-qualification-self-test`, `--v64-self-test`, `--v60-self-test`, and older ones) |

Internal flags, used by the GUI and not for manual use: `--platform-worker <platform> <job.json>` and `--worker <job.json>`. `--neon-sync-worker` is disabled in this build.

Instagram qualification statuses:

- Excluded from collection: `CONFIRMED_INELIGIBLE`, `CONFIRMED_NONEXISTENT`. Both require written evidence. A bare HTTP 401/403/404 is rejected as evidence.
- Retried: `UNKNOWN`, `TEMPORARILY_BLOCKED`, `PROFESSIONAL_UNVERIFIED`, `VERIFIED_USABLE`, `CATALOG_INCOMPLETE`.

Examples:

```bash
python3 Social_Collector_V65.py --print-universe
python3 Social_Collector_V65.py --v65-approve-instagram @example_account
py Social_Collector_V65.py --backup-now
```

## Configuration

Precedence differs by credential:

- Instagram Graph values and the YouTube API key: `config.yaml` first, then environment variables, then the legacy token files.
- Every other setting: environment variables only.

### config.yaml

`config.yaml` is optional and must sit next to the script. It may be shared with other tools; this application reads only the `youtube` and `facebook_instagram` (or `instagram`) sections. The collector re-reads it while running (Instagram on each Graph call, YouTube on each API call or platform pass). It writes to the file only when you click Save in the GUI's credential panel. That save updates one key in place, leaves the rest of the file unchanged, and sets the file mode to 600.

Redacted example (values are placeholders):

```yaml
youtube:
  api_key: "<YOUTUBE_DATA_API_KEY>"        # alias: access_token

facebook_instagram:                         # a top-level "instagram:" block is also accepted;
                                            # facebook_instagram wins on conflicts
  access_token: "<META_GRAPH_ACCESS_TOKEN>" # aliases: instagram_access_token, graph_access_token
  instagram_business_id: "<IG_USER_ID>"     # needed for Business Discovery; aliases: instagram_user_id, ig_user_id
  app_id: "<META_APP_ID>"                   # optional, enables long-lived token exchange; aliases: facebook_app_id, client_id
  app_secret: "<META_APP_SECRET>"           # optional; aliases: facebook_app_secret, client_secret
  username: "<your_ig_username>"            # optional; alias: instagram_username
  account_map: {}                           # optional mapping/JSON string of account IDs
```

Other keys in these sections (for example OAuth client or refresh-token fields that other tools use) are ignored by this collector.

### social_universe.csv

This file lists the targets to collect. The header row is:

```csv
platform,locator,enabled
youtube,@examplechannel,1
instagram,@exampleaccount,1
tiktok,@exampleuser,0
```

- `platform`: `youtube`, `instagram`, or `tiktok`. Rows with any other value are ignored.
- `locator`: a handle, URL, or channel/account name. `handle` or `account` are accepted as alternative column names.
- `enabled`: `0`, `false`, `no`, `off`, or `disabled` skip the row. A blank value means enabled.
- Duplicates within a platform are dropped (case-insensitive).

### Credential and cookie files (optional)

All of these are read from the script folder. They contain secrets and must not be committed.

| File | Purpose |
|---|---|
| `config.yaml` | Primary credential store (see above) |
| `instagram_graph_access_token.txt` | Fallback Instagram Graph token. The collector also writes renewed long-lived tokens here (mode 600). |
| `youtube_api_key.txt` | Legacy fallback YouTube Data API key |
| `cookies.json` | Browser-exported YouTube cookies (Netscape or JSON) |
| `tiktok_cookies.json` | Browser-exported TikTok cookies |
| Instaloader session file | Path set by `INSTAGRAM_SESSION_FILE` |

### Environment variables

Credentials:

| Variable | Purpose |
|---|---|
| `YOUTUBE_API_KEY` | YouTube Data API key (after `config.yaml`) |
| `INSTAGRAM_GRAPH_ACCESS_TOKEN`, `INSTAGRAM_GRAPH_IG_USER_ID`, `INSTAGRAM_GRAPH_USERNAME`, `INSTAGRAM_GRAPH_ACCOUNT_MAP` | Meta Graph (after `config.yaml`) |
| `META_APP_ID` / `FACEBOOK_APP_ID` / `INSTAGRAM_GRAPH_APP_ID`, `META_APP_SECRET` / `FACEBOOK_APP_SECRET` / `INSTAGRAM_GRAPH_APP_SECRET` | Meta app credentials for token exchange |
| `INSTAGRAM_LOGIN_USERNAME`, `INSTAGRAM_SESSION_FILE` | Authenticated Instaloader session (no passwords are read) |
| `TIKTOK_RESEARCH_ACCESS_TOKEN`, or `TIKTOK_RESEARCH_CLIENT_KEY` + `TIKTOK_RESEARCH_CLIENT_SECRET` | TikTok Research API |
| `TIKTOK_BUSINESS_ACCESS_TOKEN`, `TIKTOK_BUSINESS_ID`, `TIKTOK_BUSINESS_OWNED_HANDLES` (comma-separated) | TikTok Business comments for owned accounts |
| `SOCIAL_COLLECTOR_IG_PROXY_ENABLED`, `_SCHEME`, `_HOST`, `_PORT`, `_USERNAME`, `_PASSWORD` (prefix `SOCIAL_COLLECTOR_IG_PROXY`) | Static Instagram public-fallback proxy |

Behavior (defaults in parentheses):

| Variable | Purpose |
|---|---|
| `SOCIAL_COLLECTOR_DATA_DIR` | Data root (script folder) |
| `SOCIAL_COLLECT_COMMENTS` | Comment collection (off). When run from the GUI, the **Collect comments** checkbox overrides this for the workers. |
| `SOCIAL_COLLECTOR_EXTENDED_SURFACES` | Extended surfaces (on). Overridden by the GUI checkbox. |
| `SOCIAL_COLLECTOR_IG_PUBLIC_FALLBACK` | Fall back to Instagram's public route when Business Discovery is configured but returns nothing (off) |
| `SOCIAL_COLLECTOR_THROUGHPUT_PROFILE` | `normal` or `high` pacing (normal). Overridden by the GUI **FAST / Attended** checkbox (checked = `high`). |
| `SOCIAL_YOUTUBE_YTDLP_WORKERS`, `SOCIAL_YOUTUBE_COMMENT_WORKERS` | YouTube worker pools, 1 to 8 each (6 and 4; FAST raises yt-dlp workers to 8 unless you set the variable) |
| `YOUTUBE_API_DAILY_BUDGET`, `SOCIAL_YOUTUBE_API_RESERVE` | YouTube API quota budget and comment reserve (8000, 1000) |
| `INSTAGRAM_GRAPH_API_VERSION`, `INSTAGRAM_GRAPH_DAILY_REQUEST_LIMIT` | Graph API version and daily request cap (v24.0, 10000) |
| `SOCIAL_COLLECTOR_IG_GRAPH_MIN_INTERVAL_SECONDS` | Minimum spacing between Graph calls (2.0) |
| `SOCIAL_COLLECTOR_IG_GRAPH_COOLDOWN_INITIAL`, `_MAX` | Graph app-throttle cooldown in seconds (3600, 21600) |
| `SOCIAL_COLLECTOR_IG_ACCOUNT_THROTTLE_SECONDS` | Per-account Instagram public circuit cooldown (1800) |
| `SOCIAL_TIKTOK_POST_COMPLETE_TIMEOUT` | TikTok finalizer watchdog in seconds (120) |
| `SOCIAL_DB_BACKUP_INTERVAL_HOURS`, `SOCIAL_DB_BACKUP_RETENTION` | Automatic backup interval and number of sets kept (24, 3) |
| `SOCIAL_DB_BUSY_TIMEOUT_MS`, `SOCIAL_DB_CACHE_KIB`, `SOCIAL_DB_MMAP_BYTES`, `SOCIAL_DB_WAL_*` | SQLite tuning |
| `SOCIAL_COLLECTOR_ALLOW_LOCAL_POSTGRES` + `DATABASE_URL` | Use PostgreSQL instead of SQLite. Honored only when both are set **and** the URL host is `localhost`, `127.0.0.1`, or `::1`. Requires `psycopg2-binary`. |

The source reads additional `SOCIAL_YOUTUBE_*`, `SOCIAL_TIKTOK_*`, and `SOCIAL_COLLECTOR_IG_*` retry and TTL variables. Their defaults are fine for normal use; search the source for the names.

## Output: databases and files

All of these live under the data root (the script folder unless redirected):

| Path | Contents |
|---|---|
| `youtube_maximum.db` | `channels`, `channel_inputs`, `channel_snapshots`, `videos`, `video_snapshots`, `video_ingestion`, `youtube_work_queue`, `comments`, `caption_tracks`, `formats`, `raw_data`, `scrape_runs`, `errors`, `runtime_state`, `surface_items`, `surface_snapshots` |
| `instagram.db` | `accounts`, `account_inputs`, `account_snapshots`, `posts`, `post_snapshots`, `post_ingestion`, `sidecar_items`, `comments`, `captions`, `formats`, `raw_data`, `scrape_runs`, `errors`, `runtime_state`, `surface_items`, `surface_snapshots` |
| `tiktok.db` | `accounts`, `account_inputs`, `account_snapshots`, `videos`, `video_snapshots`, `video_ingestion`, `comments`, `captions`, `formats`, `raw_data`, `scrape_runs`, `errors`, `runtime_state`, `surface_items`, `surface_snapshots` |
| `instagram_qualification_v65.json` | Instagram approval/qualification registry (kept separate from SQLite) |
| `exports/<platform>/` | Per-run CSV exports |
| `backups/<YYYYMMDD-HHMMSS>/` | Database backup sets with `backup_info.json` |
| `audit_reports/` | Output of `--v65-audit` |
| `staging/` | Legacy local staging databases (not used by the local-only build) |
| `.social_collector.lock` | Single-GUI lock file |

Each database runs in WAL mode, so `-wal` and `-shm` sidecar files appear while the app is running. Temporary `social_collector_*.json` job files are created in the script folder while workers run.

`runtime_state` holds persisted cooldowns (for example `*_rate_limited_until`) and the pending-item pointers.

## Scheduling, rate limits, and backups

**Scheduler.** Each platform runs as its own subprocess, with its own restart backoff (60 s up to 900 s after a worker failure), cooldown, and completion state. A throttle, challenge, or access block pauses only the affected platform; the other platforms keep running. One worker exiting never shuts down the others. The GUI stops only on End or Force Stop. After a successful rotation, a platform waits 5 minutes before its next pass.

**Throttle detection.** Worker output is scanned for HTTP 429, "rate limited", `feedback_required`, `checkpoint_required`, challenge/verification, and similar signals. The resulting cooldown is written to that platform's `runtime_state` table and honored exactly on the next start.

**YouTube pacing.** With FAST / Attended unchecked, yt-dlp video extraction targets 300/240/180 starts per hour in Fast/Full/Maximum mode. With it checked (the default, equivalent to `SOCIAL_COLLECTOR_THROUGHPUT_PROFILE=high`), the target is 600 per hour in every mode and 8 yt-dlp workers are used. The hard floor of 5 s between starts applies in both cases. A detected yt-dlp rate limit triggers a cooldown of about 1 hour. The YouTube Data API path stops when the daily quota budget runs out.

**Instagram pacing.** Graph calls are spaced at least 2 s apart. Meta's app-level throttle (OAuth error code 4) parks Graph traffic for a cooldown that grows from 1 h to 6 h. An expired or invalid token parks only the Instagram worker. Public-route blocks (401, "Please wait") open a per-account circuit.

**TikTok.** HTTP 429 and challenge pages persist a cooldown and a pending item. Items that could not be fully captured are marked `COMPLETE_LIMITED` and retried later.

**Backups.**

- Automatic: every `SOCIAL_DB_BACKUP_INTERVAL_HOURS` (default 24 h) while the GUI is open. The first automatic backup runs one interval after launch, not at startup.
- Manual: `--backup-now`.
- Each backup uses SQLite's online backup API, then runs `PRAGMA quick_check` on the copy. Only the newest `SOCIAL_DB_BACKUP_RETENTION` sets (default 3) are kept.
- WAL files are checkpointed when they grow past 64 MB, and `PRAGMA optimize` runs every 30 minutes.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Another Social Collector is already using <dir>` | A GUI is already running against this data root; close it first. The lock is an OS file lock released when the process exits, so a leftover `.social_collector.lock` file does not block a new launch. |
| `config.yaml found but PyYAML is not installed` | Run `python3 -m pip install pyyaml`. |
| Config changes don't take effect | Use **Reload config** in the GUI. Check that keys sit under `youtube:` or `facebook_instagram:` with consistent indentation. |
| YouTube signature/JS challenge errors from yt-dlp | Install `yt-dlp-ejs` plus Deno, or Node.js 22+, and update yt-dlp (`python3 -m pip install -U "yt-dlp[default]"`). |
| A platform shows COOLDOWN for a long time | The platform reported a rate limit. The cooldown is persisted and honored on purpose; wait it out. Lowering concurrency (uncheck FAST) reduces how often this happens. |
| Instagram handles listed as "approval required (not scraped)" | They were not in `social_universe.csv` when the registry was first created (this includes handles typed only into the GUI box). Run `--v65-approve-instagram @handle`. |
| Instagram "no token configured" / Graph auth invalid | Paste a fresh token in **API Credentials** or update `facebook_instagram.access_token`. Business Discovery also needs `instagram_business_id`. |
| Instagram returns nothing for non-professional accounts while Graph is configured | Business Discovery only covers professional accounts. Set `SOCIAL_COLLECTOR_IG_PUBLIC_FALLBACK=1` to allow the public route. |
| TikTok rows stuck in `COMPLETE_LIMITED` | Install `curl_cffi`. Limited rows are reopened once impersonation is available. |
| Garbled characters in the Windows console | The script forces UTF-8 output. Use Windows Terminal, or set `PYTHONUTF8=1`. |
| `DATABASE_URL` is ignored | By design, PostgreSQL is used only when `SOCIAL_COLLECTOR_ALLOW_LOCAL_POSTGRES=1` is set and the URL points at localhost. |

To check database coverage without changing anything, run `--v65-audit`. It never deletes or disables inputs.

## Code layout

`Social_Collector_V65.py` (about 35,000 lines) was built by merging three standalone collectors and then appending versioned layers. Later layers replace earlier functions by reassignment, so the **last definition of a function is the active one**.

| Lines (approx.) | Section |
|---|---|
| 1-60 | Module docstring, shared imports, UTF-8 console setup |
| 63-5760 | YouTube collector (config, InnerTube, YouTube Data API, yt-dlp, DB, CSV export, engine, original GUI) |
| 5765-8470 | Instagram collector (config, rate-limit state, DB, Instaloader/yt-dlp, engine, exports, GUI) |
| 8472-11620 | TikTok collector (config, cookies, rate limit, DB, yt-dlp, Research API, engine, exports, GUI) |
| 11623-11890 | Extended content-surface tables |
| 11890-12730 | Unified multi-platform scheduler and GUI (`UnifiedApp`, `unified_main`) |
| 12733-27390 | Versioned layers V13-V48: checkpoints, concurrent scheduler (V20), PostgreSQL option (V21), Business Discovery (V23), isolated lifecycle (V28), DB-first sync (V29+), YouTube work queue (V44), local SQLite engine/backups/migration (V46), TikTok Business comments (V47), shared comment toggle (V48) |
| 27392-27546 | Production consolidation: `social_universe.csv` loader and production worker dispatcher |
| 27548-34500 | Versioned layers V49-V65: platform fixes, static proxy (V59), `config.yaml`-first auth (V61), GUI credential paste (V62), compact GUI (V63), conservative audit (V65) |
| 34506-34806 | V65 Instagram qualification registry and the `__main__` command-line dispatch |
