# 🎬 *Arr Stack with TorBox & Plex on Windows 10 via Docker

A complete guide to setting up a self-hosted media automation stack on Windows 10 using Docker, TorBox as a debrid service, and Plex for media playback. This guide covers Radarr, Sonarr, Lidarr, Prowlarr, Decypharr, and Pulsarr — all running in Docker containers with automatic Plex watchlist integration.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setting Up WSL2 on Windows 10](#setting-up-wsl2-on-windows-10)
- [Installing Docker Desktop](#installing-docker-desktop)
- [Installing Portainer](#installing-portainer)
- [Folder Structure](#folder-structure)
- [Docker Compose Stack](#docker-compose-stack)
- [Decypharr Setup](#decypharr-setup)
- [Prowlarr Setup](#prowlarr-setup)
- [Radarr Setup](#radarr-setup)
- [Sonarr Setup](#sonarr-setup)
- [Lidarr Setup](#lidarr-setup)
- [Pulsarr Setup](#pulsarr-setup)
- [TorBoxarr Setup](#torboxarr--usenet-downloads)
- [Plex Integration](#plex-integration)
- [How It All Works Together](#how-it-all-works-together)

---

## Overview

This stack automates the discovery, downloading, and organisation of movies, TV shows, and music. TorBox acts as a debrid service — it caches torrent files on its servers and serves them to you as direct downloads, removing the need for a VPN or direct P2P connections.

### Stack Components

| App | Purpose | Port |
|---|---|---|
| **Radarr** | Movie automation | 7878 |
| **Sonarr** | TV show automation | 8989 |
| **Lidarr** | Music automation | 8686 |
| **Prowlarr** | Indexer management | 9696 |
| **Decypharr** | TorBox/debrid integration | 8282 |
| **Pulsarr** | Plex watchlist automation | 3003 |
| **TorBoxarr** | TorBox Usenet integration | 8085 |
| **Portainer** | Docker management UI | 9443 |

---

## Prerequisites

- Windows 10 (build 19041 or higher recommended)
- At least 8GB RAM
- A [TorBox](https://torbox.app) account and API key
- A [Plex](https://plex.tv) account and media server
- Sufficient storage for your media library

---

## Setting Up WSL2 on Windows 10

Docker Desktop on Windows 10 requires WSL2 (Windows Subsystem for Linux 2). Follow these steps carefully.

### Step 1: Enable Required Windows Features

Open **PowerShell as Administrator** and run:

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Restart your PC when prompted.

### Step 2: Install the WSL2 Kernel Update

Download and install the WSL2 Linux kernel update package directly from Microsoft:

👉 [WSL2 Kernel Update](https://aka.ms/wsl2kernel)

Run the installer, then open PowerShell as Administrator and run:

```powershell
wsl --install
wsl --set-default-version 2
```

This installs Ubuntu as your default WSL2 distribution. You may be prompted to create a Linux username and password — set anything simple as it is only used internally.

### Step 3: Verify WSL2

```powershell
wsl --list --verbose
```

The output should show your Ubuntu distribution with **VERSION 2**.

### Step 4: Enable Virtualisation in BIOS

Docker requires hardware virtualisation to be enabled in your system BIOS:

- **Intel CPUs:** Enable `Intel Virtualization Technology (VT-x)`
- **AMD CPUs:** Enable `SVM Mode` or `AMD-V`

The location varies by motherboard manufacturer but is typically found under **Advanced → CPU Configuration**.

---

## Installing Docker Desktop

### Step 1: Download

Download Docker Desktop for Windows from [docker.com](https://www.docker.com/products/docker-desktop/).

### Step 2: Install

Right-click the installer and select **Run as administrator**. Follow the prompts and restart when complete.

> **Note:** If you encounter a `ProgramData\DockerDesktop must be owned by an elevated account` error, delete `C:\ProgramData\DockerDesktop`, reboot, and run the installer as administrator again.

### Step 3: Configure File Sharing

Docker Desktop needs explicit permission to access your media drives:

1. Open Docker Desktop
2. Go to **Settings → Resources → File Sharing**
3. Add each drive that contains your media (e.g. `D:\`, `E:\`, `F:\`, `G:\`)
4. Click **Apply & Restart**

### Step 4: Configure WSL2 Resource Limits (Recommended)

To prevent Docker from consuming all available RAM, create a `.wslconfig` file:

Open Notepad and save the following to `C:\Users\YOUR_USERNAME\.wslconfig`:

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
```

Adjust values based on your system specifications.

---

## Installing Portainer

Portainer provides a web UI for managing your Docker containers.

Open **PowerShell as Administrator** and run:

```powershell
docker volume create portainer_data

docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v portainer_data:/data `
  portainer/portainer-ce:latest
```

Access Portainer at `https://localhost:9443` and create your admin account.

---

## Folder Structure

### Windows Host Folders

Create the following folder structure on your Windows machine. Adjust drive letters to match your setup:

```
C:\docker\arr\
├── radarr\          # Radarr config
├── sonarr\          # Sonarr config
├── lidarr\          # Lidarr config
├── prowlarr\        # Prowlarr config
├── decypharr\       # Decypharr config (config.json goes here)
└── pulsarr-data\    # Pulsarr database

G:\
├── Movies\          # Radarr media library
├── TV\              # Sonarr media library
└── Downloads\
    └── Completed\
        ├── radarr\  # Radarr download staging
        ├── sonarr\  # Sonarr download staging
        └── lidarr\  # Lidarr download staging

E:\
└── Music\           # Lidarr media library
```

> **Important:** The download staging folder names are case-sensitive inside Docker. Use lowercase for `radarr`, `sonarr`, and `lidarr` subfolders.

Create the config folders:

```powershell
mkdir C:\docker\arr\radarr
mkdir C:\docker\arr\sonarr
mkdir C:\docker\arr\lidarr
mkdir C:\docker\arr\prowlarr
mkdir C:\docker\arr\decypharr
mkdir C:\docker\arr\pulsarr-data
mkdir G:\Downloads\Completed\radarr
mkdir G:\Downloads\Completed\sonarr
mkdir G:\Downloads\Completed\lidarr
```

### How Docker Sees Your Folders

Inside Docker containers, Windows paths are mapped to Linux paths:

| Windows Path | Inside Container | Used By |
|---|---|---|
| `G:\Movies` | `/movies` | Radarr |
| `G:\TV` | `/tv` | Sonarr |
| `E:\Music` | `/music` | Lidarr |
| `G:\Downloads\Completed` | `/downloads` | All apps |

---

## Docker Compose Stack

Create `C:\docker\arr\docker-compose.yml` with the following content. Replace drive letters to match your setup:

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
    volumes:
      - C:\docker\arr\radarr:/config
      - //g/Movies:/movies
      - //g/Downloads/Completed:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
    volumes:
      - C:\docker\arr\sonarr:/config
      - //g/TV:/tv
      - //g/Downloads/Completed:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped

  lidarr:
    image: ghcr.io/hotio/lidarr:nightly
    container_name: lidarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
    volumes:
      - C:\docker\arr\lidarr:/config
      - //e/Music:/music
      - //g/Downloads/Completed:/downloads
    ports:
      - 8686:8686
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
    volumes:
      - C:\docker\arr\prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  decypharr:
    image: ghcr.io/sirrobot01/decypharr:latest
    container_name: decypharr
    volumes:
      - C:\docker\arr\decypharr\config.json:/app/config.json
      - //g/Downloads/Completed:/downloads
    ports:
      - 8282:8282
    restart: unless-stopped

  pulsarr:
    image: lakker/pulsarr:latest
    container_name: pulsarr
    ports:
      - 3003:3003
    volumes:
      - C:\docker\arr\pulsarr-data:/app/data
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
    restart: unless-stopped
```

Deploy the stack:

```powershell
cd C:\docker\arr
docker compose up -d
```

> **Note:** On non-C drives, Docker Desktop for Windows requires forward-slash UNC path format. Use `//g/Movies` instead of `G:\Movies` in volume mappings.

---

## Decypharr Setup

Decypharr acts as a qBittorrent-compatible interface that routes downloads through TorBox instead of direct torrenting. The arr apps think they are talking to qBittorrent, but Decypharr transparently handles everything through your debrid service.

### Create config.json

Create `C:\docker\arr\decypharr\config.json`:

```json
{
  "bind_address": "0.0.0.0",
  "url_base": "/",
  "port": "8282",
  "log_level": "info",
  "debrids": [
    {
      "provider": "torbox",
      "name": "Torbox",
      "api_key": "YOUR_TORBOX_API_KEY",
      "download_api_keys": [
        "YOUR_TORBOX_API_KEY"
      ],
      "rate_limit": "250/minute",
      "unpack_rar": true,
      "minimum_free_slot": 1,
      "torrents_refresh_interval": "10m",
      "download_links_refresh_interval": "40m",
      "workers": 300,
      "auto_expire_links_after": "3d"
    }
  ],
  "arrs": [
    {
      "name": "radarr",
      "host": "http://radarr:7878",
      "token": "YOUR_RADARR_API_KEY",
      "download_uncached": false,
      "selected_debrid": "Torbox"
    },
    {
      "name": "sonarr",
      "host": "http://sonarr:8989",
      "token": "YOUR_SONARR_API_KEY",
      "download_uncached": false,
      "selected_debrid": "Torbox"
    },
    {
      "name": "lidarr",
      "host": "http://lidarr:8686",
      "token": "YOUR_LIDARR_API_KEY",
      "download_uncached": true,
      "selected_debrid": "Torbox"
    }
  ],
  "download_folder": "/downloads",
  "refresh_interval": "15s",
  "categories": [
    "radarr",
    "sonarr",
    "lidarr"
  ],
  "folder_naming": "original_no_ext",
  "default_download_action": "download",
  "retries": 3,
  "remove_stalled_after": "10m",
  "notifications": {}
}
```

Replace all `YOUR_*_API_KEY` placeholders with your actual keys. API keys for the arr apps are found under **Settings → General → Security** in each app after first launch.

### How Decypharr Works

When an arr app finds a torrent it wants to download, it sends it to Decypharr via the qBittorrent API. Decypharr then:

1. Submits the torrent to TorBox
2. Waits for TorBox to cache/download the file
3. Downloads the cached file from TorBox's servers to your `/downloads` folder
4. Reports completion back to the arr app for import

This means you are never downloading directly from peers — everything comes from TorBox's servers.

---

## Prowlarr Setup

Prowlarr manages all your torrent indexers in one place and syncs them automatically to Radarr, Sonarr, Lidarr, and any other connected apps.

### Connect Apps to Prowlarr

Go to `http://localhost:9696` → **Settings → Apps** and add each arr app:

For each app you need:
- The app's internal Docker URL (e.g. `http://radarr:7878`)
- The app's API key (found in each app under Settings → General → Security)
- The Prowlarr URL as seen by each app (`http://prowlarr:9696`)

> **Important:** Use container names (e.g. `radarr`, `sonarr`) rather than `localhost` when connecting apps within the same Docker network. All containers deployed via the same docker-compose.yml share a network and can reach each other by container name.

### Add Indexers

Go to **Indexers → Add** and search for your preferred torrent indexers. Prowlarr supports hundreds of public and private trackers.

For indexers protected by Cloudflare, you can use [Byparr](https://github.com/ThePhaseless/Byparr) as a FlareSolverr replacement:

```powershell
docker run -d --name byparr `
  --restart always `
  -p 8191:8191 `
  ghcr.io/thephaseless/byparr:latest
```

Connect Byparr to the arr Docker network:

```powershell
docker network connect arr_default byparr
```

Then in Prowlarr go to **Settings → Indexers → Add FlareSolverr** and set the URL to `http://byparr:8191`. Tag Cloudflare-protected indexers with the `byparr` tag.

### Recommended Public Indexers

| Indexer | Needs Byparr | Account Required |
|---|---|---|
| 1337x | Yes | No |
| Kickass Torrents | Yes | No |
| Rutracker | No | Free registration |
| Knaben | No | No |
| AudioBookBay | No | No |

---

## Radarr Setup

Radarr automates movie downloads. Go to `http://localhost:7878`.

### Root Folder

**Settings → Media Management → Root Folders → Add:** `/movies`

### Download Client

**Settings → Download Clients → Add → qBittorrent:**

| Field | Value |
|---|---|
| Host | `decypharr` |
| Port | `8282` |
| Username | your Decypharr username |
| Password | your Decypharr password |
| Category | `radarr` |

### Remote Path Mapping

**Settings → Download Clients → Remote Path Mappings → Add:**

| Field | Value |
|---|---|
| Host | `decypharr` |
| Remote Path | `/downloads/radarr` |
| Local Path | `/downloads/radarr` |

### Quality Profile for x265

Create a custom quality profile that prioritises x265/HEVC encodes to minimise file sizes without sacrificing quality.

**Settings → Custom Formats → Import** the following:

**x265 Format:**
```json
{
  "name": "x265",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x265",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": { "value": "x265|h265|HEVC" }
    }
  ]
}
```

**Reject x264 (too large):**
```json
{
  "name": "x264-Reject",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x264 or H264",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": { "value": "(?i)\\b(x264|h264|AVC)\\b" }
    }
  ]
}
```

**Size limits:**
```json
{
  "name": "Under 4GB",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Under 4GB",
      "implementation": "SizeSpecification",
      "negate": false,
      "required": true,
      "fields": { "min": 0, "max": 4000 }
    }
  ]
}
```

**In your quality profile set scores:**

| Custom Format | Score |
|---|---|
| x265 | +10 |
| Under 4GB | +5 |
| x264-Reject | -10000 |

Set **Minimum Custom Format Score** to `10` to ensure only x265 releases are grabbed.

---

## Sonarr Setup

Sonarr automates TV show downloads. Go to `http://localhost:8989`.

### Root Folder

**Settings → Media Management → Root Folders → Add:** `/tv`

### Download Client

Same as Radarr but set **Category** to `sonarr`.

### Remote Path Mapping

Same as Radarr but set paths to `/downloads/sonarr`.

### Quality Profile for TV

Import these custom formats for preferred release groups:

**PSA Release Group:**
```json
{
  "name": "PSA",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "PSA",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bPSA\\b" }
    }
  ]
}
```

**Megusta Release Group:**
```json
{
  "name": "Megusta",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Megusta",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bmegusta\\b" }
    }
  ]
}
```

**DVDRip/VHSRip for older content:**
```json
{
  "name": "DVDRip-VHSRip",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "DVDRip or VHSRip",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "(?i)\\b(DVDRip|VHSRip|VHS)\\b" }
    }
  ]
}
```

**Recommended scores:**

| Custom Format | Score |
|---|---|
| PSA | +15 |
| Megusta | +15 |
| x265 | +10 |
| DVDRip-VHSRip | +5 |
| x264-Reject | -10000 |
| Under 1GB per episode | +5 |

Set episode size hard limit to **1000 MB** per episode in quality settings.

---

## Lidarr Setup

Lidarr automates music downloads. Go to `http://localhost:8686`.

> **Note:** Use the **nightly** Lidarr build (`ghcr.io/hotio/lidarr:nightly`) for plugin support including Last.fm integration.

### Root Folder

**Settings → Media Management → Root Folders → Add:** `/music`

### Download Client

Same as Radarr/Sonarr but set **Category** to `lidarr`.

### Quality Profile

Prioritise lossless audio:

| Quality | Priority |
|---|---|
| FLAC | 1st |
| MP3-320 | 2nd |

Import custom formats:

**FLAC:**
```json
{
  "name": "FLAC",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "FLAC",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bFLAC\\b" }
    }
  ]
}
```

**MP3 320:**
```json
{
  "name": "MP3 320",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "MP3 320",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "320" }
    }
  ]
}
```

### Last.fm Import List

To automatically add artists based on your listening history:

1. Create a free [Last.fm](https://last.fm) account
2. Get a free API key at [last.fm/api/account/create](https://www.last.fm/api/account/create)
3. In Lidarr go to **Settings → Import Lists → Add → Last.fm User**
4. Enter your Last.fm username and API key
5. Select **Top Artists** and/or **Loved Tracks**
6. Set sync interval to 12 hours

Scrobble your listening activity to Last.fm using:
- **Pano Scrobbler** on Android (supports YouTube Music, Spotify, and more)
- **Web Scrobbler** browser extension on desktop

---

## Pulsarr Setup

Pulsarr monitors your Plex watchlist and automatically adds movies and TV shows to Radarr and Sonarr when you add them to your watchlist in Plex.

Go to `http://localhost:3003`.

### Connect Radarr

**Settings → Radarr → Add Instance:**

| Field | Value |
|---|---|
| URL | `http://radarr:7878` |
| API Key | Your Radarr API key |
| Root Folder | `/movies` |
| Quality Profile | Your x265 profile |

### Connect Sonarr

**Settings → Sonarr → Add Instance:**

| Field | Value |
|---|---|
| URL | `http://sonarr:8989` |
| API Key | Your Sonarr API key |
| Root Folder | `/tv` |
| Quality Profile | Your x265 TV profile |

### Connect Plex

**Settings → Plex:**

Enter your Plex token to connect Pulsarr to your Plex server. Your Plex token can be found in Plex web app → Account → Account Settings → Privacy → under any XML file URL as `X-Plex-Token`.

Once connected, Pulsarr will:
- Monitor your Plex watchlist continuously
- Automatically add movies to Radarr when added to your watchlist
- Automatically add TV shows to Sonarr when added to your watchlist
- Remove them from the watchlist once downloaded and available in Plex

> **Important:** When configuring Pulsarr's connections to Radarr and Sonarr, use the container names (`http://radarr:7878`) not `localhost`. All apps in the same Docker Compose stack share a network and communicate by container name.

---

## Plex Integration

### How the Full Flow Works

```
Plex Watchlist
      ↓
   Pulsarr (monitors watchlist every few minutes)
      ↓
Radarr / Sonarr (adds to wanted list)
      ↓
   Prowlarr (searches configured indexers)
      ↓
   Decypharr (submits torrent to TorBox)
      ↓
   TorBox (caches and serves the file)
      ↓
Decypharr (downloads from TorBox to /downloads)
      ↓
Radarr / Sonarr (imports to /movies or /tv)
      ↓
   Plex (scans library and makes available)
      ↓
 Pulsarr (removes from watchlist)
```

### Plex Library Setup

In Plex, configure your libraries to point to the same folders Docker maps your media to:

| Library | Folder |
|---|---|
| Movies | `G:\Movies` (and any additional movie drives) |
| TV Shows | `G:\TV` (and any additional TV drives) |
| Music | `E:\Music` |

Enable **automatic library scanning** in Plex so new content appears as soon as it is imported by the arr apps.

---

## Networking Notes

All containers deployed in the same `docker-compose.yml` file are automatically placed on a shared Docker network (named `arr_default` by default). This means:

- Containers can reach each other using their **container name** as the hostname
- `http://radarr:7878` works from any other container in the stack
- `localhost` inside a container refers to that container only, not the host machine
- To reach the host machine from a container use `host.docker.internal`
- To reach containers from your browser use your server's local IP (e.g. `http://192.168.x.x:7878`)

---

## Windows Firewall Rules

Open **PowerShell as Administrator** and run:

```powershell
New-NetFirewallRule -DisplayName "Radarr" -Direction Inbound -Protocol TCP -LocalPort 7878 -Action Allow
New-NetFirewallRule -DisplayName "Sonarr" -Direction Inbound -Protocol TCP -LocalPort 8989 -Action Allow
New-NetFirewallRule -DisplayName "Lidarr" -Direction Inbound -Protocol TCP -LocalPort 8686 -Action Allow
New-NetFirewallRule -DisplayName "Prowlarr" -Direction Inbound -Protocol TCP -LocalPort 9696 -Action Allow
New-NetFirewallRule -DisplayName "Decypharr" -Direction Inbound -Protocol TCP -LocalPort 8282 -Action Allow
New-NetFirewallRule -DisplayName "Pulsarr" -Direction Inbound -Protocol TCP -LocalPort 3003 -Action Allow
```

---

## Remote Access

For secure remote access to your stack without opening ports on your router, [Tailscale](https://tailscale.com) is recommended. Install it on your server and any devices you want to access the stack from. All containers will be accessible via your server's Tailscale IP using the same ports as above.

---

## Troubleshooting

### Containers Can't Reach Each Other
Make sure you are using container names (e.g. `http://radarr:7878`) not `localhost` when configuring connections between apps. All apps in the same Compose stack share a network.

### Drive Volumes Not Mounting
Ensure drives are added to Docker Desktop **File Sharing** (Settings → Resources → File Sharing) and use the `//driveletter/path` format in docker-compose.yml for non-C drives (e.g. `//g/Movies`).

### Download Client Path Errors
The arr apps and Decypharr must all agree on the download path. Set Remote Path Mappings in each arr app to map `/downloads/appname` to `/downloads/appname`. Make sure subfolder names match exactly including case.

### WSL2 Using Too Much RAM
Add a `.wslconfig` file at `C:\Users\USERNAME\.wslconfig` with a memory cap as described in the WSL2 setup section above.

---

---

## TorBoxarr — Usenet Downloads

TorBoxarr is an alternative to Decypharr that supports both torrents and **Usenet (NZB) downloads** through TorBox. It emulates two API surfaces — the qBittorrent Web API for torrent integration and the SABnzbd API for Usenet integration — allowing your arr apps to use TorBox as a unified download backend for both sources.

If you have a TorBox plan that includes Usenet access, TorBoxarr lets you add Usenet indexers in Prowlarr and have NZB files downloaded through TorBox's servers the same way torrents are — no separate Usenet provider subscription required beyond TorBox itself.

### Why Use TorBoxarr Instead of (or Alongside) Decypharr?

| Feature | Decypharr | TorBoxarr |
|---|---|---|
| Torrent support | ✅ | ✅ |
| Usenet/NZB support | ❌ | ✅ |
| Multiple debrid providers | ✅ | TorBox only |
| SABnzbd emulation | ❌ | ✅ |

If you want Usenet support, use TorBoxarr. If you only need torrents across multiple debrid services, Decypharr is the better choice. You can run both simultaneously if needed.

### Add TorBoxarr to docker-compose.yml

Create the config folder first:

```powershell
mkdir C:\docker\arr\torboxarr
```

Add this service to your `C:\docker\arr\docker-compose.yml`:

```yaml
  torboxarr:
    image: ghcr.io/mrjoiny/torboxarr:latest
    container_name: torboxarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=*TYPE_YOUR_COUNTRY*/*TYPE_YOUR_CITY*
      - TORBOXARR_API_KEY=YOUR_TORBOX_API_KEY
      - TORBOXARR_DATA_ROOT=/downloads
      - TORBOXARR_LISTEN_ADDR=0.0.0.0:8085
    volumes:
      - C:\docker\arr\torboxarr:/config
      - //g/Downloads/Completed:/downloads
    ports:
      - 8085:8085
    restart: unless-stopped
```

Deploy it:

```powershell
cd C:\docker\arr
docker compose up -d torboxarr
```

### Add Usenet Indexers in Prowlarr

Prowlarr supports Usenet indexers (Newznab compatible) alongside torrent indexers. Go to `http://localhost:9696` → **Indexers → Add** and search for Usenet indexers.

Popular free and paid Usenet indexers compatible with Prowlarr:

| Indexer | Type | Notes |
|---|---|---|
| NZBGeek | Paid | Reliable, good coverage |
| DrunkenSlug | Free/Invite | Good free option |
| NZBFinder | Free tier | Limited free searches |
| Althub | Free | Open registration |
| NZBIndex | Free | Basic but functional |

Add your chosen indexers to Prowlarr — they will automatically sync to your arr apps alongside your torrent indexers.

### Connect TorBoxarr as SABnzbd Download Client

Setup is identical to the qBittorrent API connection. In your arr apps, add a SABnzbd downloader using the same IP address and port as TorBoxarr.

In each arr app go to **Settings → Download Clients → Add → SABnzbd**:

| Field | Value |
|---|---|
| Host | `torboxarr` |
| Port | `8085` |
| API Key | Your TorBox API key |
| Category | `radarr` / `sonarr` / `lidarr` |

Test and Save in each app.

### Download Folder Setup for Usenet

TorBoxarr categories map to subfolders under the data root. Create category folders manually if they don't already exist.

Your existing download folders work perfectly:

```
G:\Downloads\Completed\
├── radarr\     ← used by both torrent and Usenet downloads
├── sonarr\     ← used by both torrent and Usenet downloads
└── lidarr\     ← used by both torrent and Usenet downloads
```

### Remote Path Mappings for TorBoxarr

Add Remote Path Mappings in each arr app for TorBoxarr the same way you did for Decypharr:

**Settings → Download Clients → Remote Path Mappings → Add:**

| Host | Remote Path | Local Path |
|---|---|---|
| `torboxarr` | `/downloads/radarr` | `/downloads/radarr` |
| `torboxarr` | `/downloads/sonarr` | `/downloads/sonarr` |
| `torboxarr` | `/downloads/lidarr` | `/downloads/lidarr` |

### How Usenet Downloads Flow

```
Prowlarr finds NZB via Usenet indexer
              ↓
    Arr app sends NZB to TorBoxarr
    (via SABnzbd API on port 8085)
              ↓
  TorBoxarr submits NZB to TorBox API
              ↓
   TorBox downloads from Usenet servers
              ↓
 TorBoxarr downloads completed file to
         /downloads/category
              ↓
  Arr app imports to /movies or /tv
```

### Firewall Rule

```powershell
New-NetFirewallRule -DisplayName "TorBoxarr" -Direction Inbound -Protocol TCP -LocalPort 8085 -Action Allow
```

---

## Credits

- [Radarr](https://radarr.video)
- [Sonarr](https://sonarr.tv)
- [Lidarr](https://lidarr.audio)
- [Prowlarr](https://github.com/Prowlarr/Prowlarr)
- [Decypharr](https://github.com/sirrobot01/decypharr)
- [Pulsarr](https://github.com/jamcalli/Pulsarr)
- [TorBox](https://torbox.app)
- [LinuxServer.io Docker Images](https://linuxserver.io)

