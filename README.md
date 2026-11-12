# media-stack

A self-hosted media stack for movies, TV, and music: acquisition (*arr apps,
Prowlarr, qBittorrent, slskd), streaming (Jellyfin, Navidrome), discovery/requests
(Jellyseerr, explo, Suggestarr), all torrent/Soulseek traffic tunneled through
ProtonVPN, and every service published on its own `*.deepblue597.org` subdomain via
Traefik.

See [ARCHITECTURE.md](ARCHITECTURE.md) for a diagram of how the networks, the VPN
boundary, and the media pipeline fit together.

## What's deployed

| Service | Purpose |
|---|---|
| **vpn** (gluetun) | ProtonVPN over WireGuard; tunnels everything that needs to torrent/P2P |
| **qBittorrent** | Torrent client, runs inside the VPN's network namespace |
| **mousehole** | Keeps a MyAnonaMouse (MAM) tracker session pinned to the VPN's current IP/port |
| **Prowlarr** | Indexer manager, feeds indexers to Radarr/Sonarr/Lidarr |
| **cross-seed** | Daemon that cross-seeds existing torrents across trackers via Prowlarr + qBittorrent |
| **Radarr** | Movie library management |
| **Sonarr** | TV show library management |
| **Lidarr** | Music library management |
| **slskd** | Soulseek client/daemon, used for music acquisition |
| **Navidrome** | Subsonic-compatible music streaming server |
| **explo** | Music discovery app for finding new music to listen to |
| **aurral** | Automated music downloader |
| **lrcget** | Watches the music library and auto-downloads missing `.lrc` lyrics from LRCLIB |
| **Jellyfin** | Media server/streamer for movies & TV (NVIDIA GPU transcoding) |
| **Jellyseerr** | Request management front-end for Jellyfin |
| **Suggestarr** *(optional)* | Recommends movies/shows from Jellyfin watch history and auto-requests them through Jellyseerr |
| **Traefik** *(separate compose file)* | Reverse proxy/TLS termination, routes each subdomain to its container |

## Requirements

- Docker version 28.0.1 or later
- Docker Compose version v2.33.1 or later
- A domain with DNS you control (this deployment uses `deepblue597.org`), so you can
  point `*.yourdomain` subdomains at the host running Traefik
- A ProtonVPN account with WireGuard config (or another provider — see
  [Gluetun's supported providers](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers))

## Networking prerequisites

Two external Docker networks must exist before bringing up the stack:

```bash
docker network create --subnet 172.20.0.0/16 mynetwork   # internal service-to-service traffic
docker network create proxy                                # Traefik ingress
```

Traefik itself is a separate compose file (`docker-compose-traefik.yml`) so it can be
brought up/torn down independently of the media stack:

```bash
DOMAIN=deepblue597.org LE_EMAIL=you@example.com HASHED_ADMIN_USER_PASS='...' \
  docker compose -f docker-compose-traefik.yml up -d
```

`HASHED_ADMIN_USER_PASS` protects the Traefik dashboard — generate it with:

```bash
htpasswd -B -C 10 -c .htpasswd user1
cat .htpasswd | sed -e s/\\$/\\$\\$/g
```

Point DNS `A`/`AAAA` records for each subdomain used in `docker-compose.yml`'s Traefik
labels (`radarr.`, `sonarr.`, `lidarr.`, `prowlarr.`, `qbit.`, `slskd.`, `navidrome.`,
`explo.`, `aurral.`, `jellyfin.`, `jellyseerr.`, `suggestarr.`) at the host running
Traefik.

## Environment variables (`.env`)

The compose file reads secrets/config from a `.env` file (gitignored — never commit
it). Required variables, by service:

- **VPN:** `WIREGUARD_PRIVATE_KEY` (ProtonVPN WireGuard key)
- **Static IPs** (needed because Prowlarr runs behind the VPN and reaches Radarr/Sonarr
  by IP, not service name): `RADARR_STATIC_CONTAINER_IP`, `SONARR_STATIC_CONTAINER_IP`
- **mousehole:** `MOUSEHOLE_AUTH_PASSWORD`, `MOUSEHOLE_ALLOWED_HOSTS`
- **slskd:** `SLSKD_USERNAME`, `SLSKD_PASSWORD`, `SLSKD_API_KEY`, `SOULSEEK_USERNAME`,
  `SOULSEEK_PASSWORD`
- **explo:** `EXPLO_UI_USERNAME`, `EXPLO_UI_PASSWORD`

## Deploying

```bash
# With VPN (recommended — required for torrenting privacy)
docker compose --profile vpn up -d

# Without VPN (discouraged)
docker compose --profile no-vpn up -d

# Add optional recommendations (Suggestarr)
COMPOSE_PROFILES=vpn,suggestarr docker compose up -d
```

Running `docker compose up -d` with no profile deploys nothing — this is intentional,
so you can't accidentally start torrenting containers without a VPN.

## cross-seed configuration

`cross-seed-config.example.js` is a template (gitignored so a filled-in copy never gets
committed). Copy it to `/mnt/mediaSSD/mediaStack/cross-seed-config/config.js` and fill
in your real Prowlarr API key and qBittorrent WebUI password. **Never commit the
filled-in file** — it contains live credentials.

## Configure qBittorrent

- Open `https://qbit.deepblue597.org`. Default username is `admin`; temporary password
  is in the container log: `docker logs qbittorrent`
- Tools → Options → WebUI → change the password
- Inside the container, create download subfolders as needed:
  ```bash
  docker exec -it qbittorrent bash
  mkdir /downloads/movies /downloads/tvshows
  chown 1000:1000 /downloads/movies /downloads/tvshows
  ```

## Configure Radarr / Sonarr / Lidarr

- Open `https://radarr.deepblue597.org` (or `sonarr.`/`lidarr.`)
- Media Management → enable "deleted from disk are automatically unmonitored" → add
  root folder pointing at your downloads path
- Download Clients → add qBittorrent. **Since qBittorrent runs inside the VPN's network
  namespace, use host `vpn` (not `qbittorrent`) and port `5080`.**
- General → enable advanced settings → set up authentication
- Indexers are added automatically once Prowlarr's "Apps" are configured (below)

## Configure Prowlarr

- Open `https://prowlarr.deepblue597.org`
- General → Authentication → set up a username/password
- Indexers → Add Indexer → search, pick, configure, test, save
- Settings → Apps → add Radarr/Sonarr/Lidarr. **Because Prowlarr runs behind the VPN,
  it can't reach them by service name — use their static container IPs** (e.g.
  `http://172.20.0.2:7878`), matching `RADARR_STATIC_CONTAINER_IP` /
  `SONARR_STATIC_CONTAINER_IP` from your `.env`.

## Configure slskd (Soulseek)

- Open `https://slskd.deepblue597.org`, log in with `SLSKD_USERNAME`/`SLSKD_PASSWORD`
- In Lidarr, add slskd as a download client using host `vpn` and slskd's port, since it
  also shares the VPN's network namespace

## Configure Navidrome / explo

- Navidrome (`https://navidrome.deepblue597.org`) points at the shared music library —
  no extra setup needed beyond first login
- explo (`https://explo.deepblue597.org`) is a discovery front-end over the same music
  library — log in with `EXPLO_UI_USERNAME`/`EXPLO_UI_PASSWORD`

## Configure Jellyfin

- Open `https://jellyfin.deepblue597.org`
- First-run guided setup will walk you through adding libraries — point movies/TV at
  the shared downloads/media path

## Configure Jellyseerr

- Open `https://jellyseerr.deepblue597.org`
- First-run guided setup: connect it to Jellyfin, then Radarr and Sonarr
- See the [Overseerr docs](https://docs.overseerr.dev/) (Jellyseerr is a fork of
  Overseerr) for anything not covered by the wizard

## Configure Suggestarr (optional)

Suggestarr reads your Jellyfin watch history, finds similar titles on TMDB, and sends
requests to Jellyseerr (which then hands them to Radarr/Sonarr). You need a free
[TMDB API key](https://www.themoviedb.org/settings/api). An OpenAI-compatible LLM is
optional, for AI-based suggestions.

- Open `https://suggestarr.deepblue597.org` and follow the setup wizard
- TMDB → paste your API key
- Media server → Jellyfin, URL `http://jellyfin:8096`, API key from Jellyfin
  Dashboard → API Keys, then pick the users/libraries to use
- Seer → URL `http://jellyseerr:5055`, API key from Jellyseerr Settings → General
- Set the schedule and how many suggestions per watched item; use the connection
  tests in each step

## Disclaimer

> Neither the author nor the developers of the code in this repository **condone or
> encourage** downloading, sharing, seeding, or peering of **copyrighted material**.
> Such activities are **illegal** under international laws.
>
> This project is intended for **educational purposes only**.

## Credits

Originally based on [linuxshots/media-stack](https://github.com/linuxshots/media-stack).
