# 🔧 Dropping a systemd Service From Root to Least-Privilege

> *A network daemon only needed two specific kernel capabilities, not everything root can do. Getting there cleanly took more than the "obvious" systemd directive.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | systemd, Linux capabilities (`CAP_NET_ADMIN`, `CAP_NET_BIND_SERVICE`) |
| **Impact** | A TUN/TAP-creating, privileged-port-binding daemon ran as full root with no technical need to |
| **Status** | ✅ Resolved |

---

## 🔍 Symptom / Goal

A daemon that creates a TUN/TAP interface and binds a port below 1024 had traditionally run as full root, but it only actually needs one or two specific kernel capabilities, not everything root can do.

## 🚫 Ruled Out

The "obvious" approach, `User=_<service>` plus `AmbientCapabilities=CAP_NET_ADMIN` and `CapabilityBoundingSet=CAP_NET_ADMIN` in the unit, failed outright on this systemd/kernel combination with `Failed at step EXEC: Operation not permitted` (status 203/EXEC). `AmbientCapabilities=` alone is not reliably sufficient to let a non-root `User=` actually receive the capability at exec time.

## 🎯 Root Cause / Approach

**Set the capability as a file property directly on the binary** instead, which sidesteps the systemd/kernel interaction that blocks `AmbientCapabilities` alone:
```bash
setcap cap_net_admin,cap_net_bind_service+ep /path/to/binary
```
Still combined with `User=_<service>` in the unit so the process runs unprivileged apart from the granted capabilities.

## 🛠️ Fix

1. Create a dedicated system user: `useradd --system --no-create-home --shell /usr/sbin/nologin _<service>`
2. Fix ownership of its real data directory: `chown -R _<service>:_<service> /var/lib/<service>`
3. `setcap` the binary as above, iterating capabilities from real errors rather than guessing upfront: `CAP_NET_ADMIN` alone got the TUN/TAP interface working but left a separate failure binding a privileged port (`failed to bind udp socket on <ip>:53: permission denied`), which needed `CAP_NET_BIND_SERVICE` added too.
4. A separate gotcha unrelated to capabilities: if the service fails with `bind: permission denied` on a unix socket directly under `/run/`, that's a plain directory-permission problem: `/run` itself is `root:root drwxr-xr-x` and a non-root user can't create new top-level entries there regardless of capabilities. Fixed with systemd's `RuntimeDirectory=` directive, which auto-creates an owned subdirectory under `/run/` before each start, then pointed the service's config at the new path.

## ✅ Verification

A clean `systemctl status` isn't enough to confirm the privilege drop didn't silently break real functionality. Checked for actual evidence in the logs (real outbound connections, an actual interface up with a real address, genuine traffic) and confirmed via `ps aux` that the process runs as the intended unprivileged user, not root, across every service touched in the same pass.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
