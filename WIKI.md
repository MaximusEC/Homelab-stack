# 📖 Wiki — Servidor Casero Self-Hosted

Documentación completa en español del proyecto de servidor casero.

-----

## Índice

1. [Preparación del hardware](#1-preparación-del-hardware)
1. [Instalación de Debian 13](#2-instalación-de-debian-13)
1. [Configuración base del sistema](#3-configuración-base-del-sistema)
1. [Montaje de discos](#4-montaje-de-discos)
1. [Docker](#5-docker)
1. [Tailscale VPN](#6-tailscale-vpn)
1. [Firewall UFW](#7-firewall-ufw)
1. [Servicios — Docker Compose](#8-servicios--docker-compose)
1. [Acceso desde el celular](#9-acceso-desde-el-celular)
1. [Descargas automáticas](#10-descargas-automáticas)
1. [Respaldo automático](#11-respaldo-automático)
1. [Mantenimiento](#12-mantenimiento)

-----

## 1. Preparación del hardware

Antes de instalar nada, verifica el estado de los discos con **CrystalDiskInfo** (Windows).

Los atributos críticos a revisar:

|ID|Nombre                 |Qué significa si está alto|
|--|-----------------------|--------------------------|
|05|Reallocated Sectors    |Sectores físicos muertos  |
|C5|Current Pending Sectors|Sectores a punto de fallar|
|C6|Uncorrectable Sectors  |Sectores irrecuperables   |

**Regla general:**

- Disco sano → úsalo para datos críticos (fotos, música)
- Disco con sectores reasignados → úsalo solo para contenido reemplazable (películas)

-----

## 2. Instalación de Debian 13

### Descargar la ISO

Descarga Debian 13 netinstall desde el sitio oficial de Debian.

### Grabar el USB con Rufus

Configuración en Rufus:

|Campo               |Valor         |
|--------------------|--------------|
|Esquema de partición|GPT           |
|Sistema destino     |UEFI (non CSM)|
|Sistema de archivos |FAT32         |
|Modo de escritura   |DD            |

### Selección de paquetes

En la pantalla “Selección de programas”, deja **solo** marcado:

- ✅ SSH server
- ✅ Utilidades estándar del sistema

Desmarca todo lo demás — no necesitas interfaz gráfica.

-----

## 3. Configuración base del sistema

### Instalar sudo

Debian no incluye sudo por defecto:

```bash
su -
apt install sudo -y && usermod -aG sudo TU_USUARIO && exit
exit  # cerrar sesión y volver a entrar
```

### Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

-----

## 4. Montaje de discos

### Identificar los discos

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

### Formatear los discos externos

```bash
# Disco de datos primario (el sano)
sudo mkfs.ext4 -L "datos" /dev/sdX

# Disco de media (el secundario)
sudo mkfs.ext4 -L "media" /dev/sdY
```

### Crear puntos de montaje

```bash
sudo mkdir -p /mnt/datos /mnt/media
```

### Montaje permanente via fstab

Obtén los UUIDs:

```bash
sudo blkid /dev/sdX && sudo blkid /dev/sdY
```

Agrega al `/etc/fstab`:

```bash
echo 'UUID=TU-UUID-DATOS /mnt/datos ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
echo 'UUID=TU-UUID-MEDIA /mnt/media ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a && sudo systemctl daemon-reload
```

### Estructura de carpetas

```bash
sudo mkdir -p /mnt/datos/fotos /mnt/datos/musica
sudo mkdir -p /mnt/media/peliculas /mnt/media/series /mnt/media/anime /mnt/media/descargas
sudo mkdir -p /opt/docker/{immich,navidrome,jellyfin,nextcloud,arr}
sudo chown -R $USER:$USER /mnt/datos /mnt/media /opt/docker
```

-----

## 5. Docker

### Instalación

```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo usermod -aG docker $USER
```

Cierra sesión y vuelve a entrar para aplicar el grupo docker.

### Verificar instalación

```bash
docker --version && docker compose version
```

-----

## 6. Tailscale VPN

### Instalación y autenticación

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Abre el enlace que aparece en el navegador para autenticar el servidor.

### Verificar IP de Tailscale

```bash
tailscale ip
```

Anota esta IP — es la que usarás para acceder a todos los servicios.

-----

## 7. Firewall UFW

```bash
sudo apt install ufw fail2ban -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow 22/tcp
sudo ufw enable
```

### Abrir puertos para los servicios

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

## 8. Servicios — Docker Compose

### Immich (fotos)

```bash
cd /opt/docker/immich
curl -o docker-compose.yml https://raw.githubusercontent.com/immich-app/immich/main/docker/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/immich-app/immich/main/docker/example.env
```

Edita `.env` y cambia:

```
UPLOAD_LOCATION=/mnt/datos/fotos
```

```bash
docker compose up -d
```

### Navidrome (música)

```yaml
# /opt/docker/navidrome/docker-compose.yml
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
    volumes:
      - ./data:/data
      - /mnt/datos/musica:/music:ro
```

### Jellyfin (películas)

```yaml
# /opt/docker/jellyfin/docker-compose.yml
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

### Nextcloud (NAS)

```yaml
# /opt/docker/nextcloud/docker-compose.yml
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
      - NEXTCLOUD_ADMIN_USER=TU_USUARIO
      - NEXTCLOUD_ADMIN_PASSWORD=TU_CONTRASEÑA
      - NEXTCLOUD_TRUSTED_DOMAINS=TU_IP_TAILSCALE
    depends_on:
      - db
  db:
    image: mariadb:10.11
    container_name: nextcloud_db
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=TU_ROOT_PASS
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=TU_DB_PASS
    volumes:
      - ./db:/var/lib/mysql
```

### Stack de descargas (Radarr + Sonarr + Prowlarr + qBittorrent)

Ver `/opt/docker/arr/docker-compose.yml` en el repositorio.

-----

## 9. Acceso desde el celular

### Requisitos

1. Instala **Tailscale** en tu celular (App Store / Play Store)
1. Inicia sesión con la misma cuenta que usaste en el servidor
1. Activa la conexión

### Apps recomendadas (iOS)

|Servicio |App                        |
|---------|---------------------------|
|Immich   |Immich (oficial)           |
|Navidrome|Substreamer o play:Sub     |
|Jellyfin |Jellyfin (oficial) o Infuse|
|Nextcloud|Nextcloud (oficial)        |

### URLs de acceso

Reemplaza `TU_IP_TAILSCALE` con tu IP de Tailscale:

```
Immich:     http://TU_IP_TAILSCALE:2283
Navidrome:  http://TU_IP_TAILSCALE:4533
Jellyfin:   http://TU_IP_TAILSCALE:8096
Nextcloud:  http://TU_IP_TAILSCALE:8080
```

-----

## 10. Descargas automáticas

### Películas con Radarr

1. Abre Radarr → **Movies → Add New Movie**
1. Busca la película
1. Selecciona calidad **HD-1080p**
1. Root folder: `/movies`
1. **Add Movie → Search**

qBittorrent descarga automáticamente → Radarr mueve a `/mnt/media/peliculas` → Jellyfin detecta y agrega con metadata.

### Música con yt-dlp

```bash
# Instalar versión actualizada
sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
sudo chmod a+rx /usr/local/bin/yt-dlp

# Descargar canción con metadata completa
yt-dlp -x --audio-format mp3 \
  --embed-thumbnail \
  --embed-metadata \
  --add-metadata \
  -o "/mnt/datos/musica/%(title)s.%(ext)s" \
  "URL_DE_YOUTUBE"
```

-----

## 11. Respaldo automático

Backup nocturno automático a las 2:00 AM:

```bash
sudo crontab -e
```

Agrega:

```
0 2 * * * rsync -av --delete /mnt/datos/ /mnt/media/backup_datos/ >> /var/log/rsync_backup.log 2>&1
```

Verifica el log:

```bash
cat /var/log/rsync_backup.log
```

-----

## 12. Mantenimiento

### Ver estado de todos los contenedores

```bash
docker ps
```

### Reiniciar un servicio

```bash
cd /opt/docker/SERVICIO && docker compose restart
```

### Actualizar todos los contenedores

```bash
cd /opt/docker/immich && docker compose pull && docker compose up -d
cd /opt/docker/navidrome && docker compose pull && docker compose up -d
cd /opt/docker/jellyfin && docker compose pull && docker compose up -d
cd /opt/docker/nextcloud && docker compose pull && docker compose up -d
cd /opt/docker/arr && docker compose pull && docker compose up -d
```

### Ver uso de disco

```bash
df -h | grep mnt
```

### Ver uso de RAM

```bash
free -h
```

-----

*Documentación generada para uso personal. No incluye IPs ni credenciales reales.*