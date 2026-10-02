# 🔧 Self-Healing USB Drive Remount via udev

> *Every client against a share failing with I/O errors looks like corruption: it was actually a stale device handle left behind after a routine USB re-enumeration.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | Linux host (Proxmox + LXC bind mount), USB-attached external enclosure |
| **Impact** | Every client touching the share (file manager, kernel CIFS mount, smbclient) failed with I/O errors after a mid-session drop/reconnect |
| **Status** | ✅ Resolved, self-healing automation in place |

---

## 🔍 Symptom

A USB-attached drive dropped and reconnected mid-session, and Linux assigned it a **different device letter** on reconnect (`/dev/sdc` → `/dev/sdg`). Anything that had mounted it by the old device path, or a stale fstab entry using `/dev/sdX` instead of a UUID, silently broke. Every client against the share failed with I/O errors, regardless of which tool or mount option was tried, which looks alarming enough to suspect filesystem corruption or a dying drive.

The real signature in dmesg/journal was a stale handle, not corruption:

```
EXT4-fs warning (device sdc1): dx_probe:791: inode #...: lblock 0:
comm smbd: error -5 reading directory block
```

(`error -5` = EIO on a device node that no longer exists.)

## 🚫 Ruled Out

```bash
smartctl -H /dev/sdc                        # "No such device" = the node vanished, not a SMART failure
dmesg | grep -i 'ata.*reset\|ata.*error'    # zero bus-reset/controller errors = not a hardware fault
```

Both came back clean, and the drive reappeared under a new letter with the same LABEL/UUID and data reading back intact, confirming this was a stale-handle problem, not corruption or drive failure. The fix is a remount, not an `fsck` or a replacement.

## 🎯 Root Cause

The mount was keyed to a device path (`/dev/sdX`) rather than the filesystem's UUID, so when the kernel re-enumerated the drive under a new letter, every existing handle pointed at a device node that no longer resolved to anything.

## 🛠️ Fix

**Manual fix first**, to confirm the theory before automating anything:
```bash
pct stop <ctid>          # release the container's hold on the mount, if any
umount -l /mount/point   # lazy-unmount the stale handle
mount /mount/point       # remounts via fstab's UUID entry -> resolves to the new letter
pct start <ctid>
```
This only works cleanly once the fstab entry references the drive by UUID, not a raw device path; confirmed that first.

**Then automated it** with a UUID-keyed fstab entry plus a udev rule that fires on any reconnect:

```
# /etc/fstab
UUID=<drive-uuid>  /mount/point  ext4  defaults  0  2
```

```bash
#!/bin/bash
# uuid-remount.sh, safe to run even if the mount looks fine; idempotent
UUID="<drive-uuid>"
MOUNTPOINT="/mount/point"
umount -l "$MOUNTPOINT" 2>/dev/null
mount UUID="$UUID" "$MOUNTPOINT"
```

```
# /etc/udev/rules.d/99-remount.rules
ACTION=="add", SUBSYSTEM=="block", ENV{ID_FS_UUID}=="<drive-uuid>", RUN+="/path/to/uuid-remount.sh"
```

For the Proxmox case, a dependent LXC's bind-mount doesn't refresh just because the host remounted underneath it: the udev script also restarts the dependent container:
```bash
pct stop <ctid> && pct start <ctid>
```

## ✅ Verification

`udevadm control --reload-rules` then `udevadm trigger` exercises the whole chain (device reconnect → host remount → container restart) without physically unplugging anything, confirming it end-to-end.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
