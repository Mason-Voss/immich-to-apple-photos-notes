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
---

## When there is no EXIF at all: rescuing dates from filenames

About 7,500 files in this library — WeChat exports, screenshots, OPPO camera shots, re-saved images —
carry **no date metadata whatsoever**. Both Immich and Photos fell back to the file's save time,
which for a bulk copy in Nov 2025 dumped them all into November 2025. That is what "the timeline is a
mess" looks like in practice.

The filename is the only remaining evidence:

```
IMG20220813165207.jpg          -> 2022-08-13 16:52:07
wx_camera_1669803238932.jpg    -> epoch ms -> 2022-11-30
mmexport1661100000000.jpg      -> epoch ms
Screenshot_2022-11-19-18-22-26_...jpg
```

### The algorithm, and the two ways it goes wrong

1. Find **all** candidate date patterns, then take the one appearing **earliest** in the string.
   Not "try epoch first" — a trailing numeric ID beats an epoch in priority otherwise
   (`IMG_20181223_182458_1581298172029.jpg` would land in 2020 instead of 2018).
2. **Validate the year.** Pure-numeric names like `2102052040521054.mp4` will happily match
   `20xx` and produce year 2052. Reject anything outside 1990..now+1.

The first run of this fix hit both bugs and produced years 2029/2034/2052. It was rolled back from a
full database backup and re-run. **Always dry-run: print the target-year histogram before writing.**

### EXIF still wins

631 of the candidates turned out to have real EXIF that agreed with the stored date — the *filename*
was misleading. Trust order: EXIF → filename → file time.

### What remains unfixable

1,033 files have neither EXIF nor a date in the name. The only other clue would be the file's
creation time, which for copied files *is* the wrong value being corrected. For the 4 AVI files,
`ffprobe` was tried for an embedded `creation_time` — none had one.

**Result of the pass: 8,147 wrong dates → 863 (89.4% fixed).**
### Second pass: boundaries matter more than coverage

The first pass used an epoch pattern of `1[5-9]\d{11}`, which silently only covers timestamps after
2017. Everything from 2013–2017 (1.3–1.49 trillion ms) was skipped. Widening it to any 13-digit
`1\d{12}` recovered those — but only with **digit boundaries**:

```python
epoch = re.compile(r'(?<!\d)(1\d{12})(?!\d)')
```

Without the boundaries, `2101102315334972.mp4` (a 16-digit ID) gets sliced into a plausible 2002
date, and `52058797899713231641678292816480.jpg` (32 digits) yields 2022. Twelve such corrections
from the first pass had to be reverted from the backup.

**Rule: a date pattern without boundaries is a date generator, not a date parser.**

### Six ways to date a file that has no date — five of them failed

For the 1,179 photos in Oct–Dec 2025 that carry nothing usable, every route was tested:

| Route | Outcome |
|---|---|
| EXIF | absent — stripped during re-save |
| filename | no date token |
| file created/modified time | *is* the wrong value being corrected |
| source folder name | UUID substrings produced fake dates (2059, 2084) |
| same byte size as another asset | 72 matches, all false (GIF paired with JPG) |
| content hash vs whole library | all 1,147 hashed; **4** true duplicates |

They are single copies. The information was destroyed before it reached this machine, and the only
remaining source is the owner's memory.
---

## Known limitation (accepted 2026-11-25)

**1,179 photos dated Oct–Dec 2025 have no recoverable capture date.** They are WeChat exports,
screenshots and re-saved images from 2013–2018 that were bulk-copied into both Immich and Photos in
the autumn of 2025; every date-bearing field was stripped before they arrived.

Six recovery routes were tested (EXIF, filename, file timestamps, source folder names, byte-size
matching, full content hashing) — see above. They are single copies.

**Owner's decision: accept rather than approximate.** Assigning a guessed year with a fabricated day
would replace a known-unknown with a confident error. These files keep their copy date and are
understood to be undated.

Impact: 1,179 of 103,412 assets (1.1%). The remaining 92.6% of previously-wrong dates were corrected.
