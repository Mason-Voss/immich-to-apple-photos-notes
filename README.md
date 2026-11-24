# Mirroring Immich into Apple Photos

**Field notes from moving 55,924 assets into a closed, GUI-only photo library — with a 99.83% success rate and a painful lesson about where the time actually goes.**

[中文版](README.zh-CN.md)

---

## TL;DR

| | |
|---|---|
| Goal | Immich as the system of record; Apple Photos + iCloud as the viewing layer. One-way, append-only. |
| Scope | 55,924 unique files / 252.7 GB |
| Result | 55,859 in the library (**99.83%**), 95 rejected (0.17%) |
| Library | 46,189 → 101,963 assets |
| Wall clock | 104.5 hours |
| **Actual work** | **~20 hours (19%)** |
| Conclusion | **The bottleneck was never throughput. It was stability engineering.** |

---

## 1. The real shape of the problem

It looks like copying files. It is not.

It is **writing across a trust boundary into a closed system**:

- The source (Immich) is open: SQLite and a REST API. Read anything.
- The destination (Photos) exposes exactly one channel: AppleScript. **No error codes, no transactions, no rollback — its failure mode is to freeze the UI.**
- In between sit two OS policies: TCC (privacy consent) and the app sandbox.

Moving bytes was 20% of the work. The other 80% was discovering the conditions under which the destination silently stops working.

---

## 2. Architecture

    Immich (system of record)
        │  ① read-only: asset table / API
        ▼
    state.db  (one row per asset: pending → imported / skipped / failed / unrepairable)
        │  ② hard-link staging (same volume, zero extra space, keeps original filenames)
        ▼
    staging/<year>/<original_filename>
        │  ③ osxphotos import (batched, with --report)
        ▼
    Photos Library (external APFS volume)
        │  ④ iCloud upload
        ▼
    iPhone / iPad / Mac

### Three non-negotiable design rules

1. **The report file is the only ledger.** Never trust the exit code.
   Exit code 0 does not mean "imported". Only `--report` does.
2. **Append-only.** Nothing is ever deleted. The worst case is "not imported", never "lost".
3. **Idempotent.** Any batch can be re-run forever (content-fingerprint dedupe), which downgrades "crash" from disaster to inconvenience.

---

## 3. The five traps

Full write-up with reproducible symptoms: **[docs/traps.md](docs/traps.md)**

| # | Trap | Cost |
|---|---|---|
| 1 | A modal dialog in Photos blocks **every** AppleScript event for 240 s | ~20 h |
| 2 | The SQLite WAL grew to **60 GB**, making Photos appear hung | ~1 day |
| 3 | `--stop-on-error` aborts **and silently skips writing the report** | ~half a day |
| 4 | launchd-spawned processes have **no TCC identity** — a watchdog ran 330 times and never worked | ~2 days |
| 5 | Photos stores fingerprints for images but **not reliably for videos** | design-level |

The single most misleading one is #1: it presents as "Photos is busy", then as "the machine is overloaded", then as "a permissions problem". It took three wrong diagnoses to find a dialog box.

---

## 4. Patterns that actually held up

| Pattern | Why it mattered |
|---|---|
| Report-based ledger | Across 230 failure events, it never once recorded "not imported" as "imported" |
| Hard-link staging | Same inode, zero bytes, yet preserves original filenames (otherwise every asset is named after a UUID) |
| Binary-search retry | One bad file no longer takes down 59 good ones; 34% of batches failed, but losses stayed local |
| Re-encode repair | Recovered 569 of 631 failures (90%) |
| Heartbeat liveness | Judge by log freshness, not by process existence |

---

## 5. Where 80% of the time went

At a measured healthy rate of 0.8 assets/s, 55,859 assets need ~20 hours. It took 104.5.

| Loss | Estimate |
|---|---|
| Scheduler self-inflicted (4 h cap with no continuation, supervisor killed with the session, launchd watchdog that never ran) | 35–40 h |
| Waiting on modal dialogs (230 events × 2–4 min) | 10–15 h |
| Photos restarts (101 × 40 s) | ~1 h |
| Binary-search retries and backoff | 3–5 h |

**Only 15–20 hours were genuinely unavoidable. The rest was engineering error.**

---

## 6. Runbook

Step-by-step, with verification points and monitoring thresholds: **[docs/runbook.md](docs/runbook.md)** · **[docs/journal.md](docs/journal.md)** · **[docs/metadata-fidelity.md](docs/metadata-fidelity.md)**

## 6b. The operational layer

Making it survive reboots, locks, and crashes is a separate problem with its own traps —
App Nap freezing timers, detached children losing TCC identity, `pgrep -f` matching itself,
and rebuilds silently revoking permissions.

**[docs/operation.md](docs/operation.md)** — the runtime topology, the `status.json` contract,
and the four traps that only appear once you leave the terminal.

---
## 7. Disclosure

This is a **field report, not a packaged tool**. Be aware of two things before borrowing the approach:

**Direct database reads.** The method described here queries Immich's PostgreSQL directly (via `docker exec psql`) instead of the REST API. That was a deliberate trade-off for a one-off local migration: it avoided tens of thousands of HTTP round-trips and gave exact `originalPath` values.

For anything long-running or multi-user, **use the REST API instead.** Direct database access couples you to Immich's internal schema, which changes between versions (it did between v3.2 and v3.3).

**Not production-grade.** The scripts were written against one specific library on one machine. There is no test suite, no sandbox, and the paths are hard-coded. The reusable part is the *method*, not the code.

---

## 8. Limits (read before you start)

- ❌ **It cannot auto-start.** TCC is a contract, not a bug. One-time manual grants are required.
- ❌ **No Live Photo pairing.** Immich stores HEIC and MOV as two separate assets.
- ⚠️ It depends on AppleScript. Apple changes it, this rots.
- ⚠️ Modal dialogs can be driven to near-zero, never to zero.
- ⚠️ Photos ignores AppleScript while the screen is locked.

---

## 9. If this became a product

Three layers. The third is now partially built — see [docs/operation.md](docs/operation.md):

1. **Defense**: pre-flight validation → canonical transcoding → accessibility-based dialog dismissal
2. **Identity**: an app bundle registered via `SMAppService` (grant once, survives reboots)
3. **Experience**: menu-bar status, notifications, pause/resume — *implemented*

The moat is not the code. It is the traps above.

---

## License

MIT