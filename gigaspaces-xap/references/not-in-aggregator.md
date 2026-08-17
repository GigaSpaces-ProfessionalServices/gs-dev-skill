# NOT IN Custom Aggregator

`field NOT IN (...)` cannot be served by an index the way `field = ?` or `field IN (...)` can. An
`EQUAL`/hash index maps a value to the entries that match it — a positive lookup. There is no
equivalent probe for "every entry whose value is *not* one of these" — the engine has to visit
every entry to decide whether it's excluded. So `NOT IN` forces a full scatter-gather scan across
every partition, even when the field is indexed.

**Use a custom aggregator for `NOT IN` instead of a `SQLQuery`/JDBC `NOT IN` clause** whenever the
type holds a non-trivial number of entries. The exclusion check still runs against every entry, but
it runs **server-side, colocated with the data, per partition** — only the entries that pass the
filter are serialized and sent to the client, instead of shipping the whole dataset over the wire
for client-side filtering.

See `custom-aggregators.md` for the base aggregator mechanics (`AbstractPathAggregator`,
`Externalizable`, `@SupportCodeChange`) — this file only covers what's different for `NOT IN`.

---

## Required index

**The field in the `NOT IN` clause must carry `@SpaceIndex(type = SpaceIndexType.EQUAL)`** (or
`EQUAL_AND_ORDERED`). Without an index, `isIndexUsed()` below always returns `false` and every
entry is fully deserialized to evaluate the predicate. With the index, XAP can resolve the path
value from the cached index entry instead of deserializing the whole object just to check one
field — the aggregator still visits every entry (there's no way around that for a negation), but
each visit is cheap.

```java
@SpaceIndex(type = SpaceIndexType.EQUAL)
public TransactionStatus getStatus() { return status; }
```

(`Payment.status` in `billbuddy-domain.md` is already indexed this way — used as the example below.)

---

## NOT IN Custom Aggregator Template

```java
package com.example.aggregators;

import com.gigaspaces.annotation.SupportCodeChange;
import com.gigaspaces.internal.query.RawEntry;
import com.gigaspaces.metadata.index.SpaceIndex;
import com.gigaspaces.metadata.index.SpaceIndexType;
import com.gigaspaces.query.aggregators.AbstractPathAggregator;
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext;
import java.io.Externalizable;
import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.util.*;
import java.util.stream.Collectors;

@SupportCodeChange(id = "1")
public class NotInAggregator extends AbstractPathAggregator<ArrayList<Object>>
        implements Externalizable {

    private Collection<Object> excludeValues; // values to EXCLUDE
    private ArrayList<Object> result = new ArrayList<>();

    public NotInAggregator() { super(); }

    public NotInAggregator(String propertyPath, Collection<Object> excludeValues) {
        super();
        setPath(propertyPath);
        this.excludeValues = excludeValues;
    }

    // --- Server-side: called once per matching entry on the partition ---
    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        // getRawEntry() avoids full POJO deserialization for entries we're about to keep
        if (!excludeValues.contains(context.getPathValue(getPath()))) {
            result.add(context.getRawEntry());
        }
    }

    @Override
    public ArrayList<Object> getIntermediateResult() { return result; }

    @Override
    public void aggregateIntermediateResult(ArrayList<Object> partitionResult) {
        if (partitionResult != null) result.addAll(partitionResult);
    }

    @Override
    public ArrayList<Object> getFinalResult() {
        // Convert RawEntry instances to POJOs on the client side
        return result.stream()
                .map(raw -> (Object) toObject((RawEntry) raw))
                .collect(Collectors.toCollection(ArrayList::new));
    }

    /**
     * If true, XAP can resolve the path value from the index/raw entry instead of a full
     * deserialize-then-check. Still visits every entry — NOT IN can't skip entries the way
     * IN can — but each visit is cheap when the field is indexed.
     */
    @Override
    public boolean skipFullScanSupported(Map<String, SpaceIndex> indexes, boolean isMemoryScan) {
        return !isMemoryScan || isIndexUsed(indexes, isMemoryScan);
    }

    @Override
    public boolean isIndexUsed(Map<String, SpaceIndex> indexes, boolean isMemoryScan) {
        SpaceIndex index = indexes.get(getPath());
        return getFunctionCallColumn() == null && index != null
                && (index.getIndexType() == SpaceIndexType.EQUAL
                    || index.getIndexType() == SpaceIndexType.EQUAL_AND_ORDERED);
    }

    @Override
    public boolean skipProcessedUidStore() { return true; }

    @Override
    public String getDefaultAlias() { return "notIn(" + getPath() + ")"; }

    // Externalizable — must serialize all state fields
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);
        out.writeObject(excludeValues);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        super.readExternal(in);
        excludeValues = (Collection<Object>) in.readObject();
    }
}
```

---

## Calling a NOT IN Aggregator

Put routing/filter conditions that *don't* involve the excluded field in the `SQLQuery` as usual;
the exclusion itself is handled entirely by the aggregator, not by a `NOT IN` clause in SQL.

```java
import com.gigaspaces.query.aggregators.AggregationSet;
import com.j_spaces.core.client.SQLQuery;

// Other filter/routing conditions go here — NOT IN is NOT expressed in the query text
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "receivingMerchantId = ?");
query.setParameter(1, merchantId);

Set<Object> excludedStatuses = new HashSet<>(Arrays.asList(
        TransactionStatus.CANCELLED, TransactionStatus.FAILED));

AggregationSet aggregationSet = new AggregationSet()
        .add(new NotInAggregator("status", excludedStatuses)); // path = indexed POJO property

com.gigaspaces.query.aggregators.AggregationResult result = gigaSpace.aggregate(query, aggregationSet);

@SuppressWarnings("unchecked")
List<Payment> notCancelledOrFailed = (List<Payment>) (List<?>) result.get("notIn(status)");
```

---

## Rewriting a flagged `NOT IN` query

If you see this in code review:

```java
// ❌ Full scatter-gather scan, even though status is @SpaceIndex(EQUAL)
SQLQuery<Payment> query = new SQLQuery<>(Payment.class,
        "receivingMerchantId = ? AND status NOT IN (?, ?)");
query.setParameter(1, merchantId);
query.setParameter(2, TransactionStatus.CANCELLED);
query.setParameter(3, TransactionStatus.FAILED);
Payment[] results = gigaSpace.readMultiple(query, 500);
```

Replace it with: routing/equality conditions stay in the `SQLQuery`; the `NOT IN` predicate moves
into a `NotInAggregator` as shown above.

---

## When to Use / When Not to Use

| Situation | Recommendation |
|---|---|
| `NOT IN` list is small, field is `@SpaceIndex(EQUAL)`, type holds many entries per partition | ✅ Use `NotInAggregator` — server-side filtering avoids shipping excluded entries over the network |
| You only need a count/sum of the non-excluded entries, not the entries themselves | ✅ Still use a custom aggregator, but return the aggregate instead of `RawEntry` objects — see `custom-aggregators.md` for a `Double`/`Long` intermediate-result pattern |
| Field is **not** indexed | ⚠️ Aggregator still works correctly, but `isIndexUsed()` returns `false` — every entry is fully deserialized. Add `@SpaceIndex(type = SpaceIndexType.EQUAL)` first |
| Exclusion list is effectively the whole domain (e.g. excluding 2 of 3 possible enum values) | ❌ Rewrite as a positive `IN` query (`custom-aggregators.md`'s `InListAggregator`, or plain `SQLQuery status IN (...)`) — filtering *for* the remaining value(s) is cheaper than filtering *against* most of them |
| Small, bounded result set already available client-side (e.g. cached reference data) | ❌ Custom aggregator is overkill — plain `SQLQuery` + client-side filter is simpler and fast enough |

---

## Anti-Patterns

| ❌ Don't | ✅ Do instead |
|---|---|
| `SQLQuery` / JDBC `status NOT IN (...)` on a large or growing type | Use a `NotInAggregator` — see above |
| Filter `NOT IN` on an unindexed field and expect it to be fast | Add `@SpaceIndex(type = SpaceIndexType.EQUAL)` to the field before optimizing the query |
| `readMultiple` everything then filter with `!list.contains(...)` in client code | Push the exclusion into a server-side aggregator — avoids transferring excluded entries over the network |
| Omit `isIndexUsed()` / `skipFullScanSupported()` overrides | Implement them — otherwise XAP always falls back to fully deserializing every entry to evaluate the predicate |
| Implement `Serializable` only on the aggregator | Implement `Externalizable` — aggregators are sent to every partition (see `custom-aggregators.md`) |
