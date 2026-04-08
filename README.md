# Poolsuite FM for Roon

Stream [Poolsuite FM](https://poolsuite.net/) — the retro-styled, SoundCloud-powered internet radio — directly to your [Roon](https://roonlabs.com/) home audio system.

This bridge runs a local internet radio station that Roon picks up natively as a Live Radio stream, complete with track metadata so Roon can display artist info, album art, and link to matching content in Tidal or Qobuz.

![Poolsuite FM](artwork.jpg)

| Dark Mode | Light Mode |
|-----------|------------|
| ![Dark Mode](screenshots/Poolside_Roon-Dark_mode.jpg) | ![Light Mode](screenshots/Poolside_Roon-Light_mode.jpg) |

![Web UI Demo](screenshots/Poolside_Roon-Darkmode_player_vid.gif)

## How It Works

```
Poolsuite API → yt-dlp (resolve SoundCloud) → ffmpeg (transcode) → HTTP MP3 stream → Roon
```

1. Fetches curated playlists from the [Poolsuite API](https://api.poolsidefm.workers.dev)
2. Resolves SoundCloud tracks to direct audio URLs using `yt-dlp`
3. Transcodes to a constant-bitrate MP3 stream via `ffmpeg`
4. Serves the stream at `http://YOUR_IP:8489/stream`
5. Roon connects and plays it on any zone — with full DSP, grouping, and volume control

Track metadata is injected via the ICY protocol, so Roon displays the current artist and song title in real time and can cross-reference against Tidal/Qobuz for rich metadata.

## Requirements

| Dependency | Version | Purpose |
|------------|---------|---------|
| **Python** | 3.9+ | Runtime |
| **ffmpeg** | Any recent | Audio transcoding (must include `libmp3lame`) |
| **yt-dlp** | Latest recommended | Resolves SoundCloud URLs to direct streams |

### macOS

```bash
brew install python ffmpeg yt-dlp
```

### Ubuntu / Debian

```bash
sudo apt update && sudo apt install -y python3 python3-pip ffmpeg
pip3 install yt-dlp
```

### Arch Linux

```bash
sudo pacman -S python python-pip ffmpeg yt-dlp
```

### Windows (WSL recommended)

Install [WSL](https://learn.microsoft.com/en-us/windows/wsl/install), then follow the Ubuntu instructions above.

## Quick Start

```bash
# Clone the repo
git clone https://github.com/thepeterberg/poolsuite-roon-streaming.git
cd poolsuite-roon-streaming

# Install Python dependencies
pip3 install -r requirements.txt

# Start the bridge
python3 main.py
```

You'll see:

```
============================================================
  Poolsuite -> Roon Bridge
============================================================

  Stream URL:  http://YOUR_LOCAL_IP:8489/stream
  Status:      http://YOUR_LOCAL_IP:8489/status
  Web UI:      http://YOUR_LOCAL_IP:8489/

  Add the stream URL as a Live Radio station in Roon:
    Roon > My Live Radio > + > paste the stream URL

============================================================
```

## Adding to Roon

1. Find your machine's local IP:
   ```bash
   # macOS
   ipconfig getifaddr en0

   # Linux
   hostname -I | awk '{print $1}'
   ```
2. Open **Roon** on any client
3. Go to **My Live Radio** in the sidebar
4. Click **+ Add Station**
5. Paste: `http://YOUR_IP:8489/stream`
6. Name it **Poolsuite FM**
7. Play on any zone

Roon treats this like any internet radio station — zone grouping, volume, DSP, and signal path all work normally.

## CLI Options

```
python3 main.py [options]

Options:
  -c, --config FILE     Path to config JSON file
  -p, --port PORT       HTTP server port (default: 8489)
  --host HOST           Bind address (default: 0.0.0.0)
  --no-shuffle          Play tracks in playlist order
  --playlist NAME       Filter to a specific Poolsuite playlist
  -v, --verbose         Enable debug logging
```

### Examples

```bash
# Verbose logging (recommended for first run)
python3 main.py -v

# Custom port
python3 main.py --port 9000

# Only play tracks from the "Indie" channel
python3 main.py --playlist "Indie"

# Use a config file
python3 main.py --config config.json
```

## Web UI & Controls

The bridge includes a retro Miami 80s-styled stereo interface at `http://YOUR_IP:8489/` with dark and light themes:

- **LCD display** with scrolling marquee showing the current track (linked to SoundCloud)
- **Transport controls** — Previous, Play/Pause, Next
- **Analog channel selector dial** that animates when switching channels
- **Channel buttons** to switch between Poolsuite playlists on the fly
- **In-browser audio player** for listening directly
- **Play history** with SoundCloud links for each track
- **Copy-to-clipboard** buttons for all endpoint URLs

| Endpoint | Description |
|----------|-------------|
| `/` | Web UI — retro stereo interface with transport controls and play history |
| `/stream` | MP3 audio stream (this is what you add to Roon) |
| `/stream.mp3` | Alias for `/stream` |
| `/skip` | Skip to the next track (GET or POST) |
| `/prev` | Go back to the previous track (GET or POST) |
| `/channel?name=X` | Switch channel; omit `name` to list available channels (GET) |
| `/status` | JSON API — now playing, listeners, uptime |

### Track Controls

Roon's transport controls (next/previous) don't work with radio streams. Use these instead:

- **Web UI**: Open `http://YOUR_IP:8489/` and use the **PREV** / **NEXT** transport buttons
- **API**: `curl http://YOUR_IP:8489/skip` or `curl http://YOUR_IP:8489/prev`
- **macOS Shortcut**: Create a Shortcuts automation that fetches the `/skip` or `/prev` URL, then assign a keyboard shortcut
- **Home Assistant / Streamdeck**: Call the `/skip` or `/prev` endpoint as an HTTP action

### Channel Switching

Switch between Poolsuite playlists without restarting:

- **Web UI**: Click any channel button below the dial, or use the `--playlist` CLI flag for the initial channel
- **API**: `curl http://YOUR_IP:8489/channel?name=Indie`
- **List channels**: `curl http://YOUR_IP:8489/channel` (returns JSON with available channels and current selection)

## Configuration

Copy `config.example.json` to `config.json` and edit as needed:

```json
{
  "host": "0.0.0.0",
  "port": 8489,
  "bitrate": "192k",
  "crossfade_seconds": 3,
  "shuffle": true,
  "playlist_filter": null
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `host` | `"0.0.0.0"` | Bind address |
| `port` | `8489` | HTTP server port |
| `bitrate` | `"192k"` | MP3 output bitrate |
| `format` | `"mp3"` | Output audio format |
| `crossfade_seconds` | `2` | Seconds of silence between tracks |
| `shuffle` | `true` | Randomize track order |
| `poolsuite_api` | `"https://api.poolsidefm.workers.dev"` | Poolsuite API base URL |
| `playlist_filter` | `null` | Only play tracks from playlists matching this name |

## Running as a Background Service

### systemd (Linux)

```bash
sudo tee /etc/systemd/system/poolsuite-roon.service << 'EOF'
[Unit]
Description=Poolsuite FM for Roon
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/path/to/poolsuite-roon-streaming
ExecStart=/usr/bin/python3 main.py --config config.json
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now poolsuite-roon.service

# Check status
sudo systemctl status poolsuite-roon.service

# View logs
journalctl -u poolsuite-roon.service -f
```

### launchd (macOS)

```bash
cat > ~/Library/LaunchAgents/com.poolsuite.roon.plist << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.poolsuite.roon</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/python3</string>
        <string>main.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/path/to/poolsuite-roon-streaming</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.poolsuite.roon.plist
```

## Architecture

```
┌──────────────────────────────────────────────────────┐
│  main.py — Orchestrator                              │
│  Fetches playlists, resolves tracks, manages queue    │
│  Pre-resolves next track while current plays          │
│  Feeds realtime silence to keep stream alive in gaps  │
│  Handles skip, previous track, and channel switching  │
├──────────────────────────────────────────────────────┤
│  poolsuite_client.py — Poolsuite API Client           │
│  GET /v1/get_tracks_by_playlist → playlist + tracks   │
│  GET /v2/get_sc_mp3_stream?track_id=X → audio URL     │
│  Rate limiting with exponential backoff               │
├──────────────────────────────────────────────────────┤
│  audio_pipeline.py — Audio Pipeline                   │
│  yt-dlp: resolve SoundCloud → direct audio URL        │
│  ffmpeg master encoder: continuous 192k MP3 stream    │
│  ffmpeg decoder: per-track PCM fed into master encoder│
├──────────────────────────────────────────────────────┤
│  stream_server.py — HTTP Radio Server                 │
│  /stream: continuous MP3 with ICY metadata injection   │
│  /skip, /prev: track navigation                       │
│  /channel: playlist switching                         │
│  /status: JSON now-playing info                       │
│  /: retro stereo web UI with transport + history      │
├──────────────────────────────────────────────────────┤
│  template.html — Retro Miami 80s Web UI               │
│  LCD marquee, transport controls, analog channel dial │
│  Play history with SoundCloud links, dark/light theme │
└──────────────────────────────────────────────────────┘
          │
          ▼
    Roon (Live Radio)
```

## Troubleshooting

**"yt-dlp not found" / "ffmpeg not found"**
Install the missing dependency. Keep yt-dlp updated — SoundCloud extractors break periodically:
```bash
pip3 install -U yt-dlp
```

**Tracks skipping or failing to resolve**
SoundCloud URLs expire and rate limits apply. The bridge retries with backoff and pre-resolves the next track to minimize gaps. Run with `-v` for detailed logs.

**"Address already in use" on startup**
A previous instance is still running. Kill it:
```bash
lsof -ti :8489 | xargs kill -9
```

**Roon says "could not find a radio station at this URL"**
- Make sure the bridge is running and tracks are actively streaming (check the terminal logs)
- Use your machine's actual local IP, not `0.0.0.0` or `localhost`
- Ensure your Roon Core can reach the bridge (same network, port not firewalled)
- Test by opening `http://YOUR_IP:8489/` in a browser first

**Stream dies between tracks**
This was a known issue that's been fixed. Make sure you're on the latest version. The bridge now pre-resolves the next track and pumps silence during transitions.

**Rate limiting (429 errors in logs)**
The bridge automatically retries with exponential backoff (2s, 4s, 8s). If you see persistent 429s, SoundCloud is throttling aggressively — the bridge will recover on its own.

## Poolsuite Channels

The Poolsuite API provides several curated channels. Use `--playlist` to filter:

- **Poolsuite FM** — The flagship mix
- **Indie** — Indie poolside vibes
- **Balearic** — Mediterranean chill
- **Tokyo** — Japanese city pop and funk
- **Friday** — Weekend starters
- **Hangover** — Sunday recovery
- **Mixtapes** — Guest-curated long mixes

## Credits

- [Poolsuite](https://poolsuite.net/) for the incredible curation
- Music is sourced from [SoundCloud](https://soundcloud.com/poolsuite) — support the artists
- Built for [Roon](https://roonlabs.com/) home audio systems
- Powered by [yt-dlp](https://github.com/yt-dlp/yt-dlp) and [ffmpeg](https://ffmpeg.org/)

## License

MIT — for personal use. Please respect SoundCloud's and Poolsuite's terms of service.
