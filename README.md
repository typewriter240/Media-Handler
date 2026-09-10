# Multimedia Handler setup

## First-run dashboard wizard

The dashboard opens an interactive three-step setup wizard the first time it
runs in a browser. The wizard explains the YouTube API key handler and the
spotdl/Spotify handler. It stores only a local browser completion marker; API
secrets are never entered into or stored by the wizard. To see it again, clear
the site's local storage for the dashboard.

## YouTube API key handler

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Create or select a project.
3. Enable **YouTube Data API v3**.
4. Create an API key and restrict it to the API and trusted hosts where
   possible.
5. Add it to `config\config.json`:

```json
"youtube_api_key": "your-youtube-api-key"
```

Restart `server.py` after changing the configuration.

## Spotify Downloader

The Spotify workspace uses `spotdl` first. When Spotify API credentials are
configured, `spotdl` uses Spotify's official Web API through its client
credentials flow. If anonymous Spotify metadata resolution fails and no API
credentials are configured, the server can use its public Spotify metadata and
YouTube matching fallback for supported links.

### 1. Create Spotify API credentials

1. Open the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)
   and sign in.
2. Select **Create app**.
3. Enter an app name and description, accept the terms, and create the app.
4. Open the app's **Settings**.
5. Copy the **Client ID** and reveal/copy the **Client Secret**.

This project uses client credentials, so an OAuth redirect URL is not required.
If Spotify requires one while creating the app, you can register:

```text
http://127.0.0.1:8888/callback
```

### 2. Add credentials to `config.json`

Open `config\config.json` in the project directory and set these fields:

```json
"spotify_client_id": "your-client-id",
"spotify_client_secret": "your-client-secret"
```

Keep the JSON quotes and commas valid. The project configuration already
contains these fields; replace the empty values rather than adding duplicate
keys.

If spotdl reports `HTTP 403` with `Active premium subscription required for
the owner of the app`, Spotify is rejecting the app's Web API credentials.
This is a Spotify account/app entitlement restriction, not an invalid album
URL. The server automatically switches supported track, album, and playlist
links to its public Spotify metadata plus YouTube matching fallback. Successful
fallback jobs are labeled `spotify-public-fallback`. You can also resolve the
restriction by using credentials belonging to an eligible Spotify developer
account and waiting for Spotify's entitlement change to propagate.

Environment variables are safer and override `config.json`:

```powershell
$env:VIDEO_RETRIEVER_YOUTUBE_API_KEY = "your-youtube-api-key"
$env:VIDEO_RETRIEVER_SPOTIFY_CLIENT_ID = "your-client-id"
$env:VIDEO_RETRIEVER_SPOTIFY_CLIENT_SECRET = "your-client-secret"
```

These values are read at startup and are not written back to `config.json`.
Restart `server.py` after changing them.

Login attempts are rate limited per client IP and username. Five failed
attempts within 15 minutes trigger a five-minute lockout.

Root users can use **Network devices → Nearby device scan** to inspect the
hosting computer's local ARP cache. The scan is intentionally root-only and
read-only; it reports recently observed private-LAN IP/MAC entries without
actively probing or connecting to those devices.
When available, reverse DNS, Windows `ping -a`, and NetBIOS names are resolved
in parallel and shown beside each IP; devices without a discoverable hostname
continue to display their IP address.

### 3. Install or update dependencies

From the project directory, run the supplied batch installer:

```bat
.\install-dependencies.bat
```

The batch file upgrades pip and installs every Python package listed in
`requirements.txt` using the active Python interpreter. It also validates that
Python is available and returns a failure exit code if installation fails.

The dashboard includes a customization panel that keeps the default dark-red
look, offers preset design templates, and lets a user choose custom accent
colours while keeping the same overall palette family. The same values are
available as `ui_theme`, `ui_theme_preset`, and custom colour keys in
`config\config.json`.

The equivalent PowerShell command is:

```powershell
python -m pip install --upgrade -r requirements.txt
```

The Spotify artwork and audio conversion fallback also require FFmpeg. Verify
it is available with:

```powershell
ffmpeg -version
```

### 4. Restart the server

Configuration is loaded when a Spotify job starts, but restarting the server
is recommended after changing credentials:

```powershell
python .\server.py
```

Then open the dashboard, choose **Spotify Downloader**, paste a Spotify track,
album, or playlist URL, and submit it.

For album and playlist fallback downloads, configure `youtube_api_key` in
`config\config.json`. Each track is searched through YouTube Data API v3,
validated through the video metadata endpoint, and scored against the Spotify
track title, artist, channel, and official/audio signals before downloading.
If the API is unavailable, the existing yt-dlp search fallback is used.

## Directory configuration

All runtime directories are configurable in `config\config.json`. Relative paths are
resolved from the project directory; absolute paths and environment variables
such as `%USERPROFILE%` are supported. The server creates missing directories
automatically at startup or when configuration is loaded.

```json
"downloads_directory": "./downloads",
"output_directory": "./downloads/Single Videos",
"spotify_output_directory": "./downloads/Spotify",
"download_pages_directory": "./download_pages",
"thumbnails_directory": "./downloads/Single Videos/thumbnails",
"logs_directory": "./logs",
"nginx_temp_directory": "./temp"
```

After changing paths, restart `server.py`. Existing files are not moved
automatically; copy them to the new location if required.

Additional managed directories are created automatically:

```json
"sql_directory": "./data/sql",
"web_directory": "./web",
"favicon_directory": "./web/assets",
"config_directory": "./config",
"failed_downloads_directory": "./data/failed"
```

Managed files are organized as follows:

- `config\config.json` and `config\nginx-video-retriever.conf`
- `data\sql\.video_retriever_accounts.sqlite3`
- `data\.video_retriever_device_policies.json`
- `data\failed\failed_downloads.json`
- `data\history\download_history.json`
- `data\state\download_progress.json` and pause/resume state
- `data\cache\` transient metadata/cache files
- `web\dashboard.html` and `web\assets\favicon.ico`
- `modules\`, `scripts\`, `docs\`, and `extensions\` modular extension areas
- `extensions\youtube-bot-extension\` contains the unpacked browser extension
- `extensions\remote-control-extension\` provides pause, resume, and cancel controls
- The dashboard's **Resource monitor** workspace shows authenticated,
  read-only CPU, memory, disk, and process telemetry. When available, `psutil`
  and Windows Management Instrumentation (`WMI`) are sampled together and
  averaged per metric to reduce single-source readings and false representation.
- The dashboard includes a live queue with pause/resume/cancel controls,
  queue activity graphs, searchable and status-filterable history, drag-and-drop
  URL input, dark/light themes, mobile remote mode, and per-user preference
  persistence
- **Activity and reports** records queue, playback, deletion, and preference
  actions per account; use **Download CSV report** to export the activity log
- Successful root logins append the client IP, hostname, and timestamp as JSON
  lines to `logs\root_login_ips.log`; the path follows `logs_directory`.
- If `psutil` is missing, `server.py` automatically runs
  `python -m pip install psutil` on startup
- The dashboard reports automatic package installations and runs a background
  dependency audit against `requirements.txt`; it reports missing or outdated
  packages without upgrading them automatically
- Downloads use adaptive bandwidth protection by default: above 75% CPU or
  memory pressure the configured rate is reduced proportionally, at 95% it is
  restricted to 1 KB/s, and below 75% the original rate is restored. Configure
  `adaptive_bandwidth_enabled`, the two pressure thresholds, and
  `adaptive_bandwidth_ceiling_kbps` in `config/config.json`. This applies to
  the main YouTube queue, Spotify fallback downloads, and temporary audio
  downloads used for transcription. Spotify spotdl jobs also reduce download
  concurrency to one thread under pressure.
- `docs\` contains the detailed API, architecture, Nginx, operations,
  troubleshooting, and extension guides

The application and Nginx resolver use these locations. Legacy root copies are
migrated when possible and removed only when the destination is identical.

`Bot.py` contains an embedded absolute-directory metadata record. At startup it
compares that record with the file's current absolute directory and reports a
move in the console. The same `script_location` status is included in dashboard
data, while relative configured paths continue to resolve from the detected
workspace root after a move. Update `PYTHON_FILE_METADATA` when intentionally
creating a new distributed copy of the project.

When a move is detected, the first startup resets the `config`, `data`, `logs`,
`temp`, and `download_pages` runtime areas, removes an explicitly configured
cookie file, and clears credential environment variables. Downloaded media is
preserved. A `.relocation-reset.json` marker prevents the reset from repeating
on every subsequent startup in the same moved directory.

Nginx is configured in `config\nginx-video-retriever.conf` to serve the
`web` tree, route the favicon and generated download pages, proxy API/download
traffic to Python, and deny direct access to `config`, `data`, `logs`, and
modular source directories. Run Nginx with that configuration file after
starting `server.py`, then open `http://<host>/dashboard` (port 80) for the
Multimedia Handler dashboard. The dashboard prefers this Nginx origin for API
requests even when it was initially opened directly from Python on port 5000.
Port 5000 remains available as a direct-backend fallback for installations
without Nginx.

If a browser reports `Unable to reach the Video Retriever API` for
`http://<host>:5000/api`, it is bypassing Nginx and connecting directly to the
Python server. Check both services with
`curl http://<host>:5000/api/health` and
`curl http://<host>/api/health`. If the first succeeds and the second fails,
reload Nginx and check its `logs\error.log`; if only remote clients fail,
allow inbound TCP 80 in Windows Firewall. The expected LAN URL is the
port-80 dashboard URL, not the direct port-5000 URL.

## Browser extension

Load `extensions\youtube-bot-extension` as an unpacked extension from your
browser's extension developer page. Its service worker calls the local Python
API on ports 5000/8000 and does not require Nginx to serve extension source.
Nginx intentionally denies direct `/extensions/` access. The authenticated
`/api/extensions/status` endpoint reports whether the managed manifest is
present.

Detailed documentation is available in `docs\README.md`, including API
reference, Nginx deployment, troubleshooting, backups, and extension
development guidance.

## Scripts directory

The `scripts` directory contains operational PowerShell helpers:

- `setup.ps1` creates every configured directory. Add `-InstallDependencies`
  to install `requirements.txt`.
- `validate-config.ps1` validates JSON, credentials presence, and configured
  directory paths.
- `health-check.ps1` checks Python, yt-dlp, FFmpeg, spotdl, and the local API.
- `backup-data.ps1` backs up configuration, Nginx configuration, state, and
  documentation to `backups\`.
- `start-services.ps1` starts the Python server and optionally Nginx with
  `-Nginx`, recording managed process IDs under `data\state`.
- `stop-services.ps1` stops only the process IDs recorded by the start script.
- `cleanup-downloads.ps1` removes files older than the selected age. Use
  `-WhatIf` first; for example `.\scripts\cleanup-downloads.ps1 -Days 30 -WhatIf`.

## Download behavior

- `spotdl` is the primary Spotify URL handler.
- The native `spotdl` executable is tried first, followed by `python -m spotdl`.
- With both credentials set, official Spotify Web API mode is enabled.
- Track, album, and playlist fallback downloads search YouTube using artist-aware
  matching when anonymous spotdl resolution is unavailable.
- Official Spotify album artwork is embedded into the resulting MP3 as an ID3
  attached picture (`attached_pic`). FFmpeg is required for this step.
- Spotify fallback files also receive ID3 title, artist, album, track/total,
  disc, source URL, YouTube comment, and cover fields. `mutagen` is required
  for this metadata pass.
- Spotify album downloads are placed under
  `downloads\Spotify\<album metadata name>\`; tracks and playlists remain
  directly under `downloads\Spotify`.
- Spotify spotdl output defaults to `320k`; change `spotify_audio_quality` in
  `config\config.json` to `auto`, `256k`, `192k`, `128k`, `96k`, `64k`, or
  `32k`.
- The generic yt-dlp integration accepts supported YouTube, SoundCloud,
  Bandcamp, Twitch, TikTok, Instagram, and podcast media URLs. See
  `docs\integrations.md`.
- YouTube labels such as `(Visualizer)`, `(Official Music Video)`, `(Audio)`,
  and trailing video IDs are removed from fallback filenames.

## Credential safety

Do not commit or share the Spotify Client Secret or YouTube API key. Prefer
environment variables for production use, rotate any credential that has been
exposed, and keep `config.json` local/private.
