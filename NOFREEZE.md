# Mission Planner — NoFreeze Fork

A fork of the official [ArduPilot/MissionPlanner](https://github.com/ArduPilot/MissionPlanner) that **fixes the UI freeze when operating multiple vehicles**.

**The problem:** When flying several UAVs over a single shared link (e.g. 5 drones streaming into one UDP port), commands like Arm/Takeoff/Set WP **froze the entire UI for 30–60 seconds** while waiting for the vehicle's acknowledgement — and telemetry processing for *every* connected vehicle stalled at the same time.

**The fix (branch: `fix/multi-uav-freeze`):**
- Arm/Disarm, Takeoff and Set WP now send their commands **asynchronously**: the UI never freezes, only the clicked button is disabled while the command is in flight, and all success/error messages behave exactly as before.
- Fixed a `giveComport` leak: a connection error during a command could previously leave the link's reader paused forever, killing telemetry for every vehicle until reconnect (now guarded by `try/finally`).
- Command timeouts are intentionally unchanged: a command to an unreachable vehicle still reports "No response" after ~40 s — it just no longer freezes anything while waiting.

Technical deep dive: [docs-nofreeze/ANALYSIS.md](docs-nofreeze/ANALYSIS.md) (root cause analysis, side-by-side comparison with QGroundControl) and [docs-nofreeze/PATCH-NOTES.md](docs-nofreeze/PATCH-NOTES.md) (change list, build and test notes).

---

## Install — Linux (no build required)

A pre-built package is available on the **[Releases](../../releases)** page. Mission Planner runs on Linux via Mono:

```bash
# 1) Install Mono (Ubuntu/Debian; Mono >= 6 recommended)
sudo apt update
sudo apt install -y mono-complete unzip

# 2) Download the zip from the Releases page, then:
unzip MissionPlanner-NoFreeze-*.zip -d ~/MissionPlanner-NoFreeze

# 3) Run
cd ~/MissionPlanner-NoFreeze
mono MissionPlanner.exe
```

Notes:
- First start can be slow (Mono JIT); settings are stored under `~/Documents/Mission Planner/`.
- Connecting: pick **UDP** in the top-right corner → Connect → enter your port. All vehicles streaming into that port appear on one connection and are selectable from the vehicle drop-down.
- Video streaming is optional (`sudo apt install -y gstreamer1.0-tools gstreamer1.0-plugins-good` etc.); not needed for GCS functions.
- If anything misbehaves, the console output in the terminal shows the error directly.

## Install — Windows

Download the zip from Releases → extract to a folder → double-click `MissionPlanner.exe`. (Do **not** run it at the same time as an officially installed Mission Planner — they share the same settings folder.)

## Updating

When a new release is published, just download the new zip and extract it over the old folder. Your settings live in `Documents/Mission Planner`, so they are preserved.

## Disclaimer

This is an experimental build. Verify it in SITL or on the bench (props off) before using it as your primary GCS in field operations. Base: upstream master `a2fcd74` (2026-07-11) plus the patch commits; built in Debug configuration.
