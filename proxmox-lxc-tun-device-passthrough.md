# 🔧 Unprivileged LXC: VPN Daemon Won't Start (Missing /dev/net/tun)

> *A deployment stalling silently at the tunnel daemon's startup step, with no useful error — a one-line Proxmox config fix.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | Proxmox VE, unprivileged LXC container, Tailscale/WireGuard |
| **Impact** | Tunnel daemon setup stalled at startup, with no error pointing at the real cause |
| **Status** | ✅ Resolved |

---

## 🔍 Symptom

An unprivileged Proxmox LXC container has no `/dev/net/tun` device by default — required by Tailscale, WireGuard, and similar tunnel daemons. The symptom is deceptive: deployment/setup just stalls at the tunnel daemon's own startup step, without an error that points at the missing device.

## 🎯 Root Cause

Unprivileged LXCs don't expose `/dev/net/tun` by default, so anything that needs to create a TUN interface silently has nothing to attach to.

## 🛠️ Fix

Add to the container's config (`/etc/pve/lxc/<ctid>.conf`):
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
then `pct reboot <ctid>` for the device passthrough to take effect.

## ✅ Verification

This is a one-time, per-container change — worth checking for first (`cat /etc/pve/lxc/<ctid>.conf | grep tun`) whenever setting up a new unprivileged LXC that needs its own Tailscale/WireGuard client, before troubleshooting the tunnel itself.

---

*Part of the [lab-troubleshooting](README.md) collection — real incidents from a live, multi-node home lab, documented as they happened.*
