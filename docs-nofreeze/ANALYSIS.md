# Mission Planner vs QGroundControl — Multi-Vehicle Freeze: Root Cause Analysis

**Scenario:** 5 drones, one shared UDP port (14550), Microhard mesh modem (good on average, occasionally lossy link).
**Symptom:** In Mission Planner nearly every button (Arm/Takeoff/…) hard-freezes the UI for 30–60 s; in QGroundControl there is never a freeze — if a command doesn't get through, a clear error appears within ~1.5 s.

**Code examined:** `MissionPlanner` master `a2fcd74` (2026-07-11), `qgroundcontrol` master `5bc5dfb` (2026-07-10). All line numbers below were verified against these trees.

---

## 0. Executive Summary

The difference is not in the UDP/network layer — it is the **command-wait architecture**:

| | Mission Planner | QGroundControl |
|---|---|---|
| Sending a command | **Synchronous wait on the UI thread** (`AwaitSync`) | Async queue (`MavCommandQueue`), call returns immediately |
| ARM with no reply | 10 s timeout × 4 sends = **~40 s UI freeze**, then error | 1.2 s timeout × 1 try = **error in ~1.5 s** |
| Other 4 vehicles during the wait | `giveComport` pauses the link reader → **telemetry processing for all vehicles stalls** | Per-vehicle queues/timers; others **completely unaffected** |
| Error presentation | Modal MessageBox (after the freeze ends) | Non-modal QML dialog (immediately) |
| Who receives a UDP command | **All** endpoints seen on the port (broadcast) | **All** endpoints seen on the port (broadcast) — **identical**, not the difference |

The observed "30–60 s freeze" falls straight out of the constants in the code: **ARM = 10 000 ms × (1 initial + 3 retries) = 40 s** (Section 2). QGC's "instant" error also falls out of constants: **1 200 ms ack timeout, no retry for arm, 500 ms checker timer → 1.2–1.7 s** (Section 5).

---

## 1. Anatomy of an "Arm" Click in Mission Planner

Verified chain:

1. `BUT_ARM_Click` — WinForms click handler, runs **on the UI thread** → [GCSViews/FlightData.cs:1057](../GCSViews/FlightData.cs)
   ```csharp
   bool ans = MainV2.comPort.doARM(!isitarmed);   // UI thread, synchronous
   ```
2. `doARM` → `doARMAsync(...).AwaitSync()` → [ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs:2629](../ExtLibs/ArduPilot/Mavlink/MAVLinkInterface.cs)
3. `AwaitSync` = `Task.Run(...).GetAwaiter().GetResult()` → **blocks the calling (UI) thread until done** → [ExtLibs/Utilities/Extensions.cs:110-118](../ExtLibs/Utilities/Extensions.cs)
4. Inside `doCommandAsync`: `giveComport = true` → [MAVLinkInterface.cs:2716] — see Section 3.
5. ACK wait: a `while(true)` loop calling `readPacketAsync()` inline → [MAVLinkInterface.cs:2775-2800]
6. No reply: resend 3 times on timeout, then `TimeoutException` → [MAVLinkInterface.cs:2786-2797]
7. The exception lands in `BUT_ARM_Click`'s `catch` → the "No response" MessageBox appears **after** the freeze ([FlightData.cs:1076-1079]).

WinForms has a single UI thread: for those 40 s the window cannot repaint and no click is processed. **Every click made during the freeze is queued and fires after it ends** — most of them starting yet another blocking handler. That is the mechanical explanation of "every button freezes".

---

## 2. Timeout Arithmetic — Where "30–60 s" Comes From

`doCommandAsync` constants ([MAVLinkInterface.cs:2729-2768]):

```csharp
int retrys = 3;
int timeout = 2000;                                       // default
...
else if (actionid == MAV_CMD.COMPONENT_ARM_DISARM)
{
    // 10 seconds as may need an imu calib
    timeout = 10000;                                      // ARM-specific
}
```

The retry loop ([MAVLinkInterface.cs:2784-2797]) resends with `confirmation++` when the window expires, resets `start`, decrements `retrys`; when none remain it throws `TimeoutException`.

| Command | Timeout | Tries | **Worst-case UI freeze** |
|---|---|---|---|
| ARM / DISARM | 10 000 ms | 1+3 | **40 s** |
| "Force Arm" confirmed afterwards | 10 000 ms | 1+3 | **+40 s** (dialog in between) |
| TAKEOFF (no special case → default) | 2 000 ms | 1+3 | **8 s** |
| `setWPCurrent` (restart mission / set WP) | 2 000 ms | 1+5 | ~12 s |
| `setParam` (each parameter write) | 700 ms | 1+3 | ~2.8 s |
| `BUT_resumemis_Click` (busy-wait loops + `Application.DoEvents`) | — | — | 30+ s |

On a lossy link, losing a single ACK datagram is enough: press Arm → 40 s freeze. A Takeoff clicked during the freeze queues up → +8 s… The observed 30–60 s window matches one ARM timeout (40 s) ± queued blocks.

Note: mode changes (`setMode`) are actually **non-blocking** (no ACK wait, `requireack=false` — [MAVLinkInterface.cs:4636]); sluggish mode buttons are collateral damage from an already-frozen UI or another in-flight blocking command.

---

## 3. `giveComport` — One Command Stalls Five Vehicles' Telemetry

A link has a single background reader: `MainV2.SerialReader`. While a command waits for its ACK, `giveComport=true` and the reader is **fully disabled**:

[MainV2.cs:3008] and [MainV2.cs:3046-3047]:
```csharp
// if not connected or busy, sleep and loop
if (!comPort.BaseStream.IsOpen || comPort.giveComport == true) { ... await Task.Delay(100)... }
...
while (port.BaseStream.IsOpen && port.BaseStream.BytesToRead > minbytes &&
       port.giveComport == false && serialThread && ...)
{
    await port.readPacketAsync()...
}
```

The 5 drones on one UDP port live inside **one `MAVLinkInterface`**, in its `MAVlist` table (sysid → `MAVState`, [MAVList.cs:10]). While `giveComport=true`:

- SerialReader pulls nothing → **HUD/telemetry processing for all 5 vehicles stops**;
- The only reader is the blocked command's own inline `readPacketAsync` ([MAVLinkInterface.cs:2800]): it parses all five vehicles' traffic one packet at a time, hunting for a single ACK;
- The UI thread is blocked anyway, so nothing repaints.

Architecturally: **one unanswered command to one drone holds the whole window and the whole fleet's data stream hostage for 40 s.**

---

## 4. Side Bug — `giveComport` Leak (Permanent Telemetry Loss)

There is **no `try/finally`** around `doCommandAsync`'s wait loop. The timeout path clears the flag ([2796-2797]), but if `readPacketAsync()` (line 2800) throws (stream error, link drop), the exception escapes with `giveComport=true` still set. Result: SerialReader stays paused indefinitely → **telemetry for all 5 vehicles is dead until reconnect**. The same pattern exists in the sibling methods `doCommandIntAsync`, `setWPCurrentAsync`, `setParamAsync`. This likely explains the occasional "longer than 40 s, never recovers" hangs. (The patch fixes this — see PATCH-NOTES.)

---

## 5. The Other Shore: the Same Click in QGC

In QGC master the command logic lives in `MavCommandQueue` (delegated from Vehicle, [src/Vehicle/Vehicle.cc:2144]):

1. QML arm button → `Vehicle::setArmed(true, showError=true)` → `sendMavCommand(MAV_CMD_COMPONENT_ARM_DISARM…)` ([Vehicle.cc:1437]) → queue entry (`MavCommandListEntry_t {maxTries, ackTimeoutMSecs, QElapsedTimer…}`, [MavCommandQueue.h:76]). **The call returns immediately; the UI stays free.**
2. A free-running 500 ms `QTimer` (`_responseCheckTimer`, [MavCommandQueue.cc:24, 188]) processes entries whose `elapsed > ackTimeoutMSecs` (production value **1 200 ms**).
3. **No retry for ARM** — `_shouldRetry(command)` returns true only for state-query commands; the code comment is explicit: you don't want an arm to time out twice and then suddenly work 6 seconds later ([MavCommandQueue.cc:200]). `maxTries = 1`.
4. When tries are exhausted: `commandResult(MAV_RESULT_FAILED, NoResponseToCommand)` + `QGC::showAppMessage(tr("Vehicle did not respond to command: %1")…)` ([MavCommandQueue.cc:348]) → a **non-modal** QML dialog ([QGCApplication.cc:431]).

**Timeline (no ACK at all):** t=0 send → 500/1000 ms ticks below threshold → first tick past 1 200 ms gives up + error dialog. **The user sees the error in ~1.5 s; the UI never blocks.**

Threading and isolation:

- UDP socket IO runs in a `UDPWorker` on its own `QThread`; data crosses to the main thread only via `Qt::QueuedConnection` ([UDPLink.cc:509-527]); writes are queued main→worker too ([UDPLink.cc:594], [LinkInterface.cc:93]) — **no synchronous socket wait anywhere on the command path**.
- Each sysid gets its own `Vehicle` object (heartbeat → `MultiVehicleManager`, [MultiVehicleManager.cc:103]); every Vehicle drops messages that aren't its own sysid ([Vehicle.cc:519]); command queue and comm-lost detection (3.5 s, [VehicleLinkManager.h:84]) are **per vehicle**. One drone's unanswered command mathematically cannot affect the other four.

---

## 6. UDP Targeting: NOT the Difference (a popular suspicion, eliminated)

Both programs record **every** source endpoint seen on the listening socket and send every outgoing packet to **all** of them; vehicle selection happens on the aircraft via the MAVLink `target_system` field:

- MP: every new sender is appended to `EndPointList` ([ExtLibs/Comms/CommsUdpSerial.cs:201-202]); `Write` sends to everyone in the list ([CommsUdpSerial.cs:264-276]).
- QGC: `_sessionTargets` accumulates the same way ([UDPLink.cc:472]); `writeData` sends to configured hosts + all session targets ([UDPLink.cc:390]).

So the "MP sends the command to the wrong drone" hypothesis is **not confirmed** — commands physically reach all 5 drones in both programs. 100% of the freeze difference is in the wait/threading architecture. (Shared side effect: every outgoing GCS packet is duplicated into 5 datagrams — uplink load; same in both.)

MP's ACK matching is also correct (sysid+compid+command check, [MAVLinkInterface.cs:2803-2813]) — the problem is not the matching, it is the blocking wait model when no match arrives.

---

## 7. Side by Side: "Pressed Arm, the ACK Got Lost" (5 drones, one port)

| t | Mission Planner | QGroundControl |
|---|---|---|
| 0 s | ARM sent; UI thread blocked in `GetResult()`; `giveComport=true` → all 5 vehicles' telemetry processing stops | ARM sent; call returns; UI fluid, 5 vehicles live |
| 0–10 s | Window fully frozen; clicks queue up | 1.2–1.7 s: **"Vehicle did not respond to command: Arm"** non-modal dialog; operator retries |
| 10/20/30 s | Silent resends (retry 3→0) | — |
| 40 s | `TimeoutException` → UI thaws → modal "No response" → queued clicks fire one by one (mostly starting new blocks) | — |

---

## 8. MP Handler Inventory — Which Button Blocks How Long

Handlers verified to make synchronous comm calls on the UI thread ([GCSViews/FlightData.cs]):

| Handler | Line | Call | Worst block |
|---|---|---|---|
| `BUT_ARM_Click` | 1057, 1068 | `doARM` (+force) | 40 s (+40) |
| `takeOffToolStripMenuItem_Click` | 5305-5310 | `setMode`+`doCommand(TAKEOFF)` | ~8 s |
| `BUT_setwp_Click` | 1663 | `setWPCurrent` | ~12 s |
| `BUTrestartmission_Click` | 1889 | `setWPCurrent` | ~12 s |
| `BUTactiondo_Click` (Actions/Do Action) | 1798-1865 | `doCommand`/`doReboot`/`doEngineControl`/… | 2–8 s |
| `BUT_mountmode_Click` | 1398-1404 | `setParam`/`doCommand` | ~3 s |
| `BUT_resumemis_Click` | 1557-1612 | `setMode`+`doARM`+`doCommand` busy-wait + `DoEvents` | 30+ s |

The correct pattern already exists in MP's own code but was only used in 2 handlers: `modifyandSetSpeed_Click:4428` and `setHomeHereToolStripMenuItem_Click:4854` (`await doCommandAsync(...).ConfigureAwait(true)`) — the patch extends this pattern to arm/takeoff/setwp.

Cross-reference: ArduPilot/MissionPlanner issue [#2784](https://github.com/ArduPilot/MissionPlanner/issues/2784) "extremely slow param download with multiple vehicles" — the parameter-download face of the same shared-link/`giveComport` monopolisation.

---

## 9. Practical Mitigations Without Code Changes (honest assessment)

1. **One UDP port per drone + separate connections in MP.** Since `giveComport` is per link, a command waiting on one drone no longer stops the *processing* of the other four — a real gain. Limit: the UI-thread block itself remains; the window still freezes. **Softens the problem, does not fix it.**
2. **Lower telemetry rates** (`SR0_*`/`SR1_*` per vehicle): less traffic → fewer lost ACKs → the 40 s path triggers less often.
3. **Send multi-vehicle commands from QGC** — architecturally the correct behavior is in QGC.
4. **Careful with the "Force Arm" dialog:** if arm is rejected, confirming force can start a second 40 s block on a lossy link; Cancel + retry is usually faster.

**The permanent fix is in the code** → see [PATCH-NOTES.md](PATCH-NOTES.md).

---

## 10. Reproducing with SITL

1. Start 5 ArduCopter SITL instances, all `--out` to the same GCS port (distinct `SYSID_THISMAV` 1–5).
2. Connect stock MP (UDP listen): 5 sysids appear on one connection.
3. Simulate ACK loss: suspend one SITL process, or firewall-drop its outbound packets.
4. Press Arm on that vehicle → stock MP freezes ~40 s (validates this analysis). Same scenario in QGC errors out in ~1.5 s.
5. Repeat with the patched MP → UI must not freeze; the result message arrives when the command completes.
