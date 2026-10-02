# 🧰 Lab Troubleshooting

> *Real incidents from a live, multi-node home lab, root-caused and fixed, documented as they happened.*

Every entry here started as an actual outage or a broken workflow somewhere in the lab (Proxmox host, Kali-Master, Arch-GPU, Dell-Gateway, or a connected desktop/laptop), not a hypothetical. Each one is written up the way it was actually diagnosed: symptom, what got ruled out, root cause, fix, and how it was verified.

---

## 📚 Quick Navigation

| Category | Incident | Status |
|---|---|---|
| Kernel & Storage | [Diagnosing a Wedged Kernel Storage Thread (D-State)](diagnosing-wedged-kernel-storage.md) | ✅ Resolved |
| Kernel & Storage | [Self-Healing USB Drive Remount via udev](udev-usb-drive-auto-remount.md) | ✅ Resolved |
| Networking | [WSL2 + Tailscale Direct-Peer Large-Packet Blackhole](wsl2-tailscale-mtu-blackhole.md) | ✅ Resolved |
| Networking | [Adding PXE Network-Boot Behind a Consumer Router](proxy-dhcp-tftp-pxe-boot.md) | ✅ Resolved |
| Virtualization, Containers & Security | [Unprivileged LXC: VPN Daemon Won't Start (Missing /dev/net/tun)](proxmox-lxc-tun-device-passthrough.md) | ✅ Resolved |
| Virtualization, Containers & Security | [Dropping a systemd Service From Root to Least-Privilege](systemd-privilege-drop-capabilities.md) | ✅ Resolved |
| Virtualization, Containers & Security | [Per-Process VPN Isolation via Network Namespaces](qbtvpn-per-process-vpn-isolation.md) | ✅ Resolved |
| Desktop & Devices | [TSC Firmware Bug Causing Deterministic JIT Crashes](tsc-firmware-bug-jit-crashes.md) | ✅ Resolved |
| Desktop & Devices | [i3 + NVIDIA Desktop Stability (Freeze/Crash Red Herrings)](kali-nvidia-desktop-stability.md) | ✅ Resolved |
| Desktop & Devices | [Remote-Controlling an iOS 17+ Device via pymobiledevice3](pymobiledevice3-ios17-remote-control.md) | ✅ Resolved |

---

## ✅ Operational Status

| Incident | Root Cause Class | Status |
|---|---|---|
| Wedged kernel storage thread | Kernel D-state wedge on USB I/O | ✅ Resolved |
| USB drive re-enumeration | Stale device-path mount | ✅ Resolved |
| WSL2/Tailscale tunnel drops | Path-MTU blackhole | ✅ Resolved |
| PXE boot failure | Router lacks PXE/TFTP DHCP options | ✅ Resolved |
| LXC VPN daemon won't start | Missing /dev/net/tun passthrough | ✅ Resolved |
| Service running as root | AmbientCapabilities gap, setcap fix | ✅ Resolved |
| P2P traffic leaking host route | Network-namespace isolation | ✅ Resolved |
| Deterministic browser SIGILL | TSC firmware bug masked by `tsc=reliable` | ✅ Resolved |
| Desktop freeze/crash reports | i3/NVIDIA rendering desync | ✅ Resolved |
| iOS 17+ remote diagnostics | Developer-services protocol change | ✅ Resolved |

---

## 🎯 Philosophy

Every failure here got root-caused, not just restarted away. The goal of this repository is the same as the rest of the lab: every decision documented, every failure treated as a learning opportunity, nothing left as "it just started working again."

---

*Part of the [Dinaverse](https://github.com/Dinaverse) ecosystem, documented as incidents happened, not reconstructed after the fact.*
