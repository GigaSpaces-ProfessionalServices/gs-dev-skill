# Mirror Service: Asynchronous Persistency & Exception Handling

The Mirror is a dedicated Processing Unit that asynchronously replicates space operations from
partitioned primaries into a backing store (typically via Hibernate) — "write-behind" persistence
that keeps the space itself off the write's critical path. `@SpaceClass(persist = true)` (see
`pojo-model.md`) marks a type for this; `mirrored="true"` on the source space's embedded-space
element and an `<os-core:mirror>` PU on the other end wire it up.

**Whenever a project has a mirror PU, check whether it wires a `PersistencyExceptionHandler` before
treating the persistence setup as complete.** A mirror with no exception handler still runs — which
is exactly what makes it easy to miss in review — but it fails silently in production the first time
the database rejects a write.

---

## Why this needs deliberate handling

**Default behavior with no handler configured:** when the sync endpoint throws (a constraint
violation, a down database, a timeout — anything), the exception propagates back to the *primary*
space instance and is logged there. The primary then **retries that failed batch indefinitely**
until it succeeds. There is no built-in backoff or give-up.

That default is safe in the sense that no data is silently dropped, but it has an operational cost
that's easy to miss until it happens: a single poison entry (e.g. a row that will never satisfy a DB
constraint) blocks that replication channel forever, and the primary's redo log — which exists to
hold operations not yet acknowledged by the mirror — grows without bound behind it. Depending on
redo-log capacity settings, that ends in either unbounded memory growth on the primary or writes
being rejected once the log fills.

So "no exception handler" isn't neutral — it's an implicit choice of the *retry-forever-and-block*
policy. That may genuinely be the right policy for some data (see the alternatives table below), but
it should be a deliberate choice, not the thing you get by not thinking about it.

Docs: [Asynchronous Persistency with the Mirror](https://docs.gigaspaces.com/latest/dev-java/asynchronous-persistency-with-the-mirror.html),
[Mirror Advanced / Exception Handling](https://docs.gigaspaces.com/latest/dev-java/async-persistency-mirror-advanced.html).

---

## Wiring: `pu.xml`

```xml
<bean id="hibernateSpaceSyncEndpoint"
      class="org.openspaces.persistency.hibernate.DefaultHibernateSpaceSynchronizationEndpointFactoryBean">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>

<bean id="mirrorExceptionHandler" class="com.example.persistency.MyMirrorExceptionHandler"/>

<!-- Wraps the real sync endpoint so persistence failures route through mirrorExceptionHandler
     instead of falling through to the default retry-forever-and-block behavior above. -->
<bean id="exceptionHandlingSpaceSyncEndpoint"
      class="org.openspaces.persistency.patterns.SpaceSynchronizationEndpointExceptionHandler">
    <constructor-arg ref="hibernateSpaceSyncEndpoint"/>
    <constructor-arg ref="mirrorExceptionHandler"/>
</bean>

<os-core:mirror id="mirror" url="/./mirror-service"
                space-sync-endpoint="exceptionHandlingSpaceSyncEndpoint"
                operation-grouping="group-by-replication-bulk">
    <os-core:source-space name="my-space" partitions="2" backups="1"/>
</os-core:mirror>
```

`operation-grouping="group-by-replication-bulk"` is the mirror default and what the rest of this
file assumes. Under it, `onException`'s `data` argument is an `OperationsBatchData` whose batch can
mix operations from multiple unrelated transactions/sources — there's no guarantee a retry redelivers
"the same batch" with the same composition. The other mode, `group-by-space-transaction`, delivers
one `TransactionData` per transaction instead (with transaction metadata available), which is a
different shape entirely — the per-entry retry-counting pattern below does not apply to it as-is.

---

## Rethrow semantics

`onException` communicates its decision back to XAP by whether it rethrows:

| Behavior | Meaning |
|---|---|
| Rethrow (e.g. `throw new RuntimeException(e)`) | Tell XAP to keep the batch in the primary's redo log and retry replication |
| Return normally (don't rethrow) | Tell XAP the batch was handled — it moves on, and the redo log entry is cleared |

That signal is **per batch, not per entry** — there is no API to say "retry these 3 entries, drop
these other 2." A handler that wants different outcomes for different entries within one batch (e.g.
bounded retry + dead letter) has to decide the outcome for the *whole batch* based on the entries in
it, which is what the pattern below does.

---

## Bounded-retry / dead-letter boilerplate

This is a *starting point for an actual policy decision*, not a drop-in production-ready solution —
see the alternatives table and the limitations list below before adopting it as-is. It retries a
failing entry a fixed number of times (tracked per entry, by type+UID, since the enclosing batch
composition isn't stable across retries), then diverts it to a dead-letter path instead of retrying
forever.

```java
package com.example.persistency;

import com.gigaspaces.sync.DataSyncOperation;
import com.gigaspaces.sync.OperationsBatchData;
import org.openspaces.persistency.patterns.PersistencyExceptionHandler;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class MyMirrorExceptionHandler implements PersistencyExceptionHandler {

    /** Number of consecutive failures for the same entry before it's diverted to the dead letter path. */
    private static final int MAX_ATTEMPTS = 3;

    private final Logger log = LoggerFactory.getLogger(MyMirrorExceptionHandler.class);

    // CAVEAT: entries are only ever removed here on dead-letter (below). An entry that fails once
    // and then succeeds on retry leaves its counter in this map forever — there is no "success"
    // callback on this interface to clean it up. Fine for occasional, distinct failures; under
    // sustained transient failures (e.g. a brief DB outage touching many different entries) this
    // grows without bound for the life of the mirror PU. If that's a real risk for your workload,
    // bound this (e.g. a size-capped or time-expiring cache such as Caffeine) instead of a plain map.
    private final ConcurrentHashMap<String, AtomicInteger> attemptCounts = new ConcurrentHashMap<>();

    @Override
    public void onException(Exception e, Object data) {
        if (!(data instanceof OperationsBatchData)) {
            // Not group-by-replication-bulk (see mirror-persistence.md) - fall back to
            // always-retry rather than guess at per-entry keys we don't have for this shape.
            log.warn("Error caught in mirror exception handler (unrecognized data shape {})",
                    data == null ? "null" : data.getClass().getName(), e);
            throw new RuntimeException(e);
        }

        DataSyncOperation[] items = ((OperationsBatchData) data).getBatchDataItems();
        boolean anyStillRetryable = false;

        for (DataSyncOperation item : items) {
            String key = entryKey(item);
            int attempts = attemptCounts.computeIfAbsent(key, k -> new AtomicInteger()).incrementAndGet();

            if (attempts >= MAX_ATTEMPTS) {
                log.error("Entry {} failed {} times, diverting to dead letter path instead of retrying again",
                        key, attempts, e);
                handleDeadLetter(item, e);
                attemptCounts.remove(key);
            } else {
                log.warn("Entry {} failed (attempt {}/{}), will retry", key, attempts, MAX_ATTEMPTS, e);
                anyStillRetryable = true;
            }
        }

        // CAVEAT: rethrow/not-rethrow is a whole-batch decision (see "Rethrow semantics" above).
        // Rethrowing here redelivers the ENTIRE batch, including entries that were just
        // dead-lettered a line above - their attempt count restarts from zero next time. That's
        // usually fine (dead-letter is meant to be a last resort, not a hard guarantee of exactly
        // MAX_ATTEMPTS total tries) but be aware "3 attempts" is not a precise contract.
        if (anyStillRetryable) {
            throw new RuntimeException(e);
        }
    }

    private String entryKey(DataSyncOperation item) {
        String typeName = item.supportsGetTypeDescriptor()
                ? item.getTypeDescriptor().getTypeName()
                : item.getDataSyncOperationType().name();
        return typeName + "#" + item.getUid();
    }

    /**
     * CAVEAT: this is a stub. Returning without rethrowing tells XAP the entry is handled, so
     * whatever this method does not do (write to a dead-letter table, publish to a queue, page
     * someone) is data that is now permanently missing from the database with no further signal.
     * A dead-letter path that only logs is equivalent to silently dropping the write - decide what
     * "handled" actually means for this data before shipping this to production.
     */
    protected void handleDeadLetter(DataSyncOperation item, Exception e) {
        log.error("Entry {} exhausted {} retries and has no dead-letter action implemented - " +
                        "this data is now lost from the database's perspective",
                entryKey(item), MAX_ATTEMPTS, e);
    }
}
```

---

## This is one policy — pick deliberately

| Policy | Behavior | Fits when |
|---|---|---|
| **No handler (XAP default)** | Retry the batch forever; blocks that replication channel; redo log grows | Data must never be lost and you have alerting on redo-log growth / stuck channels so a stuck poison entry gets human attention quickly |
| **Bounded retry + dead letter** (boilerplate above) | Retry a few times, then divert and move on | Transient failures (brief DB blips, lock contention) should self-heal, but a permanently-invalid entry shouldn't block healthy ones behind it forever — *and* the dead-letter path is a real durable sink (table, queue, alert), not a log line |
| **Immediate dead letter, no retry** | First failure goes straight to the dead-letter path | Failures are expected to be permanent (e.g. schema/constraint mismatches caught late), so retrying is pure overhead |
| **Rethrow always (explicit passthrough handler)** | Same effect as no handler, but explicit and lets you add metrics/alerting in `onException` before rethrowing | You want the safety of retry-forever but also want visibility (a counter, a page) the default silent-log doesn't give you |

None of these is more "correct" than the others in the abstract — it depends on whether losing an
entry or blocking replication is the worse failure mode for the data in question, which is a
business/domain call this skill can't make for you.

---

## Known limitations of the boilerplate above

| Limitation | Detail |
|---|---|
| Dead-letter is a stub | `handleDeadLetter` only logs — must be replaced with a real sink before this is production-safe |
| Retry-count map never shrinks on success | Only evicted on dead-letter; long-running mirrors under sustained transient failures can accumulate orphaned entries indefinitely |
| Coupled to `group-by-replication-bulk` | Falls back to always-retry (safe, but silently disables bounded-retry) under `group-by-space-transaction` |
| Batch-level rethrow granularity | A batch containing both retryable and just-dead-lettered entries redelivers all of them together; "N attempts" isn't a hard per-entry guarantee |

---

## Anti-Patterns

| ❌ Don't | ✅ Do instead |
|---|---|
| Deploy a mirror PU with no `PersistencyExceptionHandler` and no plan for what happens on a persistent failure | At minimum, wrap the sync endpoint in an explicit passthrough handler that logs/alerts before rethrowing — see "Rethrow always" above — so the retry-forever behavior is a visible choice, not a silent default |
| Copy the bounded-retry/dead-letter boilerplate with `handleDeadLetter` left as a log statement | Implement a real dead-letter sink (table, queue, alert) — otherwise this pattern silently loses data after `MAX_ATTEMPTS`, same as a bug, just quieter |
| Assume `operation-grouping="group-by-space-transaction"` works with this same handler unchanged | `data` becomes `TransactionData`, not `OperationsBatchData` — the per-entry key logic needs to change for that shape |
| Treat "N attempts" as a hard guarantee | Batch-level rethrow can redeliver dead-lettered entries bundled with retryable ones, resetting their count |
