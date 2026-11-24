# Project journal

A dated record of what was actually done, including the wrong turns.
Kept because the wrong turns are the expensive part.

---

## Phase 0 — Survey (Nov 8)

| Date | Action | Outcome |
|---|---|---|
| 11-08 | Read-only inventory of both libraries | 137,306 assets in Immich, 46,189 in Photos |
| 11-08 | Fingerprint every candidate file | Real gap: **55,924 files / 252.7 GB**, not the naive 91,000 |
| 11-08 | Snapshot `Photos.sqlite` before touching anything | 1.5 GB baseline |

**Lesson:** the naive gap (asset count difference) was 63% too large. Content fingerprints are the only honest measure.

---

## Phase 1 — First contact (Nov 11)

| Time | Action | Outcome |
|---|---|---|
| 08:20 | `-10004` on every library-level AppleScript call | Root cause: the calling process had no Photos permission |
| 08:29 | Pilot: 190 files from 2014, run from a terminal | 188 imported, 2 skipped, 0 errors |
| 08:36 | Verified album, dates, iCloud upload | All correct — **except filenames became UUIDs** |
| 08:55 | Full Disk Access granted | The process could now import directly; the app wrapper was unnecessary |
| 08:56 | Bulk import started | 300 per batch |

**Lesson:** the first pilot looked perfect and was still wrong — it passed every check except the one nobody wrote down (filenames).

---

## Phase 2 — Things break (Nov 12–13)

| Date | Event | Cost |
|---|---|---|
| 11-12 | Photos wedged; WAL had grown to **60 GB / 14.5M frames** | ~1 day |
| 11-12 | A watchdog job ran **330 times and never worked** (no TCC identity under launchd) | ~2 days idle |
| 11-12 | `--stop-on-error 5` aborted batches **without writing the report** → infinite retry | ~half a day |
| 11-13 | Discovered the real culprit for 240 s stalls: **a modal dialog** | ~20 hours |

**Lesson:** the most expensive failures are the ones that look like something else.

---

## Phase 3 — Finish (Nov 15)

| Time | Event | Result |
|---|---|---|
| 14:47 | Queue reached zero | 55,859 files in the library |
| 16:42 | Repair pass on 631 failures (re-encode + re-import) | **569 recovered**, 62 refused |
| 17:05 | Final | Library 46,189 → **101,963**; 95 unimportable (0.17%) |

---

## Phase 4 — Make it survive (Nov 16)

| Time | Event | Result |
|---|---|---|
| 23:41 | **Reboot test** | App started at login, spawned the guard by itself |
| 23:47 | Found: Photos never launches itself after a reboot | Watchdog now launches it — **in the background** (`open -g`) |
| 23:52 | Menu said "quit (sync continues in background)" — it did not | Relabelled, plus a confirmation dialog |
| 23:53 | `kill(pid, 0)` alone is not a liveness check — **PIDs get reused** | Now verifies the command line too |
| 23:57 | Clean restart | One app, one guard, permissions intact |
| 23:57 | Repaired the 62 remaining bad files | 62/62, none given up |

---

## Phase 5 — Living with it (Nov 17–24)

The migration was "done". Then real use found things the tests had not.

| Date | Finding | Outcome |
|---|---|---|
| 11-17 | Lock detection false-positived on a brief display sleep; a healthy system sat in `state: locked` while Photos answered in 0.2 s | Probe Photos first; use the lock flag only to *explain* a failure |
| 11-24 | Every year album displayed in a strange order | Script-created albums default to **manual sort** (`ZCUSTOMSORTKEY=1`), and there is no scripting API to change it. Fixed by rewriting the album item order in the database — 5,960 inversions → 0, member counts unchanged |
| 11-24 | User reported "many photos missing" from the year albums | Not missing. The albums contain only what this project imported (58,949); the library also holds 43,602 photos that were already in iCloud and never entered Immich |
| 11-24 | Timeline dates felt scrambled | 85.5% exact; 12% off by exactly 8h and **Immich was the wrong one**; 2.5% genuinely wrong — see [metadata-fidelity.md](metadata-fidelity.md) |

**Two lessons worth keeping:**

1. **A "finished" migration is not a verified one.** Every problem above appeared only under real use.
2. **The master library is not axiomatically correct.** In the 8-hour group, Immich — the source of
   truth — had the wrong timestamps, and the import was right.

---

## Final numbers

| | |
|---|---|
| Target | 55,924 unique files / 252.7 GB |
| In the library | 55,859 (**99.83%**) |
| Recovered by re-encoding | 569 |
| Unimportable | 95 (0.17%), originals untouched in Immich |
| Photos library | 46,189 → 102,362 |
| Wall clock | 9 days |
| **Actual work** | **~20 hours** |

The remaining 84 hours were stalls, wrong diagnoses, and scheduler bugs — mostly mine.
