# 🔧 Diagnosing a Wedged Kernel Storage Thread (D-State)

> *A VM that's "running" but unreachable on every port isn't a networking problem: it's a kernel thread stuck forever on an I/O request that will never complete.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | Proxmox VE host, VM/LXC guests, USB-attached storage enclosures |
| **Impact** | Guest unreachable (no ping, no SSH, no guest-agent) despite reporting `running`; recurred independently three times across different hosts in this lab |
| **Status** | ✅ Resolved per-incident (documented runbook now standard) |

---

## 🔍 Symptom

`qm status`/`pct list` reports the guest as `running`, but it answers nothing: no ping, no SSH, no QEMU guest-agent. The process is alive; the guest kernel (or a host-side thread it depends on) is wedged. On the host itself, commands that should be instant, like `df`, `stat`, `blkid`, `mount`, hang indefinitely instead of erroring. One variant of this looked like an SSH credentials failure (pubkey auth rejected) but was actually an `Input/output error` reading `.ssh/authorized_keys` off a wedged disk.

## 🚫 Ruled Out

Before assuming a dying drive, I always check whether this is a genuine hardware failure or just a stuck in-flight request; the fixes are completely different:

```bash
dd if=/dev/sdX of=/dev/null bs=4k count=100   # returns fast & clean? drive is readable right now
dmesg -T | grep -i -E "ata.*reset|ata.*error|usb.*disconnect"   # nothing? no real bus event occurred
```

A clean `dd` plus zero bus-reset/disconnect evidence in dmesg rules out actual hardware failure; it's a stuck request, not a dying drive.

## 🎯 Root Cause

A kernel thread (often `[usb-storage]` itself) stuck in **uninterruptible sleep (D-state)** on a block-device I/O request that never completes, usually triggered by a USB bridge/enclosure hiccup. Everything downstream wedges with it: SSH, the guest-agent, `df`, backups, and even unrelated processes that happen to touch the same disk or a shared kernel worker queue. `dmesg` typically shows `__filemap_fdatawait_range` / `blkdev_fsync` in the hung-task call trace.

**Diagnosis order, cheapest/safest first:**
1. `ps aux | awk '$8 ~ /D/'` to find the actual stuck process(es); note how long it's been stuck (the STARTED column) to confirm it's not a fresh transient blip.
2. `dmesg -T | grep -i -E "hung|blocked for|I/O error|usb.*disconnect|usb.*reset" | tail -30`
3. The hardware-vs-stuck-request check above.
4. For a VM with no network/guest-agent access, pull a console screenshot via the Proxmox HMP monitor (`qm monitor <vmid>` → `screendump /tmp/console.ppm`) rather than hunting for a CLI subcommand that doesn't exist.
5. Avoid piling on more hangs while diagnosing: don't re-run `blkid`/`mount`/`df` against the already-wedged device; resolve UUIDs via `/dev/disk/by-uuid/` instead, which doesn't touch the device.

## 🛠️ Fix

- **A wedged VM:** `qm stop <vmid> --skiplock` (a normal stop won't work on a wedged guest). If `qm start` then fails with `VM is locked (backup)`, that's a stale lock from an interrupted backup: `qm unlock <vmid>` first, then `qm start <vmid>`. Expect the DHCP lease to possibly change on reboot.
- **A wedged host-level USB device** is higher-risk since it can affect every guest on the host, not just one. Before touching anything: check for a live backup job stuck on the same wedge (resetting the bus mid-write risks that job outright), and treat a host reboot or a force-killed backup as a call that needs explicit sign-off, not something to push through under a general "keep going."
- A device that re-enumerates under a new letter (`/dev/sdc` → `/dev/sdg`) is normal and not itself the bug: never hardcode `/dev/sdX`; always resolve by UUID/LABEL.
- A stale LXC bind-mount doesn't refresh on its own even once the host side recovers; `pct reboot <ctid>` afterward.

## ✅ Verification

Re-check `ps aux | awk '$8 ~ /D/'` is clear, dmesg stays quiet, and the guest actually answers: ping, SSH, and guest-agent all respond, not just that `qm status` says `running` again.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
