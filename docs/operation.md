# The operational layer

A batch script that works is not the same as a system that keeps working.
After the migration finished, the harder question was: **how do you keep this alive for months without a human watching?**

---

## Topology

```
ImmichSync.app        (Swift, LSUIElement — menu bar only, no dock icon)
  └── guard_loop.sh    (direct child process, writes guard.pid)
        └── watchdog.sh            every 5 minutes
              ├── screen-lock check
              ├── Photos liveness probe   (30 s timeout)
              ├── Immich ping
              ├── disk free / WAL size
              ├── write status.json
              ├── stall detection → restart
              ├── repair queue when idle
              └── discover new assets
```

## The `status.json` contract

The app never talks to Python. The two halves communicate through one file:

```json
{
  "ts": "2026-11-15 18:44:29",
  "state": "idle",          // idle | syncing | stalled | error | locked
  "pending": 0,
  "failed": 0,
  "imported": 47831,
  "photos_ok": true,
  "immich_ok": true,
  "disk_free_gb": 2468,
  "wal_gb": 7.5,
  "busy": 0,
  "batch_age_min": 4
}
```

Keeping this contract dumb is what lets the UI be rewritten without touching the engine.

---

## Four traps that only appear in the operational layer

### 1. A detached child loses the parent's TCC identity

The guard must be a **direct child** of the app:

```swift
let p = Process()
p.executableURL = URL(fileURLWithPath: "/bin/bash")
p.arguments = [projectDir + "/guard_loop.sh"]
try p.run()          // keep the reference alive
```

Using `nohup ... &` instead detaches it. The child is reparented to `launchd`, and — verified empirically — **loses access to the external volume**:

```
kill(pid, 0) == 0            // process is alive
but: guard.pid never appeared, guard.log stopped updating
```

Same binary, same user, different parent → different authorization.

### 2. App Nap freezes everything, including threads

A background app with no windows gets napped. Timers do not fire. `Timer`, `DispatchSourceTimer`, and even a plain background thread with `Thread.sleep` all froze.

Fix:

```swift
ProcessInfo.processInfo.beginActivity(
    options: [.userInitiated, .idleSystemSleepDisabled, .suddenTerminationDisabled],
    reason: "sync supervisor")
```

### 3. `pgrep -f` matches its own command line

```bash
/bin/bash -c "pgrep -f guard_loop.sh"   # always returns 0 — it matches itself
```

The supervisor concluded "already running" forever and never started anything. Avoid name-based supervision entirely: **write a PID file and use `kill(pid, 0)`.**

### 4. PID reuse defeats "is it alive?" checks

A PID file plus `kill(pid, 0)` is not enough after a reboot: PIDs are reused,
so the supervisor finds "a live process" that is something else entirely and
**never starts the worker**.

Verify the command line too:

```swift
guard kill(pid, 0) == 0 else { return false }
return runCapture("ps -p \(pid) -o command=").contains("guard_loop.sh")
```

### 5. Rebuilding the app silently revokes its permissions

TCC binds grants to the code signature. Every recompile changes the ad-hoc signature, so the user's earlier approval no longer applies — with no error and no prompt.

**Rule: compile → sign → authorize → then never rebuild.** Fix bugs by editing the shell/Python it calls, not the app.

---

### 5. After a reboot, Photos does not launch itself

The login item brings the app back, the app spawns the guard — and then the whole chain reports
`state: error` because **Photos is not running**. A probe that only *checks* whether Photos is alive
will never bring it back.

The watchdog must launch it:

```bash
if pgrep -x Photos >/dev/null; then
    ... probe ...
else
    open -a Photos
    sleep 20
    ... probe again ...
fi
```

Found during the post-reboot verification, which is exactly what that verification was for.

---

## Screen lock

While the screen is locked, Photos may ignore AppleScript entirely. A naive watchdog interprets this as "Photos is hung" and restarts it — repeatedly, forever.

**But the lock flag itself is unreliable.** `CGSSessionScreenIsLocked` can be set by a brief display sleep, so treating it as a prerequisite pauses a perfectly healthy system — which is exactly what happened here: the app sat in `state: locked` while Photos was answering in 0.2 s.

The correct order is the other way round: **probe Photos first, use the lock flag only to explain a failure.**

```bash
if probe_photos; then
    PHOTOS_OK=true
elif [ "$LOCKED" = "yes" ]; then
    pause            # genuinely locked and genuinely unresponsive
else
    restart_photos   # unresponsive for another reason
fi
```

For reference, the check itself:

```python
import Quartz
d = Quartz.CGSessionCopyCurrentDictionary()
locked = bool(d and d.get('CGSSessionScreenIsLocked'))
```

If locked: write `state: locked`, do nothing, exit. Resume automatically on unlock.

---

## Permissions (one-time, on the app)

| Grant | Why |
|---|---|
| Full Disk Access (or Removable Volumes) | read/write the external library |
| Photos | send Apple events to Photos |
| Login item (SMAppService) | start at login |

`SMAppService.mainApp.register()` registers the login item programmatically; the user still confirms once in System Settings.

---

## What this layer does and does not buy you

**Does:** survive reboots, restart a dead worker, pause on lock, notify on failure, expose state.
**Does not:** adapt to a macOS major version that changes AppleScript, or to an Immich release that changes its schema.

Long-term reliability here means *"when it breaks, it says so, and the fix is small"* — not *"it never breaks"*.
