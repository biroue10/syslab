# 02 — Storage: LVM from Scratch

## Scenario
The team needs a dedicated `/data` volume for web server content — separate from the root filesystem so it can be resized independently without touching the OS. Built with LVM so it can grow later.

## What Was Configured
- 1GB virtual disk image created at `/tmp/vdisk.img`
- Attached as loop device `/dev/loop0`
- LVM stack: PV → VG (`vg_data`) → LV (`lv_data`, 500MB)
- Formatted with XFS
- Mounted at `/data`, persisted via `/etc/fstab`

## Commands Used

```bash
# 1. Create 1GB virtual disk image
dd if=/dev/zero of=/tmp/vdisk.img bs=1M count=1024

# 2. Attach as loop device
sudo losetup /dev/loop0 /tmp/vdisk.img
sudo losetup -l

# 3. Create Physical Volume
sudo pvcreate /dev/loop0

# 4. Create Volume Group
sudo vgcreate vg_data /dev/loop0
sudo vgs vg_data

# 5. Create Logical Volume (500MB)
sudo lvcreate -L 500M -n lv_data vg_data
sudo lvs vg_data

# 6. Format with XFS
sudo mkfs.xfs -f /dev/vg_data/lv_data
sudo blkid /dev/vg_data/lv_data

# 7. Create mount point
sudo mkdir -p /data

# 8. Mount
sudo mount /dev/vg_data/lv_data /data
mount | grep /data

# 9. Add to fstab (persistent mount)
# Added to /etc/fstab:
# /dev/vg_data/lv_data  /data  xfs  defaults  0 0

# 10. Verify fstab entry
sudo mount -a
```

## LVM Architecture

```
/tmp/vdisk.img  (file)
      │
/dev/loop0  (loop device — acts as a physical disk)
      │
Physical Volume (PV)
      │
Volume Group: vg_data  (1020MB pool)
      │
Logical Volume: lv_data  (500MB)
      │
XFS Filesystem → mounted at /data
```

## Key Commands Reference

| Command | Purpose |
|---------|---------|
| `dd if=/dev/zero of=file bs=1M count=N` | Create a blank file of size N MB |
| `losetup /dev/loopN file` | Attach file as loop block device |
| `losetup -l` | List active loop devices |
| `pvcreate /dev/X` | Initialize a Physical Volume |
| `vgcreate vgname /dev/X` | Create a Volume Group |
| `lvcreate -L size -n name vg` | Create a Logical Volume |
| `mkfs.xfs /dev/vg/lv` | Format with XFS |
| `mount -a` | Mount all fstab entries (test before reboot) |

## Gotchas & Lessons Learned

- **Loop devices are not persistent** — they detach on reboot. In production, use real disks.
- **Always use `mount -a` after editing fstab** — a typo can prevent the system from booting.
- **`-f` flag on mkfs.xfs** — required to force format if previous filesystem metadata exists on the device.
- **Root filesystem went read-only** — caused by I/O errors on the loop device. Fixed with `sudo mount -o remount,rw /`. In production, investigate the root cause before remounting.
- **Exam trap:** forgetting the fstab entry = mount doesn't survive reboot = automatic fail.
- **vgchange -an** deactivates a VG before detaching its underlying device — always do this in order.

## What I Learned
- LVM separates physical storage from logical storage — you can resize, move, and snapshot volumes without downtime.
- The loop device is a powerful tool for practicing storage concepts without extra hardware.
- XFS is the default and expected filesystem on RHEL — use it unless explicitly told otherwise.
- `/etc/fstab` is the persistence layer for mounts — every storage task on the exam requires a correct fstab entry.
