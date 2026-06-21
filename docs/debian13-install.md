# Debian 13 Checklist

Steps to run after installing Debian 13 on the server.

## 1. Base OS

- [ ] Install Debian 13 (minimal). Set hostname, locale, timezone.
- [ ] Create a non-root admin user; lock down SSH (key-only, non-default port, no root
      login).
- [ ] Unattended security updates.

## 2. Storage

- [ ] Identify data disks by stable ID (`/dev/disk/by-id`), not `/dev/sdX`.
- [ ] Mount each XFS data disk (e.g. `/mnt/disk1` ...). Persist in `/etc/fstab` with
      `nofail`.
- [ ] Install and configure **mergerfs**: union the data disks into `/mnt/user`. Record
      the chosen mount options and create policy.
- [ ] Mount the NVMe cache for appdata and downloads.
- [ ] Install **SnapRAID**: configure data + parity disks and content files in
      `snapraid.conf`. Run the first full `sync`.

## 3. Runtime

- [ ] Install Docker Engine + Compose plugin.
- [ ] Create the appdata and data directory trees with correct ownership (PUID/PGID).
- [ ] Clone this repo to `/opt/<repo>`; `cp .env.example .env` and fill in.
- [ ] `pre-commit install` (if committing from this host).

## 4. Verification

- [ ] `docker compose config` parses cleanly.
- [ ] `docker compose up -d`; all services healthy.
- [ ] Reverse proxy reachable through the tunnel.
- [ ] Break-one-disk test: confirm the host still boots (`nofail`) and mergerfs refuses
      to pool a missing branch (no writes misdirected to root).
