# `fix/multi-uav-freeze` Patch — Notes, Build and Test

**What it fixes:** the 30–60 s full-UI freeze on Arm/Takeoff/Set WP with multiple vehicles on a shared link, and a `giveComport` leak that could permanently kill telemetry (root cause analysis: [ANALYSIS.md](ANALYSIS.md)).

---

## 1. Changes

### A) `ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs` — `giveComport` is now released on every exit path

The ACK wait loops in `doCommandAsync`, `doCommandIntAsync` and `setWPCurrentAsync` are wrapped in `try/finally`; `giveComport = false` now lives in the `finally`. Previously, if `readPacketAsync`/`generatePacket` threw during the wait (link drop etc.), the flag stayed `true`, `MainV2.SerialReader` stayed paused forever and **telemetry for every vehicle on the link died until reconnect**. No behavioral change otherwise; only the exception paths are now safe.

### B) `GCSViews/FlightData.cs` — 3 handlers converted to async

| Handler | Before | After |
|---|---|---|
| `BUT_ARM_Click` | `doARM(...)` — blocked the UI thread up to 40 s | `await doARMAsync(...)`; the button is disabled while the command runs; the force-arm flow and the STATUSTEXT subscription are preserved (the subscription is now released in a `finally`) |
| `takeOffToolStripMenuItem_Click` | `doCommand(TAKEOFF)` — up to 8 s block | `await doCommandAsync(TAKEOFF)` |
| `BUT_setwp_Click` | `setWPCurrent(...)` — up to 12 s block | `await setWPCurrentAsync(...)`; button re-enable moved to `finally` |

The pattern is exactly the one already used elsewhere in MP (`modifyandSetSpeed_Click`, `setHomeHereToolStripMenuItem_Click` — `await ...Async(...).ConfigureAwait(true)`).

Extra hardening: `BUT_ARM_Click` captures the target `sysid/compid` **before any await** — switching the selected vehicle mid-command can no longer redirect the command (previously the force-arm second call went to whatever was selected at that moment).

### New user-visible behavior

- You press Arm, the ACK gets lost → **the UI does not freeze**; the other vehicles' HUDs stay live; after 40 s you get the same "No response" box (timeouts were deliberately left unchanged — only the waiting moved off the UI thread).
- While waiting, the Arm button is greyed out; everything else remains usable.

---

## 2. Known limits / deliberately out of scope

1. **Concurrent commands:** since the UI no longer freezes, you *can* press other command buttons during a wait. MP's command layer does not support two simultaneous ACK waits (`giveComport` contention → the second command may miss its ACK and report "failed"). This limitation already existed for MP's stock async handlers; it now fails softly instead of freezing. Practical rule: don't fire a second command on the same link before the first resolves.
2. **Handlers not converted (yet):** `BUTactiondo_Click` (Actions/Do Action, ~200-line multi-branch handler), `BUT_resumemis_Click` (busy-wait + `DoEvents` flow needs a redesign), `BUTrestartmission_Click`, `BUT_mountmode_Click`. They still block the UI thread for 2–12 s; fix (A) protects their exception paths regardless. They can be converted with the same pattern — future work.

---

## 3. Building from source

The tree builds without Visual Studio using only the .NET SDK (8.x), even on machines without .NET Framework targeting packs — a `Directory.Build.targets` at the repo root resolves the net472 reference assemblies from NuGet:

```bash
# order matters: DriverCleanup first (main csproj copies its exe as Content)
dotnet build ExtLibs/DriverCleanup/DriverCleanup.csproj -c Debug
dotnet build MissionPlanner.csproj -c Debug
# output: bin/Debug/net461/MissionPlanner.exe
```

Alternatively: Visual Studio 2022 (".NET desktop development" workload), open `MissionPlanner.sln`, build the `MissionPlanner` project.

To apply the patch to another checkout instead: `git am mp-freeze-fix.patch` (based on upstream master `a2fcd74`).

---

## 4. Test plan

### SITL (no hardware, safe)

1. Start 5 ArduCopter SITL instances, all `--out` to the same GCS UDP port (`SYSID_THISMAV` 1–5).
2. Patched MP → UDP listen → 5 vehicles appear on one connection.
3. **Freeze test:** suspend one SITL process → Arm that vehicle → expected: no UI freeze, Arm button greyed ~40 s, other vehicles' HUDs keep updating, then the "No response" box. Stock MP freezes solid for 40 s on the same test — the difference is unmistakable.
4. **Happy path:** Arm/Takeoff/Set WP on a live vehicle → identical behavior to stock MP (success/reject messages, force-arm dialog included).
5. **Leak test:** kill a SITL completely while its command is waiting → after the exception, the other vehicles' telemetry must keep flowing (stock MP could leave `giveComport` stuck and kill all telemetry).

### Field (after SITL passes)

Start with 2 vehicles, low risk; exercise arm-reject and weak-link scenarios on the ground (props off) before scaling to 5.
