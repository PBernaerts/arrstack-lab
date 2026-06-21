# Unraid to Debian 13 Migration
In-place migration: the XFS data disks are reused as-is; mergerfs + SnapRAID replace the
Unraid array. The Unraid USB stays untouched as instant rollback.

Everything below is rehearsed first in a Debian 13 VM on Proxmox.

## Testing

- [ ] Full appdata backup verified restorable.
- [ ] Rehearse the whole procedure in the Proxmox test VM.
- [ ] Inventory: container list, ports, volumes, API keys.
- [ ] Confirm the Unraid USB is the clean rollback.
- [ ] Schedule a maintenance window (no Plex users mid-stream).

## Phase A

- [ ] Stop the Docker stack on Unraid.
- [ ] Stop the array cleanly.
- [ ] Note the exact disk-to-slot mapping before pulling anything.

## Phase B

- [ ] Boot the Debian system disk (separate from the array; Unraid USB stays in place as
      fallback boot).
- [ ] Verify each data disk mounts read-only first; confirm filesystems are clean.

## Phase C

- [ ] Mount data disks (by-id) per `debian13-install.md`.
- [ ] Bring up the mergerfs pool; verify `/mnt/user` shows the unified tree.
- [ ] Configure SnapRAID; run `sync` (first parity build or re-sync).

## Phase D

- [ ] Restore appdata to the cache location.
- [ ] `docker compose up -d`.
- [ ] Verify each service reads its config and sees `/data` correctly (hardlinks intact).
- [ ] Confirm Plex transcoding (GPU passthrough) works.

## Phase E

- [ ] Rotate API keys and webhooks; re-point every consumer
      (Prowlarr/Seerr/clonarr/scripts).
- [ ] Install maintenance scripts as systemd timers (watchdogs, seeding cleanup,
      SnapRAID sync/scrub, hd-idle parity spindown).
- [ ] Add the mergerfs/branch mount alerting the watchdog needs.
- [ ] Monitor for 48h: disk health, temps, spindown behaviour, Plex playback.

## Rollback

- [ ] If anything is wrong: stop Debian, boot the Unraid USB, start the array. The data
      disks are untouched by the above, so Unraid comes back as it was.
