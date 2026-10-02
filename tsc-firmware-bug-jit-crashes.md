# 🔧 TSC Firmware Bug Causing Deterministic JIT Crashes

> *A "random" browser crash that turned out to be a CPU firmware bug, hidden by a kernel boot flag that was supposed to help.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | Multi-socket-capable Xeon-class workstation, Linux kernel clocksource/TSC subsystem, JIT-compiling application (browser JS engine) |
| **Impact** | Repeated SIGILL crashes in a daily-driver browser, always at the identical instruction pointer |
| **Status** | ✅ Resolved |

---

## 🔍 Symptom

A JIT-compiling application (in this case, a browser's JS engine) kept crashing with `SIGILL` (invalid opcode), and every crash landed at the **exact same instruction pointer** across separate runs. That determinism is the tell: this isn't random memory corruption, it's a specific JIT-compiled code path being corrupted the same way every time. GPU driver upgrades and explicit-sync fixes left it completely untouched, which ruled out the first instinct (a graphics stack problem).

## 🚫 Ruled Out

- GPU/driver bugs: upgrading drivers and toggling explicit-sync made zero difference.
- Generic memory corruption: the crash address was identical across runs, which generic corruption wouldn't produce.

## 🎯 Root Cause

`dmesg` had the answer the whole time:

```
[Firmware Bug]: TSC ADJUST: CPU0: <large-negative-number> force to 0
[Firmware Bug]: TSC ADJUST differs within socket(s), fixing all errors
```

A genuine motherboard/CPU firmware bug causing TSC (Time Stamp Counter) desync across sockets/cores at boot, more common on multi-socket-capable workstation chips than consumer single-socket CPUs. Linux normally has a watchdog that detects an unstable TSC and falls back to a slower, reliable clocksource (HPET) on its own. That watchdog's "Marking TSC unstable" message was missing here because `/etc/default/grub` had `tsc=reliable` set in `GRUB_CMDLINE_LINUX_DEFAULT`, explicitly telling the kernel to skip the check and trust a TSC that firmware itself had already flagged as broken.

Before calling this confirmed, I corroborated it with more than the one dmesg line: repeated "Time jumped backwards, rotating" events in the journal (not a one-off), crash timestamps lining up with those backward jumps, and an impossible-future kernel timestamp elsewhere in dmesg: kernel timestamps are themselves derived from the clocksource, so a bad value there is independent confirmation the clock itself was lying.

## 🛠️ Fix

1. Remove `tsc=reliable` from `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub` (leaving unrelated flags like `nvidia-drm.modeset=1` alone).
2. `sudo update-grub`
3. `sudo reboot`

## ✅ Verification

```bash
dmesg -T | grep -iE "marking tsc unstable|clocksource.*hpet"
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
```

Confirmed the watchdog engaged and the active clocksource switched away from `tsc`. Ran the previously-crashing app normally afterward and checked `coredumpctl` for any new SIGILL entries (none) before calling it closed.

---

*Part of the [lab-troubleshooting](README.md) collection, real incidents from a live, multi-node home lab, documented as they happened.*
