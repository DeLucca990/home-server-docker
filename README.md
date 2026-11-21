# Docker Media Services for Ubuntu 🐧

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

#### Observability Stack

| Service                                                | Description                     |   Port |
| ------------------------------------------------------ | ------------------------------- | -----: |
| [Portainer](https://www.portainer.io/)                 | Docker container management     | `9443` |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Service availability monitoring | `3001` |

## Repository Structure

```text
.
├── .env.example
├── compose-medias.yml
├── compose-observability.yml
├── config/
│   ├── jellyfin/
│   ├── qbittorrent/
│   ├── prowlarr/
│   ├── radarr/
│   ├── sonarr/
│   ├── bazarr/
│   └── uptime-kuma/
└── setup/
    ├── network/       # fixed LAN IP through a DHCP reservation
    ├── media/         # first-run configuration of each Web UI
    └── expose/        # remote access through Tailscale
```

Persistent application data is stored under the `config/` directory.

Media files are stored outside this repository:

```text
/home/ubuntu/media/
├── torrents/          # downloads (qBittorrent)
│   ├── movies/
│   ├── tv/
│   └── incomplete/
└── library/           # final library (Radarr/Sonarr write, Jellyfin reads)
    ├── movies/
    └── tv/
```

Create it with:

```bash
mkdir -p /home/ubuntu/media/{torrents/{movies,tv,incomplete},library/{movies,tv}}
sudo chown -R 1000:1000 /home/ubuntu/media
chmod -R 775 /home/ubuntu/media
```

## Environment Variables

The Compose files read these variables from `.env`. Only `LAN_IP` is required; the others have
defaults that match the paths used throughout this README:

| Variable | Default | Used for |
| -------- | ------- | -------- |
| `LAN_IP` | — (required, both stacks) | the server's LAN address the Web UIs are published on — see [Host binding](#host-binding) |
| `PUID` / `PGID` | `1000` | user and group the LinuxServer containers run as |
| `TZ` | `America/Sao_Paulo` | container time zone |
| `MEDIA_ROOT` | `/home/ubuntu/media` | host path of the media tree |

Before the first start, copy `.env.example` to `.env` next to the Compose files and set
`LAN_IP`; Compose loads `.env` automatically and refuses to start either stack while `LAN_IP` is
empty. If you change `PUID`, `PGID` or `MEDIA_ROOT`, adjust the `mkdir` and `chown` commands above
to match.

```bash
cp .env.example .env
```

## Setup Guides

| Guide | Covers |
| ----- | ------ |
| [Fixed LAN IP](setup/network/README.md) | Keeping the server's address fixed with a DHCP reservation on the router — do this before setting `LAN_IP` |
| [Media stack setup](setup/media/README.md) | First-run configuration of qBittorrent, Prowlarr, Radarr, Sonarr, Bazarr and Jellyfin, in order |
| [Remote access](setup/expose/README.md) | Reaching the server from outside the house through Tailscale, over HTTPS |

## Media Storage

Each container mounts only the part of `/home/ubuntu/media` it needs, but always under the same container path:

| Service     | Host path                     | Container path    |
| ----------- | ----------------------------- | ----------------- |
| Radarr      | `/home/ubuntu/media`          | `/media`          |
| Sonarr      | `/home/ubuntu/media`          | `/media`          |
| qBittorrent | `/home/ubuntu/media/torrents` | `/media/torrents` |
| Jellyfin    | `/home/ubuntu/media/library`  | `/media/library`  |
| Bazarr      | `/home/ubuntu/media/library`  | `/media/library`  |

Because the path strings match, a file qBittorrent saves as `/media/torrents/movies/x.mkv` has that exact path inside Radarr too, so no path mapping is needed anywhere in the stack.

Radarr and Sonarr mount the whole tree as a single `/media` volume, so `torrents/` and `library/` are on the same filesystem from their point of view. This allows them to import using **hardlinks** instead of copying the file data on disk.

For example:

```text
/media/torrents/movies/Movie.2026.1080p.BluRay.mkv
/media/library/movies/Movie (2026) [tmdbid-12345]/Movie (2026) [tmdbid-12345] - [Bluray-1080p].mkv
```

Both paths reference the same physical file through a hardlink.

### Root Folders

The root folders inside the containers are:

```text
Radarr               → /media/library/movies
Sonarr               → /media/library/tv
Jellyfin (Movies)    → /media/library/movies
Jellyfin (Shows)     → /media/library/tv
qBittorrent (radarr) → /media/torrents/movies
qBittorrent (sonarr) → /media/torrents/tv
```

Never point a Jellyfin library at `/media/library` as a whole; keep movies and TV as separate libraries.

See [setup/media/00-before-you-start.md](setup/media/00-before-you-start.md) for the full explanation.

## Host binding

The Web UIs that sit behind Tailscale Serve — the whole media stack and Uptime Kuma — bind their
published ports to both loopback and the LAN address, so local clients can connect directly while
Tailscale Serve uses the loopback listener without conflicting with Docker. The addresses are
defined at the top of both `compose-medias.yml` and `compose-observability.yml`, with the LAN
address taken from `LAN_IP` in `.env`:

```yaml
x-loopback-address: &loopback-address 127.0.0.1
x-lan-address: &lan-address ${LAN_IP:?set LAN_IP in .env - see .env.example}
```

Docker can only bind to an address the server actually has: if the router hands the server a
different IP, every container with a LAN port fails to start with `cannot assign requested
address`. Keep the address fixed with a DHCP reservation —
[setup/network](setup/network/README.md) walks through it.

Services can then be reached through Tailscale Serve or directly from the LAN.

Three ports are bound differently on purpose:

| Port | Binding | Why |
| ---- | ------- | --- |
| Portainer `9443` | every interface | terminates its own TLS and is not put behind Tailscale Serve, so the tailnet reaches it directly |
| qBittorrent `6881` | every interface | torrent port, reached by peers on the internet |
| FlareSolverr `8191` | loopback only | no authentication; Prowlarr uses the Compose network |

## Network

The two Compose files are independent stacks: the top-level `name:` in each file sets the Compose
project to `media` and `observability`, so they create separate Docker networks (`media_default`
and `observability_default`). Without it, both files would take their project name from the same
directory and end up as a single project, sharing one network and treating each other's
containers as orphans.

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

### Observability Stack

Start Portainer and Uptime Kuma:

```bash
docker compose -f compose-observability.yml up -d
```

Stop the observability stack:

```bash
docker compose -f compose-observability.yml down
```

Restart the observability stack:

```bash
docker compose -f compose-observability.yml restart
```

## Updating Images

Pull the latest images:

```bash
docker compose -f compose-medias.yml pull
docker compose -f compose-observability.yml pull
```

Recreate containers using the updated images:

```bash
docker compose -f compose-medias.yml up -d
docker compose -f compose-observability.yml up -d
```

Unused images can be removed periodically with:

```bash
docker image prune
```

## Migrating to Named Projects

Earlier versions of this repository did not set `name:` in the Compose files, so both stacks ran
as one project named after the directory. Containers created that way have to be recreated once
after pulling this change:

1. Find the old project name and the Portainer volume:

   ```bash
   docker compose ls
   docker volume ls --filter name=portainer_data
   ```

2. Stop both stacks under the old project name (`compose` in the example; if you ran the version
   that already named the projects, the observability stack shows up as `utils`, its earlier
   name, and its file was `compose-utils.yml`):

   ```bash
   docker compose -p compose -f compose-medias.yml down
   docker compose -p compose -f compose-observability.yml down
   ```

   The first command may warn that the network is still in use; the second one removes it.
   Configuration under `config/` and the Portainer volume are kept.

3. The Portainer volume is named after the project, `observability_portainer_data`, so the
   volume listed in step 1 (`compose_portainer_data`, for example) is not reused and Portainer
   starts with an empty one. To keep the old data, point the volume at it in
   `compose-observability.yml` before starting:

   ```yaml
   volumes:
     portainer_data:
       name: compose_portainer_data   # the name listed in step 1
   ```

4. Start both stacks again:

   ```bash
   docker compose -f compose-medias.yml up -d
   docker compose -f compose-observability.yml up -d
   docker compose ls   # should list "media" and "observability"
   ```

Uptime Kuma monitors that used service names such as `http://jellyfin:8096` only worked because of
the shared network; switch them to `http://SERVER_IP:<port>`.

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

Replace `SERVER_IP` with the local IP address of the Ubuntu Server (`LAN_IP` in `.env`). From outside the house, see
[Remote access](setup/expose/README.md).

### Media

```text
Jellyfin       http://SERVER_IP:8096
qBittorrent    http://SERVER_IP:8080
Prowlarr       http://SERVER_IP:9696
Radarr         http://SERVER_IP:7878
Sonarr         http://SERVER_IP:8989
Bazarr         http://SERVER_IP:6767
```

### Observability

```text
Portainer      https://SERVER_IP:9443
Uptime Kuma    http://SERVER_IP:3001
```

Portainer is published only over HTTPS on `9443`, with a self-signed certificate; its HTTP port
`9000` is not published.

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

All services get it from the `x-common` block at the top of each Compose file:

```yaml
x-common: &common
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
