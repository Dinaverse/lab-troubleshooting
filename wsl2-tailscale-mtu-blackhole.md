# 🔧 WSL2 + Tailscale Direct-Peer Large-Packet Blackhole

> *Pings were clean, the service was healthy, and the tunnel still died because the problem only shows up once real data tries to flow.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | WSL2 (Ubuntu) on Windows, Tailscale overlay network, autossh/SSH tunnel |
| **Impact** | Intermittent tunnel death to a direct (same-LAN) Tailscale peer; a DERP-relayed peer stayed fine throughout |
| **Status** | ✅ Resolved, persistent fix in place |

---

## 🔍 Symptom

An autossh/SSH tunnel running inside WSL2 to a Tailscale peer intermittently died:

```
autossh: Timeout, server <peer-ip> not responding.
ssh exited with error status 255; restarting ssh
```

or a request through the tunnel hung and returned 0 bytes, even though `ping` to the peer's Tailscale IP was clean, `tailscale status` showed the peer `active; direct <lan-ip>:41641` (NAT traversal succeeded), and the destination service was confirmed healthy when tested directly on that peer. A different tunnel to a host that showed `relay "<derp-node>"` instead of `direct` kept working the whole time.

## 🚫 Ruled Out

- The destination's NIC/driver checksum offload: disabling `tx/rx/tso/gso/gro` offload on the peer made no difference (tested on two different NIC drivers).
- The physical Windows host's NIC or the home router: a native Windows ping (outside WSL2) with the same oversized payload to the same peer succeeded cleanly, isolating the problem to WSL2's own virtual networking stack.
- WSL2 "mirrored" networking mode as a fix: it requires Windows 11 22H2+; on Windows 10, NAT mode is mandatory and the MTU workaround below is the only practical fix.
- Checksum/segmentation offload inside the WSL guest's own interface: also tried, no effect. The blackhole is specific to the tailscale0/WireGuard-encapsulated path, not generic WSL networking.

## 🎯 Root Cause

WSL2's own Tailscale node (a separate identity from the Windows host's Tailscale client) has a path-MTU blackhole specific to **direct** (same-LAN) peer connections: any packet over roughly 1150–1200 bytes is silently dropped, while small packets (ICMP, early SSH handshake packets) pass fine. It only surfaces once real data needs to flow, which is why a ping alone looks healthy. Confirmed directly:

```bash
for s in 1300 1200 1100 1000; do
  echo -n "size $s: "; ping -M do -s $s -c 3 -W1 <peer-tailscale-ip> 2>&1 | grep -c "bytes from"
done
```

A clean cutoff (e.g. 1100 passes, 1200 fails) confirms it. Testing the same against a `relay`-connected peer as a control passed at full size, confirming the relay path wasn't affected.

## 🛠️ Fix

Lower `tailscale0`'s MTU inside the WSL distro below the blackhole threshold:

```bash
ip link set dev tailscale0 mtu 1100
```

Made durable with a oneshot systemd unit (requires `systemd=true` under `[boot]` in `/etc/wsl.conf`):

```ini
# /etc/systemd/system/tailscale-mtu-fix.service
[Unit]
Description=Lower tailscale0 MTU to work around WSL2 NAT large-packet blackhole
After=tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 3
ExecStart=/sbin/ip link set dev tailscale0 mtu 1100
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload
systemctl enable --now tailscale-mtu-fix.service
```

## ✅ Verification

Fully restarted the WSL instance (`wsl.exe -t <DistroName>`) and re-checked `ip link show tailscale0 | grep mtu`: the lowered value was already applied within seconds of boot, with no manual intervention.

### A confusing side-effect worth flagging

While debugging, repeatedly restarting the tunnel service to retest can trip the destination's `PerSourcePenalties`/`srclimit` feature (OpenSSH 9.8+): each failed handshake (caused by the still-unfixed blackhole) counts as "exceeded LoginGraceTime" and escalates a temporary penalty against the source IP. Once the real fix is in but the penalty hasn't cleared, fresh attempts get actively reset rather than timing out, which can look like the fix didn't work. Check the destination's auth log for `penalty`/`LoginGraceTime` lines before concluding otherwise, and test with one manual foreground `ssh` attempt rather than fighting a systemd service's rapid auto-restart loop.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
