---
name: xap-offheap-blobstore
description: >
  Expert guidance for the GigaSpaces off-heap blobstore and its "read values in place"
  optimization: deploying an off-heap space, query conditions tested against the off-heap block,
  built-in and custom aggregators that declare their paths (getAggregatedPaths) so no entry is built,
  space iterators that apply the query projection, and the read-modify-write data-loss pitfall
  that projection causes. Use when the user mentions off-heap, offheap, blobstore,
  blob-store, off-heap-blob-store, OffHeapBlobStoreHandler, space.mx.off-heap, cache-entries-percentage,
  blob-store-data-policy, getAggregatedPaths, declaredPath, "An aggregator asked for the entry",
  SpaceIterator projection, or PREFETCH_UIDS. Requires GigaSpaces 18.0.1 or later and JDK 25.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# GigaSpaces Off-Heap Blobstore Skill

An off-heap blobstore stores every space entry as a block of serialized bytes outside the Java heap.
**Before 18.0.1**, any query, aggregation or space iterator that touched an entry first decoded the
whole block into a full entry object, even to read one property, and even when the entry did not
match. **From 18.0.1**, the space reads the values it needs directly from the block:

- **Query conditions** are tested against the block. Entries that don't match are dropped without
  being built.
- **Aggregations** whose aggregators all declare the paths they read build no entry at all. Built-in
  aggregators already declare their paths. Custom aggregators can opt in.
- **Space iterators with a projection** (`CURSOR` and `PREFETCH_UIDS`) assemble their results from
  the block.

There is also **one behavior change that can silently lose data**: a space iterator now applies the
query projection, as `read`/`readMultiple` always have. This applies to **on-heap spaces too**, not
just off-heap ones. See `references/iterators-and-projection.md`.

**Target version: GigaSpaces 18.0.1+ on JDK 25.** The in-place read optimization is a GigaSpaces
18.0.1 feature.

---

## Reference Files

**MANDATORY**: Read the relevant reference file(s) below **before generating any code or answering
any question** in this domain. Select by scenario, then read using paths relative to this skill's
own directory (`references/<file>.md`).

| Reference Path | Covers |
|------|--------|
| `references/deploy-and-verify.md` | Deploying an off-heap space (`space.mx.*` properties, or `pu.xml` with `<blob-store:off-heap-blob-store>`), the internal-cache setting that decides whether reads hit the block, container log lines to check, and how to measure "entries built" with a control |
| `references/queries-and-aggregators.md` | Query conditions tested against the block (incl. `change`), which built-in aggregators build nothing, writing a custom aggregator that opts in via `getAggregatedPaths()`, and the `IllegalStateException` raised by one that breaks the rule |
| `references/iterators-and-projection.md` | Space iterators now applying the projection, `CURSOR` vs `PREFETCH_UIDS` behavior, and finding/fixing read-modify-write loops that now overwrite non-projected properties with `null` |
| `references/where-it-does-not-apply.md` | Internal cache, space filters, tiered storage / RocksDB, `readByIds`, unprojected reads, OR-of-pairs conditions, nested paths, plus what 18.0.1 does for each operation in the lab |

## Quick Decision Guide

```
User wants to...
  ├── Deploy / configure an off-heap space                       → deploy-and-verify.md
  ├── Confirm they're actually getting the speed-up              → deploy-and-verify.md
  ├── Know whether a query or built-in aggregation benefits       → queries-and-aggregators.md
  ├── Write a custom aggregator that's fast on off-heap           → queries-and-aggregators.md
  ├── Debug "An aggregator asked for the entry of uid ..."        → queries-and-aggregators.md
  ├── Upgrade an off-heap app to 18.0.1 safely                    → iterators-and-projection.md (first)
  │                                                                 + queries-and-aggregators.md
  ├── Explain why an iterator now returns null properties         → iterators-and-projection.md
  └── Explain why an operation still builds every entry           → where-it-does-not-apply.md
```

**Whenever a user is upgrading any XAP app to 18.0.1 or later (off-heap or not)**, read
`iterators-and-projection.md` and ask about read-modify-write loops over projected iterators **even
if they didn't mention it**. It is the only change here that breaks existing code, and it does so
without raising an exception.

---

## Cross-Cutting Rules

1. **"Built entries" is the metric to watch, not time.** On small data sets the time barely
   changes. The count of entries the space decoded from the block shows whether the in-place path
   was taken.
2. **Zero built entries means success only if a control shows non-zero.** Run an operation that must
   build every entry (`readMultiple` with no projection, or `maxEntry`) first. If the control shows
   zero, entries are being served from the blobstore internal cache, and the other numbers mean
   nothing.
3. **The aggregator rule:** *If you declare your paths, read only those paths and only through
   `getPathValue`. If you need the entry itself, don't declare paths.*
4. **A path declaration is not inherited.** A class that extends a built-in aggregator or
   `AbstractPathAggregator` without overriding `getAggregatedPaths()` itself is handed the full
   entry, as before. That is safe, but it doesn't get the speed-up.
5. **Keep selection in the query, not in the code.** Entries the query condition rejects are never
   built. Entries handed to your code (a custom `ChangeOperation`, an aggregator that filters
   internally, a client loop) are built.
6. **Only the off-heap blobstore benefits.** Tiered storage and a persistent (RocksDB) blobstore do
   not keep entries as blocks the space can read in place.

## Relationship to `gigaspaces-xap`

`gigaspaces-xap`'s `custom-aggregators.md` covers custom aggregators in general (`AbstractPathAggregator`,
serialization, index optimization). This skill covers only what changes for off-heap spaces on
18.0.1+. Its `InListAggregator` example calls `context.getRawEntry()` and doesn't declare paths, so it
keeps working unchanged on 18.0.1. **Never add `getAggregatedPaths()` to an aggregator that calls
`getRawEntry()` or `getServerEntry()`.** On an off-heap space it then fails with
`IllegalStateException`.

---

## Imports Cheat Sheet

```java
// Aggregators
import com.gigaspaces.query.aggregators.SpaceEntriesAggregator;        // getAggregatedPaths() (18.0.1+)
import com.gigaspaces.query.aggregators.AbstractPathAggregator;        // declaredPath() (18.0.1+)
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext; // getPathValue(String)
import com.gigaspaces.query.aggregators.AggregationSet;

// Iterators and projection
import com.gigaspaces.client.iterator.SpaceIterator;
import com.gigaspaces.client.iterator.SpaceIteratorConfiguration;
import com.gigaspaces.client.iterator.SpaceIteratorType;               // CURSOR, PREFETCH_UIDS
import com.j_spaces.core.client.SQLQuery;                              // setProjections(...)
import com.gigaspaces.client.WriteModifiers;                           // PARTIAL_UPDATE
import com.gigaspaces.client.ChangeSet;
```

---

## Provenance

Distilled from `gs-core-dev-training` → `lab22-offheap_blobstore-example`. Everything in
this skill was reproduced on the 18.0.1 GA release: the lab was built against
`gigaspaces.version` 18.0.1 (after `gs.sh maven install`) and all six examples were run on JDK 25
against a local grid (`host run-agent --auto`, the three lab spaces deployed as in
`deploy-and-verify.md`). Every example printed its own "As expected" verdict, and every count
matched. Separate checks on the same grid confirmed that an aggregator calling `getRawEntry()`
after declaring paths fails the same way as `getServerEntry()`, and that the iterator projection
change also applies to a plain on-heap space.

**To re-verify after a release**, repeat the same build and run against it and check for drift.
