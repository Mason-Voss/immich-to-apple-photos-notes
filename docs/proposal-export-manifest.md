# Proposal: a stable export manifest for Immich

> Posted as a Feature Request discussion: https://github.com/immich-app/immich/discussions/33306

## Problem

Every migration/export tool built against Immich has to answer the same three questions:

1. What assets exist, with their **original filenames**, capture times and album membership?
2. How do I fetch the **original** bytes for each one?
3. How do I know I got everything, without walking the whole library twice?

Today there are only two ways, and both are fragile:

- **Paginate the REST API.** Works, but a 100k-asset library means ~1,000+ requests (throttled, restartable, and the exact endpoint shape has moved between major versions).
- **Read PostgreSQL directly.** Fast and exact, but it couples the tool to internal schema, which *did* change between v3.2 and v3.3. It also breaks the moment someone runs Immich on a managed host.

The result is that every tool re-solves the same problem, badly, and breaks on every schema change.

## Proposal

A single read-only endpoint that returns a **manifest** of exportable assets:

```
GET /api/export/manifest
    ?albumId=<uuid>          # optional
    &from=<iso8601>          # optional
    &to=<iso8601>            # optional
    &updatedAfter=<iso8601>  # optional, for incremental runs
```

Streamed as NDJSON, one asset per line:

```json
{
  "assetId": "…",
  "originalFileName": "IMG_1234.HEIC",
  "originalPath": "/uploads/…",
  "capturedAt": "2024-05-01T10:00:00Z",
  "type": "IMAGE",
  "byteSize": 4321987,
  "checksum": "…",
  "albumIds": ["…"],
  "isLivePhotoVideo": false,
  "downloadUrl": "/api/assets/…/original"
}
```

## Why this is worth doing

- **It does not add storage or processing.** It is a projection of data Immich already has.
- **It decouples every migration tool from the schema.** One stable contract instead of N implementations of `PRAGMA table_info`.
- **It serves both directions.** `immich-go`-style importers can use the same shape for verification and dry-run diffs.
- **Entering and leaving stays symmetric.** Immich is excellent at ingesting. Leaving is the part that still requires bespoke tooling — and that matters for the "own your data" promise.

## Scope

Deliberately minimal:

- Read-only. No new writes, no new tables.
- No format conversion, no metadata rewriting — just an honest inventory plus a pointer to the original bytes.
- Auth: the existing API key scopes (`asset.read`, `album.read`) are sufficient.

## Alternatives considered

| Option | Why not |
|---|---|
| Document the DB schema | Couples tools to internals; changes every release |
| Improve existing pagination | Still N requests; still schema-shaped |
| A separate CLI shipped by Immich | Larger surface to maintain than one endpoint |

## What this would have changed for us

We built a one-way mirror from Immich into Apple Photos (55,924 assets). The single largest engineering cost was reconstructing exactly this inventory from the database — and re-validating it after the v3.2 → v3.3 schema change.

With this endpoint, the same job would have been: stream the manifest, fetch originals, done.

---

*Written after migrating 55,924 assets out of a self-hosted Immich instance. Field notes: https://github.com/Mason-Voss/immich-to-apple-photos-notes*