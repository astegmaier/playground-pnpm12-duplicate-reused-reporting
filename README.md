# pnpm 12 reports every reused package twice during `pnpm dedupe`

This repository demonstrates a pnpm 12.5.1 progress-reporting bug in a warm full `pnpm dedupe` run.

The fixture contains two equivalent projects:

- `v11/` uses `packageManager: "pnpm@11.27.0"` and a pnpm 11 lockfile.
- `v12/` uses `packageManager: "pnpm@12.5.1"` and a pnpm 12 lockfile.

Both projects depend only on `express@4.21.2`. That resolves to 69 packages in each project.

## Run the reproduction

Requirements: Node.js and pnpm installed directly, not through Corepack. pnpm reads each fixture's `packageManager` field and runs the requested version.

From the repository root, run pnpm 11:

```sh
cd v11
pnpm --version
pnpm install --frozen-lockfile
pnpm dedupe
```

Then run pnpm 12:

```sh
cd ../v12
pnpm --version
pnpm install --frozen-lockfile
pnpm dedupe
```

The version commands should print `11.27.0` and `12.5.1`.

The final pnpm 11 progress line is:

```text
Progress: resolved 69, reused 69, downloaded 0, added 0, done
```

The final pnpm 12 progress line is:

```text
Progress: resolved 69, reused 138, downloaded 0, added 0, done
```

The pnpm 12 counter reports twice as many reused packages as were resolved.

## Raw progress events

The aggregate counter reflects duplicate `pnpm:progress` events, not 138 distinct packages.

After a clean install, capture one dedupe run:

```sh
pnpm dedupe --reporter=ndjson > /tmp/pnpm-progress.ndjson 2>&1
```

Counting `found_in_store` events and unique package IDs gives:

| Version | `found_in_store` events | Unique package IDs | IDs reported more than once |
|---|---:|---:|---:|
| pnpm 11.27.0 | 69 | 69 | 0 |
| pnpm 12.5.1 | 138 | 69 | 69 |

Every reused package ID is emitted exactly twice by pnpm 12.

## Root Cause Analysis

In this warm fixture, pnpm 12's dedupe command reports each tarball-shaped package's reuse from two separate phases:

1. The dedupe resolution observer checks the store index and emits `found_in_store` while resolving the dependency graph.
2. `emit_warm_snapshot_progress` in the virtual-store linking path emits `found_in_store` again when materialization reuses the same warm package snapshots.

The default reporter increments `reused` for every `found_in_store` event. It does not deduplicate equivalent events from resolution and materialization.

Resolve-time reuse reporting is useful for `pnpm dedupe --lockfile-only` and `pnpm dedupe --check`, because those modes do not run materialization. A full dedupe runs both phases, so the same packages are counted twice.

In this fixture, pnpm 11 emits one materialization reuse event per package during a full dedupe and has no resolve-time dedupe observer.

## Impact

The primary impact is reporting correctness:

- `reused` can exceed `resolved`;
- the progress summary implies that more distinct packages were reused than exist in the resolved graph;
- the counter is misleading when comparing pnpm versions or investigating install performance.

There is also some unnecessary local work during full dedupe. For each unique tarball resolution observed by the dedupe reporter, pnpm performs a store-index membership check and emits an extra progress event. The reporter folds every duplicate event into its counters and recomputes the progress frame, although terminal writes may be throttled or coalesced. This reproduction is intended to demonstrate the reporting bug, not quantify that overhead.

The duplicate events do not represent additional resolutions. In this fixture, the dependency graph and lockfile remain byte-identical before and after dedupe.
