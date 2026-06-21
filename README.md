# Homelab

Infrastructure-as-code for my self-hosted homelab: a Plex-centric media stack on
Debian 13, with a mergerfs union pool and SnapRAID parity, deployed via Docker Compose.

See [`docs/`](docs/) for the Debian install checklist, the migration steps, and the
storage layout.

> **Status:** migrating from Unraid 7.3.0 to Debian 13. The data disks (XFS) mount
> natively; mergerfs + SnapRAID replace the Unraid array in place. Validated in a Proxmox
> test VM; real-server cutover pending. The original Unraid USB is retained as instant
> rollback.

> **Note on values:** every host-specific value in this repo is a placeholder
> (`example.com`, generic ports, RFC1918 IPs). Real domains, public IPs, and secrets are
> never committed. Copy `.env.example` to `.env` and fill in your own.

---

## Architecture

```
                 Internet
                    |
            Cloudflare Tunnel
                    |
              Caddy (reverse proxy)
                    |
   +----------------+-----------------+
   |        Docker Compose stack      |
   |  Plex, *arr, downloaders, etc.   |
   +----------------+-----------------+
                    |
        mergerfs pool  (/mnt/user)
        +------+------+------+ ...     <- XFS data disks
                    |
            SnapRAID parity disk        <- outside the pool
```

- **Host:** Debian 13. Intel iGPU/Arc GPU for Plex hardware transcoding.
- **Orchestration:** Docker Compose on a custom bridge network.
- **Storage:** XFS data disks unioned by **mergerfs** into `/mnt/user`, protected by a
  **SnapRAID** parity disk. NVMe cache holds appdata and in-progress downloads.
- **Ingress:** Caddy behind a Cloudflare tunnel. No inbound ports exposed directly.

## Repository layout

| Path | Purpose |
|---|---|
| `compose.yaml` | The full service stack. |
| `.env.example` | Template for `.env` (the real `.env` is gitignored). |
| `.pre-commit-config.yaml` | gitleaks secret scanning + basic file hygiene. |
| `.gitignore` | Keeps secrets and runtime data out of git. |
| `docs/` | Debian install checklist, migration steps, storage layout. |
| `scripts/` | Maintenance scripts. _(planned)_ |
| `systemd/` | Service + timer units. _(planned)_ |

## Quick start

1. `cp .env.example .env` and set every value.
2. `docker compose up -d`.
3. For host preparation and the migration procedure, see
   [`docs/debian13-install.md`](docs/debian13-install.md) and
   [`docs/migration.md`](docs/migration.md).

> After cloning, run `pre-commit install` once so the gitleaks hook fires on commit.

## Configuration

All configuration is via `.env`:

| Variable | Meaning |
|---|---|
| `PUID` / `PGID` | User/group IDs the containers run as. |
| `TZ` | Timezone, e.g. `Europe/Brussels`. |
| `APPDATA` | Per-service config root (on NVMe cache). |
| `DATA` | Root of the media + downloads tree (the mergerfs pool). |
| `CF_TUNNEL_TOKEN` | Cloudflare tunnel token. **Secret.** |
| `DISCORD_WEBHOOK` | Webhook for alerts. **Secret.** |

## Services

All ports below are internal to the host and reached through the Cloudflare tunnel and
Caddy. Nothing is published to the internet directly.

| Service | Port | Role |
|---|---|---|
| plex | host mode | Media server (HW transcoding) |
| tautulli | 8181 | Plex analytics |
| seerr | 5055 | Request management |
| kometa | — | Metadata / collections |
| pal | — | Per-user audio/subtitle selection |
| wizarr | 5690 | User invitations |
| prowlarr | 9696 | Indexer management |
| sonarr / sonarr-anime | 8989 / 8991 | TV / anime |
| radarr / radarr-remux | 7878 / 7880 | Movies / remux |
| clonarr | 6060 | Quality-profile sync |
| cross-seed | 2468 | Cross-seeding |
| sabnzbd | 8085 | Usenet downloader |
| qbittorrent | internal | Torrent downloader + seeding |
| anibridge | 4848 | Anime bridge |
| kavita | 5000 | Ebooks / manga |
| actual_server | 5006 | Budget tracking |
| caddy | 80 / 443 | Reverse proxy |
| diun | — | Image-update notifications |

## Storage

Data disks are XFS, unioned by mergerfs into `/mnt/user`. A dedicated SnapRAID parity
disk (outside the mergerfs pool) provides single-disk-failure protection. SnapRAID
`sync`/`scrub` run on a schedule. The parity disk spins down when idle via hd-idle; data
disks stay spinning. See [`docs/storage-layout.md`](docs/storage-layout.md) for the disk
table and templated fstab/snapraid configs.

## Maintenance

- Watchdogs and cleanup jobs run as systemd timers (see `scripts/`, `systemd/`).
- Drive spindown: parity disk only.
- Backups: scheduled appdata backup; SnapRAID parity covers the media disks.

## Secret handling

- `.env` is gitignored; only `.env.example` (placeholders) is committed.
- **gitleaks** runs as a pre-commit hook to block accidental credential commits.
- Run `pre-commit install` after cloning, or the hook will not fire.

## Recovery / rollback

- The original Unraid USB boots the previous setup unchanged.
- Rebuild path: provision the host (`docs/debian13-install.md`), restore appdata, clone
  this repo, create `.env`, `docker compose up -d`.

## License

Licensed under the MIT License. See [`LICENSE`](LICENSE).
