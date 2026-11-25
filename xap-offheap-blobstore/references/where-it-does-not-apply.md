# Where In-Place Reads Don't Apply, and What 18.0.1 Does

## Cases that still build entries (or don't read the block)

| Situation | What happens | Why |
|---|---|---|
| Blobstore **internal cache** enabled (default 20% of heap) | Cached entries are served from the heap. The block isn't read, so even a control shows 0 built | They're already built objects. Not a problem to tune away, because a warm cache is faster than reading the block. In-place reads serve the entries *not* in the cache |
| **Space filter** on the read (e.g. `AFTER_READ_MULTIPLE`, code 12) | Batch reads/iterators with a projection take the regular path and build every entry | A filter receives the **whole** entry before the projection is applied. A packet built from the block would hold only the projected properties |
| Aggregation on a filtered space | Still in place (0 built) | Aggregators don't pass through read filters |
| `PREFETCH_UIDS` + projection on a filtered space | Builds all but the first batch (lab: 900 of 1000) | The first batch arrives with the ids as part of an aggregation (unfiltered, in any version). The remaining entries are read by id with a regular, filtered read |
| **Tiered storage**, **persistent (RocksDB) blobstore** | No gain | Only the off-heap blobstore keeps entries as blocks the space can read in place |
| Entry already built on the heap | Served as is | Nothing to save |
| `readByIds`; any read/iteration **without a projection** | Builds as before | The client asked for whole entries |
| `maxEntry` / `minEntry` aggregators | Builds every entry | They return an entry |
| A custom aggregator that doesn't declare paths | Builds every entry | Opt-in only. See `queries-and-aggregators.md` |
| A condition that is an **OR of compound pairs** | Builds every examined entry | Not decided from the block. Plain AND conditions are |
| Property type that can't be read in place, or a projection that names a **nested path** | Regular path | Only plain properties are read from the block |

Lab, all three spaces (number of entries built, out of 1000, including
1900 `CurrencyAuditFilter saw payment` lines in the filtered space's log: 1000 + 900):

| Space | `maxEntry` | `sum` | `CURSOR` projected | `PREFETCH_UIDS` projected |
|---|---:|---:|---:|---:|
| off-heap, cache 0 | 1000 | 0 | 0 | 0 |
| off-heap, default cache | 0 | 0 | 0 | 0 |
| off-heap, cache 0, after-read filter | 1000 | 0 | 1000 | 900 |

## What 18.0.1 does, operation by operation

Lab results with the internal cache off (`offheapDemo`, 1000 entries, 2 partitions):

- **`readMultiple where amount > 990`** (non-indexed): the condition is tested in the block, and
  only the 10 matching entries are built.
- **`sum(amount)`**: 0 entries built. Built-in aggregators declare their paths.
- **A custom aggregator that declares its paths**: 0 entries built.
- **An aggregator that declares its paths but calls `getServerEntry()`**: fails with
  `IllegalStateException`. See `queries-and-aggregators.md`.
- **`CURSOR` with a projection**: the projection is applied (`currency` is null on all 1000) and
  0 entries are built.
- **`PREFETCH_UIDS` with a projection**: the same, `currency` null on all 1000, 0 entries built.
- **Read-modify-write with a plain `write`** after a projected iteration: all 1000 currencies are
  lost. Use `PARTIAL_UPDATE` or `change`; see `iterators-and-projection.md`.

**The gain scales with entry width and selectivity.** Wide types that are read for few properties
gain the most. A narrow 3-property type gains much less. Set expectations accordingly.

Upstream docs: https://docs.gigaspaces.com/latest/dev-java/aggregators.html
