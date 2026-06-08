# 🏠 Home Server — Self-Hosted Media & NAS Stack

> A fully self-hosted home server running on recycled hardware, accessible from anywhere in the world via Tailscale VPN.

-----

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Architecture](#architecture)
- [Services](#services)
- [Network & Security](#network--security)
- [Storage Strategy](#storage-strategy)
- [Installation](#installation)
- [Backup Strategy](#backup-strategy)

-----

## Overview

This project turns an old laptop into a fully functional home server with:

- 📸 **Private photo library** (Google Photos replacement)
- 🎵 **Self-hosted music streaming** (Spotify replacement)
- 🎬 **Personal media server** (Netflix-like experience)
- 📁 **Personal cloud storage** (Dropbox/Google Drive replacement)
- 🔒 **Secure remote access** from anywhere in the world

All services run in Docker containers and are accessible remotely through Tailscale VPN — no open ports, no exposed IPs.

-----

## Hardware

|Component            |Spec                            |
|---------------------|--------------------------------|
|**CPU**              |Intel Core i5 8th Gen           |
|**RAM**              |8 GB                            |
|**OS Drive**         |1 TB SSD                        |
|**Primary Storage**  |1 TB HDD (HGST — healthy)       |
|**Secondary Storage**|2 TB HDD (media only — degraded)|
|**OS**               |Debian 13 (Trixie) — headless   |

-----

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   HOME SERVER                        │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Immich  │  │Navidrome │  │    Jellyfin      │  │
│  │  :2283   │  │  :4533   │  │     :8096        │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │Nextcloud │  │  Radarr  │  │     Sonarr       │  │
│  │  :8080   │  │  :7878   │  │     :8989        │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
│  ┌──────────┐  ┌──────────┐                         │
│  │Prowlarr  │  │qBittorrt │                         │
│  │  :9696   │  │  :8090   │                         │
│  └──────────┘  └──────────┘                         │
│                                                      │
│              [Tailscale VPN]                         │
└─────────────────────────────────────────────────────┘
         │
         │  Tailscale tunnel (no open ports)
         │
┌────────┴────────────────────────────────────────────┐
│          ANY DEVICE — ANYWHERE IN THE WORLD          │
│   iPhone · MacBook · Windows PC · Android            │
└─────────────────────────────────────────────────────┘
```

-----

## Services

|Service                               |Port|Purpose                       |Mobile App         |
|--------------------------------------|----|------------------------------|-------------------|
|[Immich](https://immich.app)          |2283|Photo library & backup        |✅ Official app     |
|[Navidrome](https://navidrome.org)    |4533|Music streaming (Subsonic API)|✅ Substreamer (iOS)|
|[Jellyfin](https://jellyfin.org)      |8096|Movies & TV streaming         |✅ Official app     |
|[Nextcloud](https://nextcloud.com)    |8080|File storage & sync           |✅ Official app     |
|[Radarr](https://radarr.video)        |7878|Movie management              |—                  |
|[Sonarr](https://sonarr.tv)           |8989|TV show management            |—                  |
|[Prowlarr](https://prowlarr.com)      |9696|Indexer management            |—                  |
|[qBittorrent](https://qbittorrent.org)|8090|Download client               |—                  |

-----

## Network & Security

- **VPN**: [Tailscale](https://tailscale.com) — WireGuard-based mesh VPN
- **No open ports** on the public internet
- **Firewall**: UFW with default deny incoming
- **Only Tailscale interface** (`tailscale0`) allowed for all services
- **Fail2ban** for brute-force protection on SSH

```
Internet ──✗──> Server (no open ports)
 
Your Device ──[Tailscale VPN]──> Server ──> All Services
```

-----

## Storage Strategy

```
/
├── SSD 1TB          → OS + Docker + Databases
│
/mnt/
├── datos/           → HDD 1TB (HGST — primary)
│   ├── fotos/       → Immich photo library
│   └── musica/      → Navidrome music library
│
└── media/           → HDD 2TB (secondary — media only)
    ├── peliculas/   → Jellyfin movies
    ├── series/      → Jellyfin TV shows
    ├── anime/       → Jellyfin anime
    ├── descargas/   → qBittorrent downloads
    └── backup_datos/→ nightly rsync backup of /datos
```

> ⚠️ The 2TB drive has reallocated sectors — used only for replaceable media content, not critical data.

-----

## Installation

### Prerequisites

- Debian 13 minimal install (headless)
- Internet connection
- Docker & Docker Compose

### Quick Start

```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Install Docker
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo usermod -aG docker $USER

# 3. Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# 4. Configure firewall
sudo apt install ufw fail2ban -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow 22/tcp
sudo ufw enable

# 5. Create folder structure
sudo mkdir -p /mnt/datos/fotos /mnt/datos/musica
sudo mkdir -p /mnt/media/peliculas /mnt/media/series /mnt/media/anime /mnt/media/descargas
sudo mkdir -p /opt/docker/{immich,navidrome,jellyfin,nextcloud,arr}
sudo chown -R $USER:$USER /mnt/datos /mnt/media /opt/docker
```

### Deploy Services

Each service has its own `docker-compose.yml` under `/opt/docker/<service>/`.
See the `/docs` folder for individual compose files.

-----

## Backup Strategy

Automated nightly backup via cron (runs at 2:00 AM):

```bash
# /etc/crontab entry
0 2 * * * rsync -av --delete /mnt/datos/ /mnt/media/backup_datos/ >> /var/log/rsync_backup.log 2>&1
```

This mirrors all critical data (photos + music) from the healthy 1TB HDD to the 2TB HDD nightly.

-----

## Music Downloads

yt-dlp is used to download music with full metadata and embedded thumbnails:

```bash
yt-dlp -x --audio-format mp3 \
  --embed-thumbnail \
  --embed-metadata \
  --add-metadata \
  -o "/mnt/datos/musica/%(title)s.%(ext)s" \
  "YOUTUBE_URL"
```

Navidrome scans the music library every hour automatically.

-----

## License

This project is for personal, educational use only.