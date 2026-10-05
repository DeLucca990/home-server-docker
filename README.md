# Ubuntu Server — Docker Services

### Services

#### Media Stack

| Service                                     | Description                         |   Port |
| ------------------------------------------- | ----------------------------------- | -----: |
| [Jellyfin](https://jellyfin.org/)           | Media streaming and management      | `8096` |
| [qBittorrent](https://www.qbittorrent.org/) | Torrent client                      | `8080` |
| [Prowlarr](https://prowlarr.com/)           | Indexer manager                     | `9696` |
| [Radarr](https://radarr.video/)             | Movie management and automation     | `7878` |
| [Sonarr](https://sonarr.tv/)                | TV series management and automation | `8989` |
| [Bazarr](https://www.bazarr.media/)         | Subtitle management and automation  | `6767` |
| [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) | Cloudflare challenge solver for Prowlarr | `8191` (localhost only) |

#### Utils Stack

| Service                                                | Description                     |   Port |
| ------------------------------------------------------ | ------------------------------- | -----: |
| [Portainer](https://www.portainer.io/)                 | Docker container management     | `9443` |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Service availability monitoring | `3001` |

## Repository Structure

```text
.
├── compose-medias.yml
├── compose-utils.yml
└── config/
    ├── jellyfin/
    ├── qbittorrent/
    ├── prowlarr/
    ├── radarr/
    ├── sonarr/
    ├── bazarr/
    └── uptime-kuma/
```

Persistent application data is stored under the `config/` directory.

Media files are stored outside this repository:

```text
/home/ubuntu/media/
├── downloads/
├── movies/
└── series/
```

## Media Storage

All media-related containers use the same host directory:

```text
/home/ubuntu/media → /media
```

This consistent path is intentional and allows Radarr, Sonarr, qBittorrent, Bazarr, and Jellyfin to reference the same files using the same container path.

It also allows the use of **hardlinks** between downloaded files and the final media library without unnecessarily duplicating the file data on disk.

For example:

```text
/media/downloads/movies/movie.mkv
/media/movies/Movie (2026)/Movie (2026).mkv
```

Both paths can reference the same physical file through a hardlink.

### Root Folders

The recommended root folders inside the containers are:

```text
Radarr → /media/movies
Sonarr → /media/series
Jellyfin → /media/movies
Jellyfin → /media/series
```

## Network

The two Compose files are independent stacks and therefore create separate Docker networks.

Because of this, services from different stacks should not rely on Docker service-name resolution.

For example, Uptime Kuma should monitor Jellyfin using:

```text
http://SERVER_IP:8096
```

rather than:

```text
http://jellyfin:8096
```

Services within the same Compose stack can communicate using their Docker service names.

## Starting the Services

### Media Stack

Start all media services:

```bash
docker compose -f compose-medias.yml up -d
```

Stop the media stack:

```bash
docker compose -f compose-medias.yml down
```

Restart the media stack:

```bash
docker compose -f compose-medias.yml restart
```

### Utils Stack

Start Portainer and Uptime Kuma:

```bash
docker compose -f compose-utils.yml up -d
```

Stop the utils stack:

```bash
docker compose -f compose-utils.yml down
```

Restart the utils stack:

```bash
docker compose -f compose-utils.yml restart
```

## Updating Images

Pull the latest images:

```bash
docker compose -f compose-medias.yml pull
docker compose -f compose-utils.yml pull
```

Recreate containers using the updated images:

```bash
docker compose -f compose-medias.yml up -d
docker compose -f compose-utils.yml up -d
```

Unused images can be removed periodically with:

```bash
docker image prune
```

## Checking Container Status

List running containers:

```bash
docker ps
```

View all containers, including stopped ones:

```bash
docker ps -a
```

View logs for a specific service:

```bash
docker logs jellyfin
```

Follow logs in real time:

```bash
docker logs -f jellyfin
```

## Access

Replace `SERVER_IP` with the local IP address of the Ubuntu Server.

### Media

```text
Jellyfin       http://SERVER_IP:8096
qBittorrent    http://SERVER_IP:8080
Prowlarr       http://SERVER_IP:9696
Radarr         http://SERVER_IP:7878
Sonarr         http://SERVER_IP:8989
Bazarr         http://SERVER_IP:6767
```

### Utils

```text
Portainer      https://SERVER_IP:9443
Uptime Kuma    http://SERVER_IP:3001
```

Portainer also exposes port `9000`, although HTTPS on port `9443` is recommended.

## Portainer

Portainer is connected directly to the Docker Engine through:

```text
/var/run/docker.sock
```

This allows it to manage containers, images, volumes, networks, and Compose stacks running on the server.

## Uptime Kuma

Uptime Kuma is used to monitor the availability of the services running on the server.

Recommended monitors:

```text
Jellyfin       → http://SERVER_IP:8096
qBittorrent    → http://SERVER_IP:8080
Prowlarr       → http://SERVER_IP:9696
Radarr         → http://SERVER_IP:7878
Sonarr         → http://SERVER_IP:8989
Bazarr         → http://SERVER_IP:6767
```

A heartbeat interval of 60 seconds is a reasonable starting point.

## Restart Policy

All services use:

```yaml
restart: unless-stopped
```

This ensures containers are automatically restarted after a Docker daemon restart or server reboot, unless they were explicitly stopped.

## Backup

The `config/` directory contains persistent application configuration and should be included in backups.

The media library is stored separately under:

```text
/home/ubuntu/media
```

At minimum, back up:

```text
config/
```

and any important Docker Compose files.

Do not rely on container images as backups. Containers can be recreated from the Compose files, while application configuration and data need to be preserved separately.
