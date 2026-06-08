# 🏠 homelab-stack

> Self-hosted home server built on recycled hardware running Debian 13. Features private photo library, music streaming, media server and personal cloud — all containerized with Docker and securely accessible from anywhere via Tailscale VPN. | Servidor casero self-hosted con Docker, accesible globalmente vía Tailscale.

-----

## 📋 Table of Contents

- [Hardware](#hardware)
- [Architecture](#architecture)
- [Services](#services)
- [Network & Security](#network--security)
- [Storage Layout](#storage-layout)
- [Step 1 — Debian 13 Install](#step-1--debian-13-install)
- [Step 2 — Base System Config](#step-2--base-system-config)
- [Step 3 — Mount Disks](#step-3--mount-disks)
- [Step 4 — Install Docker](#step-4--install-docker)
- [Step 5 — Tailscale VPN](#step-5--tailscale-vpn)
- [Step 6 — Firewall](#step-6--firewall)
- [Step 7 — Deploy Services](#step-7--deploy-services)
- [Step 8 — Automated Downloads](#step-8--automated-downloads)
- [Step 9 — Backup](#step-9--backup)
- [Maintenance](#maintenance)

-----

## Hardware

|Component            |Spec                         |
|---------------------|-----------------------------|
|**CPU**              |Intel Core i5 8th Gen        |
|**RAM**              |8 GB                         |
|**OS Drive**         |1 TB SSD                     |
|**Primary Storage**  |1 TB HDD (healthy)           |
|**Secondary Storage**|2 TB HDD (media only)        |
|**OS**               |Debian 13 (Trixie) — headless|

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

- **VPN**: Tailscale — WireGuard-based mesh VPN, zero open ports
- **Firewall**: UFW with default deny incoming
- **Only Tailscale interface** (`tailscale0`) allowed for all services
- **Fail2ban** for SSH brute-force protection

```
Internet ──✗──> Server (no open ports)

Your Device ──[Tailscale VPN]──> Server ──> All Services
```

-----

## Storage Layout

```
/
├── SSD 1TB              → OS + Docker + Databases
│
/mnt/
├── datos/               → HDD 1TB (primary — healthy disk)
│   ├── fotos/           → Immich photo library
│   └── musica/          → Navidrome music library
│
└── media/               → HDD 2TB (secondary — media only)
    ├── peliculas/        → Jellyfin movies
    ├── series/           → Jellyfin TV shows
    ├── anime/            → Jellyfin anime
    ├── descargas/        → qBittorrent downloads
    └── backup_datos/     → nightly rsync backup of /datos
```

> ⚠️ Always check disk health with CrystalDiskInfo (Windows) or `smartctl` (Linux) before using recycled drives.

-----

## Step 1 — Debian 13 Install

Download the **netinstall ISO** from the official Debian website and flash it to a USB with **Rufus**:

|Rufus Setting   |Value         |
|----------------|--------------|
|Partition scheme|GPT           |
|Target system   |UEFI (non CSM)|
|File system     |FAT32         |
|Write mode      |DD            |

During package selection, leave **only** these checked:

- ✅ SSH server
- ✅ Standard system utilities

No desktop environment needed — this is a headless server.

-----

## Step 2 — Base System Config

Debian doesn’t include `sudo` by default:

```bash
su -
apt install sudo -y && usermod -aG sudo YOUR_USERNAME && exit
exit  # log out and back in
```

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

-----

## Step 3 — Mount Disks

Identify your disks:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

Format the external drives:

```bash
# Primary data disk
sudo mkfs.ext4 -L "datos" /dev/sdX

# Media disk
sudo mkfs.ext4 -L "media" /dev/sdY
```

Create mount points:

```bash
sudo mkdir -p /mnt/datos /mnt/media
```

Get UUIDs and add to fstab for permanent mounting:

```bash
sudo blkid /dev/sdX
sudo blkid /dev/sdY
```

```bash
echo 'UUID=YOUR-UUID-HERE /mnt/datos ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
echo 'UUID=YOUR-UUID-HERE /mnt/media ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a && sudo systemctl daemon-reload
```

Create folder structure:

```bash
sudo mkdir -p /mnt/datos/fotos /mnt/datos/musica
sudo mkdir -p /mnt/media/peliculas /mnt/media/series /mnt/media/anime /mnt/media/descargas
sudo mkdir -p /opt/docker/{immich,navidrome,jellyfin,nextcloud,arr}
sudo chown -R $USER:$USER /mnt/datos /mnt/media /opt/docker
```

-----

## Step 4 — Install Docker

```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo usermod -aG docker $USER
```

Log out and back in, then verify:

```bash
docker --version && docker compose version
```

-----

## Step 5 — Tailscale VPN

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Open the URL shown in the terminal to authenticate. Then get your Tailscale IP:

```bash
tailscale ip
# Example output: 100.x.x.x
```

> This IP is how you’ll access all services from anywhere in the world.

-----

## Step 6 — Firewall

```bash
sudo apt install ufw fail2ban -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow 22/tcp
sudo ufw enable
```

Open service ports:

```bash
sudo ufw allow 2283/tcp  # Immich
sudo ufw allow 4533/tcp  # Navidrome
sudo ufw allow 8096/tcp  # Jellyfin
sudo ufw allow 8080/tcp  # Nextcloud
sudo ufw allow 7878/tcp  # Radarr
sudo ufw allow 8989/tcp  # Sonarr
sudo ufw allow 9696/tcp  # Prowlarr
sudo ufw allow 8090/tcp  # qBittorrent
```

-----

## Step 7 — Deploy Services

### 📸 Immich

```bash
cd /opt/docker/immich
curl -o docker-compose.yml https://raw.githubusercontent.com/immich-app/immich/main/docker/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/immich-app/immich/main/docker/example.env
```

Edit `.env` and set:

```env
UPLOAD_LOCATION=/mnt/datos/fotos
DB_DATA_LOCATION=./postgres
```

```bash
docker compose up -d
```

Access: `http://TAILSCALE_IP:2283`

-----

### 🎵 Navidrome

```bash
cd /opt/docker/navidrome
nano docker-compose.yml
```

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped
    ports:
      - "4533:4533"
    environment:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info
      ND_SESSIONTIMEOUT: 24h
      ND_BASEURL: ""
    volumes:
      - ./data:/data
      - /mnt/datos/musica:/music:ro
```

```bash
docker compose up -d
```

Access: `http://TAILSCALE_IP:4533`

-----

### 🎬 Jellyfin

```bash
cd /opt/docker/jellyfin
nano docker-compose.yml
```

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/media/peliculas:/media/peliculas
      - /mnt/media/series:/media/series
      - /mnt/media/anime:/media/anime
```

```bash
docker compose up -d
```

Access: `http://TAILSCALE_IP:8096`

In Jellyfin → Dashboard → Libraries → Add Media Library:

- Type: Movies → Folder: `/media/peliculas`
- Type: Shows → Folder: `/media/series`

-----

### 📁 Nextcloud

```bash
cd /opt/docker/nextcloud
nano docker-compose.yml
```

```yaml
services:
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./data:/var/www/html
      - /mnt/datos:/mnt/datos
      - /mnt/media:/mnt/media
    environment:
      - NEXTCLOUD_ADMIN_USER=YOUR_ADMIN_USER
      - NEXTCLOUD_ADMIN_PASSWORD=YOUR_STRONG_PASSWORD
      - NEXTCLOUD_TRUSTED_DOMAINS=YOUR_TAILSCALE_IP
    depends_on:
      - db
  db:
    image: mariadb:10.11
    container_name: nextcloud_db
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=YOUR_ROOT_PASS
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=YOUR_DB_PASS
    volumes:
      - ./db:/var/lib/mysql
```

```bash
docker compose up -d
```

Access: `http://TAILSCALE_IP:8080`

> During first setup, select **MySQL/MariaDB** and use `nextcloud_db` as host.

-----

### 🔽 Download Stack (Radarr + Sonarr + Prowlarr + qBittorrent)

```bash
cd /opt/docker/arr
mkdir -p /mnt/media/descargas
nano docker-compose.yml
```

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/YOUR_TIMEZONE
      - WEBUI_PORT=8090
    ports:
      - "8090:8090"
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - ./qbittorrent/config:/config
      - /mnt/media/descargas:/downloads

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/YOUR_TIMEZONE
    ports:
      - "7878:7878"
    volumes:
      - ./radarr/config:/config
      - /mnt/media/peliculas:/movies
      - /mnt/media/descargas:/downloads

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/YOUR_TIMEZONE
    ports:
      - "8989:8989"
    volumes:
      - ./sonarr/config:/config
      - /mnt/media/series:/tv
      - /mnt/media/descargas:/downloads

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/YOUR_TIMEZONE
    ports:
      - "9696:9696"
    volumes:
      - ./prowlarr/config:/config
```

```bash
docker compose up -d
```

#### Connect qBittorrent to Radarr/Sonarr

1. Get qBittorrent temp password: `docker logs qbittorrent 2>&1 | grep -i password`
1. Login at `http://TAILSCALE_IP:8090` and change password in Tools → Preferences → Web UI
1. In Radarr → Settings → Download Clients → Add → qBittorrent:
- Host: `YOUR_LOCAL_IP`
- Port: `8090`
- Category: `radarr`
1. Repeat for Sonarr with category: `sonarr`

#### Add Root Folders

- Radarr → Settings → Media Management → Add Root Folder → `/movies`
- Sonarr → Settings → Media Management → Add Root Folder → `/tv`

#### Connect Prowlarr to Radarr/Sonarr

In Prowlarr → Settings → Apps → Add:

- Radarr: `http://YOUR_LOCAL_IP:7878` + API Key from Radarr → Settings → General
- Sonarr: `http://YOUR_LOCAL_IP:8989` + API Key from Sonarr → Settings → General

#### Add Indexers

In Prowlarr → Indexers → Add Indexer, recommended:

- YTS (movies)
- The Pirate Bay (general)
- Knaben (general)

Then: Settings → Apps → **Sync App Indexers**

-----

## Step 8 — Automated Downloads

### Adding a movie

1. Radarr → Movies → Add New Movie
1. Search for the title
1. Select quality: **HD-1080p** (recommended)
1. Root folder: `/movies`
1. Add Movie → Search

Flow: Radarr finds torrent → qBittorrent downloads → Radarr moves to `/mnt/media/peliculas` → Jellyfin auto-detects.

### Downloading music

Install latest yt-dlp:

```bash
sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
sudo chmod a+rx /usr/local/bin/yt-dlp
sudo apt install ffmpeg -y
```

Download a track with full metadata:

```bash
yt-dlp -x --audio-format mp3 \
  --embed-thumbnail \
  --embed-metadata \
  --add-metadata \
  -o "/mnt/datos/musica/%(title)s.%(ext)s" \
  "YOUTUBE_URL"
```

Navidrome scans the music folder every hour automatically.

-----

## Step 9 — Backup

Automated nightly rsync backup at 2:00 AM:

```bash
sudo crontab -e
```

Add:

```
0 2 * * * rsync -av --delete /mnt/datos/ /mnt/media/backup_datos/ >> /var/log/rsync_backup.log 2>&1
```

Create backup destination:

```bash
mkdir -p /mnt/media/backup_datos
```

Check backup logs:

```bash
cat /var/log/rsync_backup.log
```

-----

## Maintenance

### Check all containers

```bash
docker ps
```

### Restart a service

```bash
cd /opt/docker/SERVICE_NAME && docker compose restart
```

### Update all containers

```bash
for service in immich navidrome jellyfin nextcloud arr; do
  cd /opt/docker/$service && docker compose pull && docker compose up -d
done
```

### Check disk usage

```bash
df -h | grep mnt
```

### Check RAM usage

```bash
free -h
```

### View container logs

```bash
docker logs CONTAINER_NAME --tail 50
```

-----

## Mobile Access

Install **Tailscale** on your phone, sign in with the same account, and use these URLs:

|Service  |URL                       |
|---------|--------------------------|
|Immich   |`http://TAILSCALE_IP:2283`|
|Navidrome|`http://TAILSCALE_IP:4533`|
|Jellyfin |`http://TAILSCALE_IP:8096`|
|Nextcloud|`http://TAILSCALE_IP:8080`|

-----

*Built on recycled hardware. Zero cloud dependency. Full privacy.*