# Space Iterators and the Projection Change (18.0.1+)

## What changed

1. **Space iterators now apply the query projection**, as `read` and `readMultiple` always have.
   Properties that aren't projected come back `null`.
   - Before 18.0.1, a `CURSOR` iterator ignored the projection completely.
   - Before 18.0.1, a `PREFETCH_UIDS` iterator ignored it for the first batch from each partition.
2. **On an off-heap space, a projection of plain properties builds no entry.** Both iterator types
   assemble the result from the block.

```java
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "").setProjections("id", "amount");
try (SpaceIterator<Payment> it = gigaSpace.iterator(query,
        new SpaceIteratorConfiguration()
                .setIteratorType(SpaceIteratorType.CURSOR)   // or PREFETCH_UIDS
                .setBatchSize(100))) {
    while (it.hasNext()) {
        Payment p = it.next();   // p.getCurrency() == null: currency was not projected
    }
}
```

Lab results (1000 entries, 2 partitions):

| Iterator | `currency` null | Built |
|---|---|---|
| `CURSOR`, no projection (control) | 0 | 1000 |
| `CURSOR`, projection `id+amount` | 1000 | 0 |
| `PREFETCH_UIDS`, projection `id+amount` | 1000 | 0 |

`PREFETCH_UIDS` runs in two steps:

1. When the iterator is created, it collects the ids of all matching entries and receives the first
   batch along with them.
2. It then reads the rest by id, one batch at a time.

On 18.0.1 neither step builds entries when there is a projection. Without a projection, entries are
built as before, because the client asked for whole entries.

---

## Pitfall: read-modify-write loops now erase data

**This is the one change here that breaks existing code, and it does so silently.** Here is a loop
that was correct before 18.0.1:

```java
// BROKEN on 18.0.1+
SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "").setProjections("id", "amount");
try (SpaceIterator<Payment> it = gigaSpace.iterator(q,
        new SpaceIteratorConfiguration().setIteratorType(SpaceIteratorType.CURSOR))) {
    while (it.hasNext()) {
        Payment p = it.next();
        p.setAmount(p.getAmount() + 1);
        gigaSpace.write(p);                 // or WriteModifiers.UPDATE_OR_WRITE
    }
}
```

The code is the same on both versions; only the outcome changes:

- **Before 18.0.1:** the iterator ignored the projection and returned whole payments, so the write
  stored them whole.
- **18.0.1+:** `currency` is now `null`, and a plain `write` stores that `null` over the existing
  value. There is no exception. In the lab, 1000 of 1000 payments lost their currency.

### Fixes (pick one)

```java
// 1. Partial update: only non-null properties are written; the rest are left alone
gigaSpace.write(p, WriteModifiers.PARTIAL_UPDATE);

// 2. Drop the projection where objects are written back
SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "");

// 3. Best: no read at all, using the change API with the selection in the query
gigaSpace.change(new SQLQuery<>(Payment.class, ""), new ChangeSet().increment("amount", 1.0));
```

In the lab, a plain write lost 1000 of 1000 currencies. With `PARTIAL_UPDATE`,
0 of 1000 were lost, and the amounts were raised. A
caveat on fix 1: `PARTIAL_UPDATE` can't set a property *to* `null` deliberately. If the loop
needs to do that, use `change` (`ChangeSet.unset` / `set`) instead.

### How to find affected code before upgrading

Search the codebase for both of the following in the same flow:

- `gigaSpace.iterator(` (or `SpaceIterator`), with a query that calls `setProjections(` (on
  `SQLQuery`, `IdQuery`, or a template with projections)
- a `write(` / `writeMultiple(` of the iterated object, without `WriteModifiers.PARTIAL_UPDATE`

Also check code that **reads** non-projected fields from iterator results, such as null checks,
`toString`, mapping to DTOs, or equality. It used to see real values and will now see `null`.

**The projection change applies to on-heap spaces too.** On a plain
on-heap space (`gs.sh space deploy onheapCheck`, no blobstore), both `CURSOR` and `PREFETCH_UIDS`
with a projection of `id+amount` returned `currency == null` for 1000 of 1000 entries. The plain
`write` loop above then erased `currency` on all 1000. So audit these loops in **every** app being
upgraded to 18.0.1, not only off-heap ones. Only the "builds no entry" part is specific to off-heap.
