# Runbook

Execute in order. Every phase has a verification point.

---

## Phase 0 — Read-only survey

1. Locate the Photos library and check iCloud settings
   - Photos → Settings → General: library path
   - Photos → Settings → iCloud: "Download Originals" or "Optimize Storage"
2. Confirm Immich is reachable; note version and asset count
3. Confirm the target volume is APFS with free space > 2× the payload
4. **Compute the real gap**: match both sides by (filename, byte size, fingerprint)

> Do not skip this. Our naive "gap" of 91,000 assets was actually 55,924.

**Verification**: a table reading "N assets / X GB to transfer".

---

## Phase 1 — Sandbox

- Clone a 500-asset library
- Validate every mechanism there first: pre-flight, binary search, re-encode, dialog handling

**Verification**: 500 assets transfer with zero manual intervention.

---

## Phase 2 — Pilot

- Pick one year or album (100–200 assets)
- Import → verify count, dates, album membership, iCloud upload

**Verification**: library delta == imported count; no date drift.

---

## Phase 3 — Bulk

```
per batch of 60:
  stage hard links → import → read report → update ledger → clean staging
  sleep 15–30 s between batches
  restart Photos every 5,000 assets (WAL reclamation)
```

**Failure handling**: on a whole-batch timeout, bisect (60 → 30+30 → … → single file).

---

## Phase 4 — Repair

For `failed` assets:
1. Re-encode: `sips -s format jpeg` or `ffmpeg`
2. Restore capture time with `exiftool` (GPS usually survives sips)
3. Re-import

**Measured recovery rate: 90%** when Photos is otherwise idle.

---

## Phase 5 — Incremental

- Every 5 minutes: discover assets from the last 7 days
- Delay Apple devices by 24 h (avoids racing iCloud)
- If the queue is non-empty and nothing is running, start a batch

---

## Four metrics to watch

| Metric | Threshold | Action |
|---|---|---|
| Log heartbeat | > 30 min stale | Force restart |
| WAL size | > 10 GB | Restart Photos |
| Free disk | < 100 GB | Stop |
| Consecutive failures | > 5 batches | Stop and alert |

---

## Permissions (one-time)

1. Full Disk Access — read/write external volume
2. Photos — access the library
3. Automation → Photos — send Apple events
4. (Optional) Accessibility — dismiss modal sheets

---

## Rollback

- The state database stores each asset's Photos UUID; delete by batch
- Albums are organised by year and can be removed wholesale
- Snapshot `Photos.sqlite` before starting
