# 🔧 Remote-Controlling an iOS 17+ Device via pymobiledevice3

> *iOS 17 quietly broke the classic developer-tools stack — the old commands fail even when everything else is set up right, because the whole protocol underneath changed.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | pymobiledevice3, iOS 17+, Linux host (USB-connected) |
| **Impact** | Needed remote diagnostics/screenshot access to a device with a broken touchscreen; classic tooling failed outright |
| **Status** | ✅ Resolved |

---

## 🔍 Symptom

The classic `libimobiledevice` tools (`idevicescreenshot`, etc.) fail on iOS 17+ with "Could not start screenshotr service," regardless of whether the developer disk image is correctly mounted. Apple replaced the classic developer-services architecture starting with iOS 17 — the old protocol simply isn't there anymore.

## 🎯 Root Cause / Fix

Use `pymobiledevice3 developer dvt <command>` instead, which speaks the new iOS 17+ remote developer tunnel protocol. Confirmed working as the first end-to-end sanity check: `pymobiledevice3 developer dvt screenshot <output.png>` — pulls a live screenshot over USB, proving the whole chain (pairing, dev mode, disk image, tunnel) actually works before attempting anything more complex.

**Important scope limit:** `developer dvt` commands are diagnostics/process-control only — screenshot, app list, launch/kill process, monitoring. They do **not** include simulated tap/swipe input. Screenshot access does not imply UI control; actual touch simulation needs a separate WebDriverAgent (WDA) setup — real additional work, not a flag away.

## 🛠️ Setup Sequence

1. `pip3 install --user pymobiledevice3` on the Linux host the device is physically connected to (add `$HOME/.local/bin` to `PATH`).
2. Enable Developer Mode: `pymobiledevice3 amfi enable-developer-mode`, run it **twice** — once to trigger the device reboot, once after reboot to confirm. The Settings > Privacy & Security > Developer Mode toggle is hidden by default on stock iOS and won't appear from a USB connection alone — this command is what surfaces it, not fiddling in Settings.
3. Mount the developer disk image: `pymobiledevice3 mounter auto-mount`.
4. Verify with the `dvt screenshot` command above before building anything on top of it.

## ✅ Other Notes

- A full device backup (`pymobiledevice3 backup2 backup --full <dest>`) should target a volume with real free space — a backup can be much larger than it looks; check available space first rather than letting it fail partway.
- Wireless (WiFi-based) `remote pair`/`remote browse` access is worth attempting but has been unreliable even when the device's mDNS record looks correct — a known discovery quirk, not necessarily a config problem worth chasing. USB with the device plugged into a host is the reliable fallback.
- For a device with a broken touchscreen specifically: AssistiveTouch plus a USB mouse/keyboard (enable AssistiveTouch via Siri voice command if the touchscreen is too broken to navigate Settings at all) is the standing on-device workaround.

---

*Part of the [lab-troubleshooting](README.md) collection — real incidents from a live, multi-node home lab, documented as they happened.*
