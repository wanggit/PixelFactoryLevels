# PixelFactoryLevels

Hot-update level bundles for **Pixel Depot** (`PixelFactory` repo). Served as
static files over GitHub Pages — this repo is a CDN, not a source of truth.

```
https://wanggit.github.io/PixelFactoryLevels/manifest.json
https://wanggit.github.io/PixelFactoryLevels/bundle-v<N>.json
```

## ⚠️ Do not edit the JSON files by hand

`manifest.json` carries a **sha256 of the bundle file's bytes**. Change one byte
of a bundle — or reformat it, or let an editor add a trailing newline — and every
client will compute a different checksum, reject the bundle, and **silently fall
back to whatever it already had**. Nothing crashes, nothing is logged server-side,
the files are all there and return 200; players just never receive the levels.

Bundles are also written in **compact encoding** (no indentation) on purpose:
the checksum is computed over the exact bytes that get published, so any
reformatting invalidates it.

To publish, use the generator in the game repo:

```bash
dart run tool/make_remote_level.dart --art=<artwork.png> --id=L114
dart run tool/build_level_bundle.dart          # version auto-increments
dart run tool/build_level_bundle.dart --check   # verify before deploying
dart run tool/verify_hot_update.dart            # real HTTP end-to-end
# then copy remote_dist/*.json here, commit, push
```

Full runbook (commands, expected output, failure handling, rollback, incident
table): **`docs/level_release.md`** in the `PixelFactory` repo.
Format and architecture: **`docs/hot_update.md`** there.

## Rules for this repo

- **Never delete old `bundle-v*.json` files.** Clients may still be on v(N-1),
  and the append→full fallback needs the current version's full bundle to be
  reachable. Keeping history is also what makes "revert the manifest" a possible
  rollback. Each level is ~3 KB, so ten versions of a 250-level bundle is ~3 MB —
  there is no cost worth the risk.
- **`latestVersion` must never go backwards.** A client treats
  `latestVersion <= its own` as "already up to date" and downloads nothing, so
  publishing a smaller number is silently ignored by every device that already
  updated. (The generator refuses to do this.)
- **Level ids are never reused** and always sort after the built-in levels
  (built-ins run to L112, so remote levels start at L113). Reusing an id would
  attach an existing player's completion record to different content.
- **No analytics, no counters, no query parameters.** The client sends nothing
  but a plain GET; that is what keeps the App Store privacy questionnaire at
  "Data Not Collected" across every category and the privacy policy at "we
  collect nothing". Do not add a hit counter, a beacon, or a "how many devices
  pulled vN" endpoint here.
- GitHub Pages caches for ~5 minutes. A stale manifest right after a push is not
  a bug — wait, then verify with `curl -sI`.

## Files

| File | What it is |
|---|---|
| `manifest.json` | ~230 bytes, fetched on every app launch. Carries `latestVersion`, `minAppVersion`, and the `full` / optional `append` bundle references with their checksums |
| `bundle-v<N>.json` | Full remote level set, compact JSON, fetched only when the version changes |
| `bundle-v<N>-append.json` | Optional delta bundle (new levels only) for clients sitting exactly on `baseVersion` |
