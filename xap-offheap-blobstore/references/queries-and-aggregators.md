# Queries and Aggregators on an Off-Heap Space (18.0.1+)

## Query conditions are tested against the block

No client change is needed. Every query gets this. For a condition on a **non-indexed** property,
the space has to examine every candidate. Before 18.0.1 it built each candidate entry in order to
test it. Now it reads the properties named in the condition from the block, and builds only the
entries that match (and only because a read without a projection returns whole entries).

Observed in the lab (1000 entries, `amount` 1..1000):

| Operation | Result | Built |
|---|---|---|
| control: `readMultiple`, no condition | 1000 | 1000 |
| `readMultiple where amount > 990` | 10 | 10 |
| `readMultiple where currency='USD' AND amount > 990` | 3 | 3 |
| `count()` aggregation `where amount > 990` | 10 | 0 |
| `change where amount > 990` (increment) | 10 | 10 |

- **The gain grows with selectivity.** The saving comes from the entries that *don't* match. A
  query that matches almost everything gains almost nothing.
- **`change`** builds only the entries it changes, because a change needs the entry. So keep the
  selection in the `SQLQuery`. A custom `ChangeOperation` that receives every entry and decides
  internally which ones to skip forces all of them to be built.
- **Conditions joined by AND** are tested against the block. **An OR of compound pairs**, such as
  `(currency='USD' AND amount>500) OR (currency='EUR' AND amount>450) OR ...`, is **not**, and every
  entry it examines gets built.

## Built-in aggregators

Single-property built-ins already declare their path, so an aggregation made only of them builds
nothing. Examples are `count`, `sum`, `average`, `minValue` and `maxValue`.

| Aggregation | Built |
|---|---|
| `count()`, `sum(amount)`, `average(amount)`, `maxValue(amount)` | 0 |
| `count() where amount > 500` | 0 |
| `count() where <OR of (currency, amount) pairs>` | all |
| `maxEntry(amount)` / `minEntry(...)` | all. These return an entry, so they need one |

An aggregation is served in place only if **every** aggregator in the `AggregationSet` declares its
paths.

## Custom aggregators: opting in

Override `SpaceEntriesAggregator.getAggregatedPaths()` and return the paths you read. That is a
promise that `aggregate()` needs nothing else from the entry. Return `null` (the default) to mean
"I need the whole entry".

> **The rule:** If you declare your paths, read only those paths and only through
> `context.getPathValue(path)`. If you need the entry itself, don't declare paths.

### Multi-path aggregator

This example counts payments above a per-currency limit, which no built-in can express.

```java
import com.gigaspaces.query.aggregators.SpaceEntriesAggregator;
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext;
import java.util.Map;

public class LargePaymentsAggregator extends SpaceEntriesAggregator<Long> {
    private static final long serialVersionUID = 1L;

    private final Map<String, Double> limits; // travels to each partition with the aggregator
    private long large;

    public LargePaymentsAggregator(Map<String, Double> limits) { this.limits = limits; }

    @Override public String getDefaultAlias() { return "largePayments"; }

    /** Declares exactly what aggregate() reads. */
    @Override
    public String[] getAggregatedPaths() { return new String[]{"currency", "amount"}; }

    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        Object currency = context.getPathValue("currency");   // only declared paths,
        Object amount   = context.getPathValue("amount");     // only via getPathValue
        Double limit = limits.get(String.valueOf(currency));
        if (limit != null && amount instanceof Number && ((Number) amount).doubleValue() > limit)
            large++;
    }

    @Override public Long getIntermediateResult() { return large; }

    @Override
    public void aggregateIntermediateResult(Long partitionResult) {
        if (partitionResult != null) large += partitionResult;
    }
}
```

### Single-path aggregator on `AbstractPathAggregator`

`AbstractPathAggregator` provides `declaredPath()`, which turns the path set with `setPath(...)`
into the declaration. The override is still required, because declarations aren't inherited:

```java
public class AmountHistogramAggregator extends AbstractPathAggregator<int[]> {
    private static final long serialVersionUID = 1L;
    private transient int[] histogram;

    @Override public String getDefaultAlias() { return "amountHistogram"; }

    @Override
    public String[] getAggregatedPaths() { return declaredPath(); }   // must be overridden here

    @Override
    public void aggregate(SpaceEntriesAggregatorContext context) {
        Object value = getPathValue(context);                          // reads the setPath() path
        if (!(value instanceof Number)) return;
        if (histogram == null) histogram = new int[4];
        histogram[Math.min(Math.max((int) ((((Number) value).doubleValue() - 1) / 250), 0), 3)]++;
    }

    @Override public int[] getIntermediateResult() { return histogram; }
    @Override public void aggregateIntermediateResult(int[] p) {
        if (p == null) return;
        if (histogram == null) histogram = new int[4];
        for (int i = 0; i < histogram.length && i < p.length; i++) histogram[i] += p[i];
    }
    @Override public int[] getFinalResult() { return histogram; }
}

// usage
AmountHistogramAggregator h = new AmountHistogramAggregator();
h.setPath("amount");
int[] buckets = (int[]) gigaSpace.aggregate(new SQLQuery<>(Payment.class, ""),
        new AggregationSet().add(h)).get(0);
```

### Breaking the rule: what happens

If an aggregator declares paths but reads from `context.getServerEntry()` or
`context.getRawEntry()`, **the aggregation fails**. It doesn't return a wrong number. Exact text,
for both `getServerEntry()` and `getRawEntry()`:

```
java.lang.IllegalStateException: An aggregator asked for the entry of uid ...
after declaring that it reads nothing but its own paths, so the entry was never built.
See SpaceEntriesAggregator.getAggregatedPaths.
```

- The client sees it as the root cause of the `aggregate(...)` exception. The same exception is
  logged as `SEVERE` in the container log of each primary. It appears as one line per primary,
  `offheapDemo SEVERE [com.gigaspaces.space.engine] - Error handling read/take/change. multiple;
  Caused by: java.lang.IllegalStateException: ...`. Don't mistake it for a space fault.
- These are still available after declaring paths: `getPartitionId()`, `getEntryUid()` and
  `getTypeDescriptor()`.
- **Fix with a one-line change**, either one:
  - Read through `context.getPathValue(path)`. This keeps the speed-up.
  - Remove the `getAggregatedPaths()` override. Entries are built again, the old code works, and
    there is no speed-up.
- On 18.0.0 and earlier, the same code returns the correct result, because declared paths are
  ignored there. So the bug surfaces only after an upgrade, and only on an off-heap space.

### Code review checklist for custom aggregators (off-heap, 18.0.1+)

- [ ] Does it override `getAggregatedPaths()`? If not, it's correct but gets no speed-up. Add the
      override if `aggregate()` only reads properties.
- [ ] If it does override it, does `aggregate()` touch `getServerEntry()`, `getRawEntry()`, or any
      path not in the returned array? If so, it will fail.
- [ ] Does it extend a built-in or `AbstractPathAggregator` and rely on the parent's declaration?
      Declarations aren't inherited, so re-declare in the subclass (`return declaredPath();`).
- [ ] Does it filter internally, for example skipping entries in code? Move the filter into the
      `SQLQuery` condition so that rejected entries are never built.
