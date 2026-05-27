# XAP Custom Aggregators

Custom aggregators run **server-side, colocated with the data** in each partition. Each partition executes `aggregate()` locally; results are sent back to the client where `aggregateIntermediateResult()` assembles the final answer.

**Use custom aggregators when built-in aggregators (`AggregationSet.sum/count/avg/max/min/groupBy`) don't cover your logic.**

---

## Required Interfaces and Inheritance

```java
// Always extend AbstractPathAggregator<INTERMEDIATE_RESULT_TYPE>
// Always implement Externalizable (not just Serializable — sent to every partition)
// Always annotate with @SupportCodeChange for hot-redeploy support
```

---

## Minimal Custom Aggregator Template

```java
package com.example.aggregators;

import com.gigaspaces.annotation.SupportCodeChange;
import com.gigaspaces.query.aggregators.AbstractPathAggregator;
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext;
import java.io.Externalizable;
import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;

/**
 * Sums a Double field across all partitions, server-side.
 * Extend AbstractPathAggregator<T> where T is the per-partition intermediate result type.
 */
@SupportCodeChange(id = "1") // increment id on each redeploy to force class reload
public class SumDoubleAggregator extends AbstractPathAggregator<Double>
        implements Externalizable {

    private double partitionSum = 0.0;

    // XAP requires a no-arg constructor for deserialization
    public SumDoubleAggregator() { super(); }

    public SumDoubleAggregator(String propertyPath) {
        super();
        setPath(propertyPath); // inherited: which property to aggregate
    }

    // --- Server-side: called once per matching entry on the partition ---
    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        Double value = (Double) context.getPathValue(getPath());
        if (value != null) {
            partitionSum += value;
        }
    }

    // --- Server-side: return this partition's result to the client ---
    @Override
    public Double getIntermediateResult() {
        return partitionSum;
    }

    // --- Client-side: merge each partition's intermediate result ---
    @Override
    public void aggregateIntermediateResult(Double partitionResult) {
        if (partitionResult != null) {
            partitionSum += partitionResult;
        }
    }

    // --- Client-side: return final merged result ---
    @Override
    public Double getFinalResult() {
        return partitionSum;
    }

    @Override
    public String getDefaultAlias() { return "sumDouble(" + getPath() + ")"; }

    // Externalizable — must serialize all state fields
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);
        out.writeDouble(partitionSum);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        super.readExternal(in);
        partitionSum = in.readDouble();
    }
}
```

---

## Calling a Custom Aggregator

```java
import com.gigaspaces.query.aggregators.AggregationSet;
import com.j_spaces.core.client.SQLQuery;

// Query scope — only entries matching this predicate are aggregated
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "status = ?");
query.setParameter(1, TransactionStatus.PROCESSED);

// Build aggregation set — can chain multiple aggregators
AggregationSet aggregationSet = new AggregationSet()
        .add(new SumDoubleAggregator("paymentAmount"));   // path = POJO property name

// Execute — runs on every matching partition
com.gigaspaces.query.aggregators.AggregationResult result = gigaSpace.aggregate(query, aggregationSet);

// Retrieve by alias
Double totalAmount = (Double) result.get("sumDouble(paymentAmount)");
// Or by index
Double totalAmount2 = (Double) result.get(0);
```

---

## IN-List Custom Aggregator (from Training Lab 23)

Returns all space entries whose property value is in a given collection — with optional index optimization.

```java
package com.example.aggregators;

import com.gigaspaces.annotation.SupportCodeChange;
import com.gigaspaces.query.aggregators.AbstractPathAggregator;
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext;
import com.gigaspaces.internal.query.RawEntry;
import com.gigaspaces.metadata.index.SpaceIndex;
import com.gigaspaces.metadata.index.SpaceIndexType;
import java.io.Externalizable;
import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.util.*;
import java.util.stream.Collectors;

@SupportCodeChange(id = "1")
public class InListAggregator extends AbstractPathAggregator<ArrayList<Object>>
        implements Externalizable {

    private Collection<Object> matchValues; // values to match against
    private ArrayList<Object> result = new ArrayList<>();

    public InListAggregator() { super(); }

    public InListAggregator(String propertyPath, Collection<Object> matchValues) {
        super();
        setPath(propertyPath);
        this.matchValues = matchValues;
    }

    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        // getRawEntry() is faster than full POJO deserialization
        if (matchValues.contains(context.getPathValue(getPath()))) {
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
     * If true, XAP can skip the full scan and use index lookups instead.
     * Return true when an EQUAL or EQUAL_AND_ORDERED index exists on our path.
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

    /**
     * skipProcessedUidStore: when true, XAP skips tracking processed UIDs.
     * Safe to return true unless you need dedup of entries modified during scan.
     */
    @Override
    public boolean skipProcessedUidStore() { return true; }

    @Override
    public String getDefaultAlias() { return "inList(" + getPath() + ")"; }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);
        out.writeObject(matchValues);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        super.readExternal(in);
        matchValues = (Collection<Object>) in.readObject();
    }
}
```

---

## Injecting GigaSpace into an Aggregator (Server-Side Access)

Aggregators can access the local partition's space via `@GigaSpaceAggregation`:

```java
import org.openspaces.core.GigaSpace;
import org.openspaces.core.aggregators.GigaSpaceAggregation;

@SupportCodeChange(id = "1")
public class EnrichedAggregator extends AbstractPathAggregator<List<MyResult>>
        implements Externalizable {

    // Injected by XAP on the server — MUST be transient (not serialized)
    @GigaSpaceAggregation
    protected transient GigaSpace gs;

    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        // Can read related objects from same partition
        Integer relatedId = (Integer) context.getPathValue(getPath());
        RelatedObject related = gs.readById(RelatedObject.class, relatedId);
        // ... enrich and accumulate
    }
    // ... rest of implementation
}
```

---

## Built-in Aggregators (prefer over custom when sufficient)

```java
import com.gigaspaces.query.aggregators.AggregationSet;

AggregationResult r = gigaSpace.aggregate(query, new AggregationSet()
    .sum("amount")           // sum of a numeric field
    .average("amount")       // average
    .max("amount")           // maximum
    .min("amount")           // minimum
    .count()                 // total matching count
    .maxEntry("amount")      // the full entry with maximum value
    .minEntry("amount")      // the full entry with minimum value
    .groupBy("category",     // GROUP BY — returns Map<Object,AggregationResult>
             new AggregationSet().sum("amount").count())
);

Long total      = (Long)   r.get("count");
Double sumAmt   = (Double) r.get("sum(amount)");
Map<?,?> byCategory = (Map<?,?>) r.get("groupBy(category)");
```

---

## Anti-Patterns

| ❌ Don't | ✅ Do instead |
|---|---|
| Implement `Serializable` only on aggregator | Implement `Externalizable` — aggregators are sent to every partition |
| Forget `@SupportCodeChange` | Add it — required for hot-redeploy without GSC restart |
| Hold non-transient non-serializable fields | Mark XAP-injected fields (`GigaSpace`) as `transient` |
| Forget `super.writeExternal(out)` / `super.readExternal(in)` | Always call `super` — `AbstractPathAggregator` serializes the path |
| Client-side `readMultiple` + loop to aggregate | Use server-side aggregator — avoids moving all data to client |
