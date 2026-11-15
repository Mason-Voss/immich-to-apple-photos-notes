# The Traps

Observed behaviour of the Photos AppleScript channel. Each entry gives a **reproducible symptom** so the next person can pattern-match quickly.

---

## 1. A modal dialog blocks everything for 240 seconds

### Symptom
- `osascript -e 'tell application "Photos" to count of albums'` → times out (`-1712 AppleEvent timed out`)
- Meanwhile `tell application "Photos" to get name` → returns in 0.1 s
- A sheet reading "Cannot import N items" is sitting on screen

### Mechanism
When an import fails, Photos raises a modal sheet and the main thread waits for a human. The app can still answer *application-level* properties, but **any event that touches the library model blocks** until AppleScript's default timeout (~240 s).

### How to tell it apart
```bash
# Answers = the app is alive. Says nothing about health.
osascript -e 'tell application "Photos" to get name'

# The real liveness probe: a library-level query with a hard timeout
osascript -e 'tell application "Photos" to count of albums'
```
A 240 s timeout on the second command with a fast answer on the first is the signature.

### Three layers of defense
1. **Pre-flight validation** (cheapest, catches ~90%)
   ```bash
   sips -g pixelWidth -g pixelHeight file.jpg        # "<nil>" = unreadable
   ffprobe -v error -select_streams v:0 \
           -show_entries stream=codec_name -of csv=p=0 file.mp4
   ```
2. **Canonical transcoding** — `sips -s format jpeg` or `ffmpeg -c:v libx264 -c:a aac`
3. **Accessibility fallback** — locate the sheet's "OK" button via AXUIElement and click it

### Measured cost
240 s per event. 230 events ≈ 10–15 hours. 101 forced Photos restarts.

---

## 2. WAL growth to 60 GB

### Symptom
`Photos.sqlite-wal` grows from megabytes to 60 GB. Photos gets progressively slower, then appears hung.

### Inspection
```bash
ls -lh "Photos Library.photoslibrary/database/Photos.sqlite-wal"

sqlite3 "file:...Photos.sqlite?mode=ro" "PRAGMA wal_checkpoint(PASSIVE);"
# 1|14588277|42  → busy; 14.5M frames pending, 42 checkpointed
```

### Mechanism
SQLite can only reclaim WAL frames once every reader releases its snapshot. `photolibraryd` holds a long-lived read connection, so the checkpoint never advances.

### What works, what doesn't
| Action | Result |
|---|---|
| `PRAGMA wal_checkpoint(PASSIVE)` | No effect (blocked by reader) |
| Kill `photoanalysisd` / `mediaanalysisd` | No effect (not the holders) |
| Kill `photolibraryd` | No effect (launchd restarts it, re-acquires) |
| **Restart Photos** | **Works**: 60 GB → 13 GB → 644 KB |

### Prevention
Restart Photos every 5,000 imports or whenever the WAL exceeds 10 GB.

---

## 3. `--stop-on-error` silently skips the report

### Symptom
The import exits non-zero **and the `--report` CSV does not exist at all**.

### Mechanism
osxphotos aborts once the `--stop-on-error` threshold is hit, and the report is never written. A ledger built on the report then concludes "the whole batch failed" and requeues it — forever.

### Fix
- Raise the threshold (we used 50).
- More importantly: **a missing report must never be interpreted as a business conclusion.** Requeue and retry; do not mark anything.

---

## 4. TCC has three layers, and launchd has no identity

### Symptom
```
sqlite3 /Volumes/.../state.db
Error: unable to open database: authorization denied
```
The same command works in a terminal, fails inside a LaunchAgent.

### Mechanism
TCC binds permissions to the **responsible application's identity**, not to a file or a script. launchd-spawned processes inherit no user identity, so external volumes and the photo library are both denied.

| Launched by | External volume | Photos |
|---|---|---|
| Terminal / IDE | ✅ | ✅ |
| launchd | ❌ | ❌ |
| Signed app bundle | ✅ | ✅ |

### Consequence
**Any macOS automation that relies on launchd must be verified with `launchctl kickstart` and its log inspected.** Our job ran 330 times and silently did nothing.

---

## 5. Fingerprints exist for images, not reliably for videos

### Symptom
Videos are repeatedly treated as new and imported again.

### Mechanism
`--skip-dups` compares against fingerprints stored in the Photos database. **Photos does not always store one for videos.**

### Mitigation
- Images: fingerprint
- Videos: `(filename, byte size, duration)` triple as a fallback
- Or: compute SHA-256 before import and keep it in your own state database

### Counter-lesson
None of filename, size, or date is reliable on its own:
- Immich stores files under UUID names
- iOS backups export timestamp-based names
- Different photos can share a name and a size

---

## Appendix: other observations

| Behaviour | Note |
|---|---|
| `photosLibraryWaitForPhotos` pre-check | Times out on libraries above ~46k assets; skip it |
| Screen locked | Photos ignores AppleScript entirely; imports hang |
| Live Photos | Immich splits HEIC/MOV into two assets; they import as photo + video |
| Albums | `--album "Immich/{filepath.parent.name}" --split-folder "/"` produces nested folders |
| Filenames | Must stage via hard links using the original name, or the library stores UUIDs |
