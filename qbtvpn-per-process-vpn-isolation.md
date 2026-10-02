# 🔧 Per-Process VPN Isolation via Network Namespaces

> *Isolating one torrent process behind its own VPN tunnel, without routing the whole host through a client that has no Linux split-tunneling support.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | WireGuard, Linux network namespaces, systemd, nftables, gluetun (Docker) |
| **Impact** | Needed P2P traffic isolated behind a VPN without touching SSH/Tailscale/WebUI routes on the same host |
| **Status** | ✅ Resolved, pattern replicated across multiple nodes |

---

## 🔍 Goal / Why Not System-Wide

The VPN client in use has no Linux split-tunneling support, so isolating only P2P traffic (leaving SSH/Tailscale/WebUIs on the host's normal route) is built instead as a dedicated **network namespace + WireGuard config per P2P instance**. The management/SSH path is deliberately excluded from any VPN rollout: only the specific P2P process gets tunneled.

## 🎯 Architecture

- A systemd service (`qbtvpn-netns.service`) creates the namespace and brings up WireGuard inside it via `NetworkNamespacePath`, driven by root-only (mode 700) up/down scripts.
- The up-script adds a host-side MASQUERADE rule for the namespace's internal subnet and sets up the netns-internal default route needed for WireGuard's handshake.
- The torrent service is configured to join that namespace, so 100% of its traffic (and only its traffic) goes through the tunnel.
- For a Docker-based app, the equivalent is `NetworkMode: container:<gluetun-container-id>`: the app shares gluetun's netns entirely, with gluetun's own killswitch blocking traffic outright if the tunnel drops rather than passing it in the clear.
- Never reuse one device's WireGuard identity/private key on a second device: one active session per key is expected, and reusing a key elsewhere has directly caused an SSH-over-overlay-network outage before. Each new instance gets its own reserved identity/config.

## 🚫 Recurring Failure Modes and Their Fixes

1. **Killswitch blocks all egress, and status reporting lies about it.** Total host egress failure (`Destination Port Unreachable`, `sendmsg: Operation not permitted`) looks like a routing/netfilter problem, and the VPN client's own status command can report the firewall as off while the block is still active: don't trust it. Fix: force the firewall into manual mode and off, then check for a leftover nftables table that can persist with only empty chains, and delete it. On a node where the namespace doesn't depend on the VPN client's daemon at all, disable and mask that daemon's helper service entirely for a durable fix.

2. **Tunnel comes up but never handshakes (0 B received) after a reboot.** The up-script's MASQUERADE rule can vanish after boot even though the script adds it unconditionally; re-adding the NAT rule alone isn't always enough if a stale, un-NATed UDP conntrack flow is being kept alive by WireGuard's own retry behavior. Fix: force a new flow by changing the runtime listen-port, which breaks the stale conntrack entry and lets a fresh, correctly-NATed handshake happen.

3. **The namespace service crashes at boot on a DNS resolution failure**, then sits dead indefinitely with no retry. Root cause: the unit only declared `After=network.target` (network stack present, not necessarily DNS-ready): a boot-ordering race against DNS coming up. Fix: change to `After=network-online.target` + `Wants=network-online.target`, and add `Restart=on-failure` + `RestartSec=5` as a safety net for any future race.

4. **A dependent WebUI stays silently broken for a long time** before anyone notices, because the failure mode is "service crashed, no retry" rather than an obvious alarm. Now always checking `systemctl status` on the namespace unit specifically, not just whether the WebUI is reachable, when a torrent WebUI suddenly resets connections.

## ✅ Verification

Checked every time a new instance goes up, not just on first build:
- WebUI reachable only via the namespace's internal veth address, not the host's normal IP.
- Exit IP seen externally is the VPN provider's, not the box's real IP.
- No IPv6 leak (a separate check from the IPv4 exit-IP check).
- Any Host-header check on the WebUI is satisfied through the tunnel path, not just the direct path.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
