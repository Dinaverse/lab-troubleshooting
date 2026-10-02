# 🔧 i3 + NVIDIA Desktop Stability (Freeze/Crash Red Herrings)

> *Two "app crashes" with no coredump and no log entry — both turned out to be the same window manager/GPU rendering desync, not the application.*

---

## 📋 Incident Snapshot

| Field | Detail |
|---|---|
| **Stack** | i3 (X11), NVIDIA GTX 1060, emulator/app windows (RPCS3/Dolphin/Cemu-class) |
| **Impact** | Recurring mouse freezes and "crashed" app windows that were actually alive the whole time |
| **Status** | ✅ Resolved — two confirmed fixes, one standing fast-path |

---

## 🔍 Symptom

A mouse freeze, or an app window that appears frozen/stopped rendering after a workspace switch, with no coredump and nothing useful in the crash log — the process is still alive, it's just not rendering where expected.

## 🚫 Ruled Out

Picom compositor config tweaks were tried first for the stale-window symptom — four different configurations (GLX vs. xrender, `use-damage` off, `unredir-if-possible` off) — and confirmed to make **no difference**. Picom was a red herring for this specific symptom and was ruled out before looking at the window manager/driver layer.

## 🎯 Root Cause(s)

Two independent, confirmed causes account for most "crash"/freeze reports on this desktop:

1. **i3 output-pinning desync.** Pinning all workspaces to a single monitor output caused an i3 output/rendering desync where a window's process stayed alive but simply stopped rendering on the correct output — indistinguishable at a glance from a crash.
2. **Stale-window bleed-through on workspace switch.** Switching workspaces left emulator windows showing stale content (looks frozen, isn't) — an NVIDIA explicit-sync interaction, not a compositor problem.

## 🛠️ Fix

**Standing fast-path for mouse issues:** apply this immediately, before further diagnosis:
```
DISPLAY=:0 i3-msg restart
```
Verified with an `xdotool` warp test (move, then read back position, expect an exact match). If the first restart doesn't resolve it, a second one has been needed more than once — worth running before digging further.

**Output-pinning desync:** check whether workspaces are pinned to a single monitor output; reverting that plus `i3-msg restart` resolves it with no reboot needed.

**Stale-window bleed-through:** an NVIDIA driver environment variable, added to `~/.xprofile`:
```
export __NV_DISABLE_EXPLICIT_SYNC=1
```

## ✅ Verification

Verified the real fix with repeated screenshot comparisons across a workspace switch, rather than a single visual glance. General principle adopted for this desktop going forward: check process state first (still running? any coredump?) before assuming an app is broken — both confirmed root causes here were i3/NVIDIA rendering issues, not the application itself. Also worth confirming current driver version against known driver-bug lists before digging into app-specific logs — a prior GPU crash across multiple apps was fixed purely by a driver upgrade, no app-side change needed.

---

*Part of the [lab-troubleshooting](README.md) collection — real incidents from a live, multi-node home lab, documented as they happened.*
