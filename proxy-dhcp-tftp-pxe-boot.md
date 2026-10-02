# 🔧 Adding PXE Network-Boot Behind a Consumer Router

> *The router had no PXE/TFTP options to configure, so the fix was a second, cooperative DHCP server, not a replacement for the one already running.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | dnsmasq (proxy-DHCP + TFTP), netboot.xyz, consumer/ISP router |
| **Impact** | Target device couldn't network-boot at all (`PXE-E53: no boot filename received`) |
| **Status** | ✅ Resolved |

---

## 🔍 Symptom

A device trying to PXE/network-boot failed with `PXE-E53: no boot filename received`, because the router didn't expose PXE/TFTP DHCP options (60/93/94/97) anywhere in its UI; there was no way to fix this on the router itself.

## 🎯 Root Cause / Approach

The fix is a **proxy-DHCP** server, not a replacement DHCP server. Proxy-DHCP answers only PXE-specific boot requests; it does not hand out IP addresses. The router's own DHCP keeps assigning IPs completely undisturbed; the proxy-DHCP server answers the separate "what do I boot" question alongside it. This is the standard way to add PXE to a network whose router can't be configured for it, without replacing or fighting the existing DHCP server.

## 🛠️ Fix

Built on a lightweight LXC/VM, confirmed to be on the same subnet as the router and the booting device first. `dnsmasq` does both proxy-DHCP and TFTP in one daemon; no separate TFTP server needed. Boot files came from `netboot.xyz` (an iPXE-based menu that lets you choose what to do with a booting device interactively, useful when you don't want to commit to one OS image in advance), both firmware variants since the target's firmware type wasn't known in advance:

```
# to /var/lib/tftpboot/
netboot.xyz.kpxe   # legacy BIOS PXE
netboot.xyz.efi    # UEFI
```

`/etc/dnsmasq.conf`:
```
interface=eth0
bind-interfaces
port=0
dhcp-range=<subnet>,proxy,<netmask>
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-match=set:efi-x86_64,option:client-arch,9
dhcp-boot=tag:efi-x86_64,netboot.xyz.efi
dhcp-boot=tag:!efi-x86_64,netboot.xyz.kpxe
enable-tftp
tftp-root=/var/lib/tftpboot
pxe-service=x86PC,"Network Boot",netboot.xyz.kpxe
pxe-service=X86-64_EFI,"Network Boot (UEFI)",netboot.xyz.efi
log-dhcp
```

The `client-arch` tags auto-detect BIOS vs. UEFI per boot request via DHCP option 93, so the correct boot file is served automatically.

## ✅ Verification

Checked the server itself before testing against the real device:
```bash
dnsmasq --test                          # syntax check
systemctl status dnsmasq                # active (running)
ss -ulnp | grep dnsmasq                 # listening on :67 (proxy-DHCP), :69 (TFTP), :4011 (PXE)
```
Then triggered PXE boot on the target device (F12/F9/Esc at startup) and watched the request arrive live with `journalctl -u dnsmasq -f` while powering it on. If no DHCPDISCOVER/PXE request shows up at all, that points to a network isolation issue upstream (e.g. AP/client isolation mode on the router) rather than a dnsmasq config problem.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
