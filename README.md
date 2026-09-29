# Homelab

Self-hosted media server, monitoring stack and personal cloud, running on OpenMediaVault with Docker Compose. Everything is reachable privately over Tailscale; nothing is exposed to the public internet.

## Architecture

```mermaid
flowchart LR
    subgraph Clients
        Mac[Mac] & iPhone[iPhone] & Laptop[Laptop]
    end

    Clients -->|Tailscale| Server

    subgraph Server[mediaserver · OMV 8 · Ryzen 7 5825U]
        Homepage[Homepage<br/>dashboard]
        subgraph Media
            Jellyfin[Jellyfin<br/>VAAPI] 
            Arr[Sonarr · Radarr<br/>Seerr · Maintainerr]
        end
        subgraph VPN[Gluetun · Proton VPN]
            qBit[qBittorrent]
            Searx[SearXNG]
        end
        subgraph Monitoring
            Mon[Uptime Kuma · Beszel<br/>Scrutiny · SmokePing<br/>Speedtest Tracker]
        end
        Proxy[Docker socket proxy<br/>read-only]
        Homepage --> Proxy
    end

    VPN -->|WireGuard, kill switch| Internet((Internet))
    Server --> NVMe[(1TB NVMe<br/>OS · containers · app data)]
    Server --> HDD[(4TB HDD · mergerfs<br/>media + downloads)]
```

## Hardware

| Component | Choice | Why |
|---|---|---|
| Server | AOOSTAR WTR Pro, AMD Ryzen 7 5825U | 8 Zen 3 cores for CPU-heavy work (subtitle rendering, photo indexing), 4 drive bays, 2× 2.5GbE |
| RAM | 16GB DDR4-3200 (1 of 2 slots) | Enough for the current stack; a second identical stick gives 32GB dual-channel |
| System disk | 1TB NVMe | OS, containers and all app data |
| Media disk | 4TB WD Red Plus (CMR) | Quiet 5400 rpm class, CMR for parity-friendly writes |
| OS | OpenMediaVault 8 (Debian 13) | Web-managed NAS OS with a Docker Compose plugin |

## Services

| Group | Service | Purpose | Port |
|---|---|---|---|
| Dashboard | Homepage | One page for everything; tiles come from Docker labels | 3000 |
| Media | Jellyfin | Streaming with VAAPI hardware transcoding; MyAnimeList sync via Ani-Sync | 8096 |
| Media | Sonarr / Radarr | Series and movie library management (renaming, organizing) | 8989 / 7878 |
| Media | Seerr | Browse and request titles; users log in with their Jellyfin accounts | 5055 |
| Media | Maintainerr | Rule-based library cleanup | 6246 |
| Downloads | qBittorrent | Download client, routed through Gluetun | 8080 |
| Privacy | Gluetun | Proton VPN (WireGuard) with port forwarding and a kill switch | — |
| Privacy | SearXNG | Private meta search engine, routed through Gluetun | 8888 |
| Monitoring | Uptime Kuma | Service uptime checks | 3001 |
| Monitoring | Beszel | CPU, RAM, disk, temperatures and per-container usage | 8090 |
| Monitoring | Scrutiny | Drive health (SMART) with history | 8083 |
| Monitoring | SmokePing | Continuous latency to the router, ISP hops and public DNS | 8081 |
| Monitoring | Speedtest Tracker | Scheduled speed tests with history | 8082 |

## Storage

```
NVMe (/)
├── /compose        this repo: one folder per stack
├── /appdata        container config and data (backed up, not in Git)
└── /backup         backups and database dumps

HDD (/srv/mergerfs/data, mergerfs pool)
├── torrents/
│   ├── downloading/
│   └── completed/
└── media/
    ├── anime/  series/  movies/  youtube/
    └── books/  manga/  music/
```

- **Why mergerfs (+ SnapRAID later) instead of RAID5:** drives can be added one at a time and in mixed sizes, a failed drive only loses its own files, and idle drives can spin down. Parity will come from a dedicated 8TB SnapRAID drive.
- **One `/data` mount** for the download client and library managers, so finished downloads are moved or hardlinked instantly instead of copied.
- **Jellyfin mounts `media` read-only.** Metadata and artwork live in its own config folder.
- **Irreplaceable data** (photos, files, passwords) lives on the NVMe with off-site backups, not on the unprotected HDD.

## Networking and access

- **Tailscale** for all access: the server sits behind Odido 5G with **CGNAT**, so there are no open ports. Clients connect over WireGuard, directly where possible.
- **Exit node:** the server can act as a personal VPN on public Wi-Fi, opt-in per device.
- **Gluetun:** qBittorrent and SearXNG share Gluetun's network namespace, so all their traffic leaves through Proton VPN. If the VPN drops, they have no network at all (verified). `FIREWALL_OUTBOUND_SUBNETS` keeps replies to the LAN, Docker and Tailscale off the tunnel.
- **Fixed IP** through a DHCP reservation on the router rather than a static IP in the OS, so moving house only needs a new reservation.

## Security

- **Secrets never go in Git.** They live in each stack's `.env` file (ignored); `.env.example` files show which variables are needed.
- **gitleaks** runs as a pre-commit hook, and GitHub push protection is enabled.
- **Docker socket access goes through a read-only proxy** (`docker-socket-proxy`, `POST=0`). Direct socket access is effectively root on the host, even with `:ro`.
- **Least privilege:** containers run as a non-root user (`PUID=1000`, `PGID=100`) where possible; media is read-only for Jellyfin.
- **SSH** with keys only; no service is published to the internet.

## Incidents and lessons learned

**Secret in Git history.** A token was committed in an early version of a compose file. gitleaks caught it before the repo went public. The token was rotated first, then the history was rewritten, and a pre-commit hook plus push protection now block it from happening again.

**Tailscale control-plane stall.** After a network change on the 5G connection, the server kept its peer connections but its long-poll to Tailscale's coordination server timed out every two minutes. DNS, IPv4, MTU and IPv6 were ruled out step by step; restarting `tailscaled` fixed it. A systemd timer now restarts it automatically if the node stays offline for two consecutive checks.

**API incompatibility.** Jellyseerr failed to authenticate against Jellyfin 12 with HTTP 400 (not 401), which pointed to a request-format change rather than bad credentials. Replaced with its successor, Seerr.

**Browser transcoding.** Browsers transcode MKV, Opus/5.1 audio and styled subtitles. One file used HEVC 4:4:4 10-bit, which the iGPU can't decode, so it fell back to the CPU. Native clients (Jellyfin Media Player, Infuse) direct-play these instead.

## Conventions

- One folder per stack, managed by the OMV compose plugin; container data in `appdata` via `CHANGE_TO_COMPOSE_DATA_PATH`.
- Every service gets Homepage labels and an Uptime Kuma monitor.
- Containers in different stacks reach each other through the host IP, not container names.
- Stacks that share Gluetun are always restarted together (**Down → Up**), never Gluetun alone.

## Adding a new service

1. Create the compose file in OMV (Services → Compose → Files).
2. Put secrets in the env box, never in the YAML, and add a `.env.example`.
3. Add Homepage labels and an Uptime Kuma monitor.
4. Include its data in backups if it stores anything important.
5. `git pull --rebase` → `git status` (no `.env` files) → `git add .` → `git commit` → `git push`.

## Rebuilding from scratch

1. Install OpenMediaVault, omv-extras, and the compose, mergerfs and sharerootfs plugins.
2. Create the `compose`, `appdata` and `backup` shared folders on the NVMe, and the `data` mergerfs pool on the HDD.
3. Clone this repo into the compose folder.
4. Recreate each stack's `.env` from the password manager, using the `.env.example` files.
5. Restore `appdata` from backup, then bring each stack up.

## Roadmap

- SnapRAID parity drive (8TB) and a second RAM stick
- Off-site backups (Kopia/restic → Hetzner Storage Box) with tested restores
- Personal cloud: Immich (photos), Vaultwarden (passwords), Nextcloud (files)
- Self-built network monitor with automatic latency-spike diagnosis
- Infrastructure as code: Ansible for the server, Terraform for a public VPS reverse proxy
