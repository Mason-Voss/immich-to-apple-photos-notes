# Metadata fidelity: what actually survived

After a migration people ask the wrong question — *"did the files arrive?"* — and skip the one that
matters a year later: *"is the timeline honest?"*

This is what was measured after 55,859 files landed, and what was found. None of it was obvious.

---

## Time: 85.5% exact, 12% explained, 2.5% genuinely wrong

Comparing every asset that exists in both systems (56,110 pairs), by comparing Immich's
`dateTimeOriginal` against `ZASSET.ZDATECREATED`, both in UTC:

| Result | Count | Share | Verdict |
|---|---|---|---|
| Same instant (< 2 s) | 47,988 | 85.52% | correct |
| Exactly 8 hours apart | 6,709 | 11.96% | **Immich is wrong, Photos is right** |
| Genuinely different | 1,413 | 2.52% | mixed causes |

### The 8-hour group

Direction was always the same: the Photos library stored an instant 8 hours *earlier* than Immich.

```
IMG_20260713_221450.jpg
  Immich:    2026-07-13 22:14 UTC   -> displays 06:14 next day   ✗
  Photos DB: 2026-07-13 14:14 UTC   -> displays 22:14            ✓ matches the filename
```

Immich had parsed a local wall-clock time and stored it as if it were UTC. Photos stored the true
instant and applied `ZTIMEZONEOFFSET = 28800`.

**Worth internalising:** the "master" library is not automatically the more correct one. Verify
against an independent source (the filename, the original EXIF) before deciding which side to fix.

### The 2.5%

Causes found, in order of frequency:

1. No usable EXIF date in the source, so the file's **modification time** was used — that is the
   copy/download date, not the capture date.
2. Immich had used the **upload time** where the file carried a date (opposite failure).

Any future pass that corrects these must decide per-file, not per-rule.

---

## Location: nothing was lost

A naive query said "45,644 assets have no location". That number is meaningless, because Photos
uses **`(-180, -180)`** as its "no location" sentinel. Filtering properly:

| | Has coordinates | No coordinates |
|---|---|---|
| Imported by this project | 29,304 | 26,559 |
| Whole library | 57,764 | 45,644 |

Then the question that matters: of the ones without coordinates, how many *had* coordinates in the
source file?

Sample of 15 taken specifically from the "no location in Photos" set, checked with `exiftool`:

```
source file also has no GPS : 15
source file has GPS, lost   : 0
```

The photos without location are screenshots, WeChat images, downloads and scans. **They never had
GPS.** Nothing was dropped.

---

## The check that finds real problems: filename vs stored date

Scanning every filename that contains a date (80,946 of them) and comparing with the stored capture
date surfaced 6,219 disagreements — which looked alarming until split by provenance:

| Set | Filename-dated | Disagreeing | Rate |
|---|---|---|---|
| Imported by this project | 45,897 | 6,033 | 13.14% |
| Pre-existing library | 35,049 | 186 | 0.53% |

A 25× difference looks like a smoking gun, and mostly isn't: for the majority the filename itself is
misleading (files renamed during transfer), and the stored date faithfully follows the EXIF. The
genuine failures are the 2.5% above.

**The lesson is about method:** a mismatch rate tells you where to look, never what is wrong. The
next step must always be *open the source file and read its EXIF*.

---

## What to check after any future import

```bash
# 1. same instant?
#    compare Immich dateTimeOriginal (UTC) with ZDATECREATED (UTC), never with 'localtime'

# 2. exactly-8h bucket
#    a large bucket here means a timezone convention bug on one side, not random error

# 3. real GPS check
#    ZLATITUDE BETWEEN -90 AND 90 AND ZLATITUDE <> -180   (NOT just "not null")

# 4. the sentinel
#    -180,-180 means "no location" in Photos. Treating it as data inflates every GPS metric.
```
