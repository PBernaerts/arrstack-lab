# Storage Layout

The array design: independent data disks unioned by mergerfs, protected by a single
SnapRAID parity disk, with an NVMe cache for appdata and in-progress downloads.

> Sanitized: serial numbers and filesystem UUIDs are intentionally not committed. Disks
> are addressed by `/dev/disk/by-id/` (model + serial) on the host; the real IDs live in
> the host's `/etc/fstab` and `snapraid.conf`, not in this repo.

## Disks

| Role | Capacity | Model | Filesystem | Mount |
|---|---|---|---|---|
| parity | 28 TB | Seagate Exos | XFS | holds the SnapRAID parity file |
| disk1 | 28 TB | Seagate Exos | XFS | `/mnt/disk1` |
| disk2 | 28 TB | Seagate Exos | XFS | `/mnt/disk2` |
| disk3 | 12 TB | Seagate Exos | XFS | `/mnt/disk3` |
| disk4 | 12 TB | Seagate Exos | XFS | `/mnt/disk4` |
| disk5 | 12 TB | Seagate Exos | XFS | `/mnt/disk5` |
| cache | 2 TB | Samsung NVMe | btrfs | `/mnt/cache` |

- The five data disks (`/mnt/disk1`..`/mnt/disk5`) are unioned by mergerfs into
  `/mnt/user`.
- The **parity disk sits outside the mergerfs pool**. Its capacity must be at least as
  large as the largest data disk (so 28 TB here). It is touched only by writes and
  `snapraid sync`/`scrub`, never by reads, which is why it is the one disk that spins down.
- The cache is not part of the parity set; appdata is backed up separately.

## Addressing convention

Always reference disks by `/dev/disk/by-id/` rather than `/dev/sdX`, because `sdX` names
are not stable across reboots. Find them with `ls -l /dev/disk/by-id/`.

## Templated `/etc/fstab` (data disks)

Replace each `REPLACE-WITH-BY-ID` with the real `/dev/disk/by-id/ata-...` entry on the host.

```fstab
# data disks (XFS) — by-id, nofail so a missing disk does not block boot
/dev/disk/by-id/REPLACE-WITH-BY-ID-1  /mnt/disk1  xfs  defaults,nofail  0 0
/dev/disk/by-id/REPLACE-WITH-BY-ID-2  /mnt/disk2  xfs  defaults,nofail  0 0
/dev/disk/by-id/REPLACE-WITH-BY-ID-3  /mnt/disk3  xfs  defaults,nofail  0 0
/dev/disk/by-id/REPLACE-WITH-BY-ID-4  /mnt/disk4  xfs  defaults,nofail  0 0
/dev/disk/by-id/REPLACE-WITH-BY-ID-5  /mnt/disk5  xfs  defaults,nofail  0 0
# parity disk (XFS)
/dev/disk/by-id/REPLACE-WITH-BY-ID-P  /mnt/parity xfs  defaults,nofail  0 0
# cache (btrfs)
/dev/disk/by-id/REPLACE-WITH-BY-ID-C  /mnt/cache  btrfs defaults,nofail 0 0

# mergerfs pool over the data disks
/mnt/disk*  /mnt/user  fuse.mergerfs  defaults,nonempty,allow_other,use_ino,category.create=mfs,minfreespace=200G,fsname=mergerfs,nofail  0 0
```

## Templated `snapraid.conf` (skeleton)

A starting point. Tune content-file locations and exclusions to taste; finalize during the
migration. See `man snapraid`.

```ini
# one parity disk protects the data disks
parity /mnt/parity/snapraid.parity

# content files: keep several copies on different disks
content /var/snapraid/snapraid.content
content /mnt/disk1/.snapraid.content
content /mnt/disk3/.snapraid.content

# data disks (the mergerfs branches)
data disk1 /mnt/disk1
data disk2 /mnt/disk2
data disk3 /mnt/disk3
data disk4 /mnt/disk4
data disk5 /mnt/disk5

# exclusions
exclude *.unrecoverable
exclude /tmp/
exclude lost+found/
exclude .Trash-*/
exclude **/incomplete/
```
