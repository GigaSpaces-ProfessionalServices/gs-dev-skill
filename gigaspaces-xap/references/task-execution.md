# XAP Task Execution

Tasks allow sending computation to where the data lives — avoiding network round-trips for large datasets.

## Task Types

| Type | Scope | Use when |
|------|-------|----------|
| `Task<R>` | Single routed partition | Processing an entity identified by its routing key |
| `DistributedTask<R,T>` | All partitions → reduce | Scatter-gather: aggregate across the whole cluster |
| `DurableTask<R,T>` | All partitions → reduce | Long-running jobs; survives partition failover |

---

## Task (Single Partition, Colocated)

Runs on the partition that owns the routing key passed to `execute(routingKey, task)`.

```java
package com.example.tasks;

import org.openspaces.core.executor.Task;
import org.openspaces.core.executor.TaskGigaSpace;
import org.openspaces.core.GigaSpace;
import com.j_spaces.core.client.SQLQuery;
import com.example.model.Order;
import java.io.Serializable;

public class AccountSumTask implements Task<Double>, Serializable {

    private final String accountId; // routing key

    public AccountSumTask(String accountId) {
        this.accountId = accountId;
    }

    // @TaskGigaSpace: injected by XAP on the server; MUST be transient
    @TaskGigaSpace
    private transient GigaSpace gigaSpace;

    @Override
    public Double execute() throws Exception {
        SQLQuery<Order> query = new SQLQuery<>(Order.class, "accountId = ?");
        query.setParameter(1, accountId);

        Order[] orders = gigaSpace.readMultiple(query, 1000);
        double total = 0;
        for (Order o : orders) {
            total += o.getAmount();
        }
        return total;
    }
}
```

**Invoking a routed Task:**
```java
// Routes to the partition owning accountId "ACC-001"
AsyncFuture<Double> future = gigaSpace.execute(new AccountSumTask("ACC-001"), "ACC-001");
Double result = future.get(5, TimeUnit.SECONDS); // always bound the wait
```

---

## DistributedTask (All Partitions → Reduce)

Sent to every partition. Each partition executes `execute()` independently; results flow back to the client's `reduce()`.

```java
package com.example.tasks;

import com.gigaspaces.async.AsyncResult;
import org.openspaces.core.executor.DistributedTask;
import org.openspaces.core.executor.TaskGigaSpace;
import org.openspaces.core.GigaSpace;
import com.example.model.Payment;
import com.example.model.Merchant;
import com.example.model.CategoryType;
import com.j_spaces.core.client.SQLQuery;
import java.io.Serializable;
import java.util.List;

public class CountPaymentsByCategoryTask
        implements DistributedTask<Integer, Long>, Serializable {

    private final CategoryType category;

    public CountPaymentsByCategoryTask(CategoryType category) {
        this.category = category;
    }

    @TaskGigaSpace
    private transient GigaSpace gigaSpace;

    @Override
    public Integer execute() throws Exception {
        // Runs on ONE partition — query against local data only.
        // Payment is routed by merchantId, so first find merchants in this
        // category, then count only their payments that live on this partition.
        SQLQuery<Merchant> merchantQuery = new SQLQuery<>(Merchant.class, "category = ?");
        merchantQuery.setParameter(1, category);
        // Demonstration only — readMultiple doesn't scale to large result sets, and
        // Integer.MAX_VALUE isn't a real bound. Prefer SpaceIterator (see space-operations.md
        // § SpaceIterator) with a real batch size for anything beyond a handful of entries.
        Merchant[] merchants = gigaSpace.readMultiple(merchantQuery, Integer.MAX_VALUE);

        int count = 0;
        for (Merchant m : merchants) {
            SQLQuery<Payment> paymentQuery = new SQLQuery<>(Payment.class, "receivingMerchantId = ?");
            paymentQuery.setParameter(1, m.getMerchantAccountId());
            count += gigaSpace.count(paymentQuery);
        }
        return count; // scoped to this partition
    }

    @Override
    public Long reduce(List<AsyncResult<Integer>> results) throws Exception {
        long total = 0;
        for (AsyncResult<Integer> r : results) {
            if (r.getException() != null) throw r.getException();
            total += r.getResult();
        }
        return total;
    }
}
```

**Invoking a DistributedTask (broadcast):**
```java
// Sent to ALL partitions; no routing key
AsyncFuture<Long> future = gigaSpace.execute(new CountPaymentsByCategoryTask(CategoryType.FOOD));
// Always bound the wait — an unbounded get() blocks forever if a partition is slow or unreachable.
Long count = future.get(5, TimeUnit.SECONDS);
```

**Return meaningful data from `reduce()`, not just a bare count.** A `DistributedTask<Integer, Long>`
that reduces to a single `Long` throws away exactly the information callers usually need next — e.g.
which category or partition the changes came from. Prefer reducing into structured data, such as a
map of count-by-category, so the caller doesn't need a follow-up query to answer the next question:

```java
public class CountPaymentsByCategoryTask
        implements DistributedTask<Map<CategoryType, Long>, Map<CategoryType, Long>>, Serializable {
    // execute() returns a Map<CategoryType, Long> of this partition's local counts

    @Override
    public Map<CategoryType, Long> reduce(List<AsyncResult<Map<CategoryType, Long>>> results) throws Exception {
        Map<CategoryType, Long> total = new HashMap<>();
        for (AsyncResult<Map<CategoryType, Long>> r : results) {
            if (r.getException() != null) throw r.getException();
            r.getResult().forEach((k, v) -> total.merge(k, v, Long::sum));
        }
        return total;
    }
}
```

---

## DurableTask (Long-Running, Survives Failover)

`DurableTask` is an extension of `DistributedTask` for long-running or resumable jobs.
- Must be annotated with `@SupportCodeChange` for hot-redeploy.
- Implements `cancel()` to support graceful interruption.
- `name()` and `description()` allow monitoring via the admin API.

```java
package com.example.tasks;

import com.gigaspaces.annotation.SupportCodeChange;
import com.gigaspaces.async.AsyncResult;
import com.gigaspaces.client.iterator.SpaceIterator;
import com.j_spaces.core.client.SQLQuery;
import org.openspaces.core.GigaSpace;
import org.openspaces.core.executor.DurableTask;
import org.openspaces.core.executor.TaskGigaSpace;
import java.io.Serializable;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

@SupportCodeChange(id = "1")  // increment each redeployment
public class ReconciliationTask implements DurableTask<Integer, Long>, Serializable {

    // Whether to auto-start on space initialization
    @Override
    public boolean isAutoStart() { return false; }

    @TaskGigaSpace
    private transient GigaSpace gigaSpace;

    // Volatile so cancel() on another thread is visible
    private transient volatile boolean cancelled = false;

    @Override
    public Integer execute() throws Exception {
        // Use SpaceIterator for large datasets — never unbounded readMultiple
        SQLQuery<Object> query = new SQLQuery<>(Object.class, "");
        SpaceIterator<Object> iterator = gigaSpace.iterator(query);
        AtomicInteger processed = new AtomicInteger();

        while (iterator.hasNext() && !cancelled) {
            Object entry = iterator.next();
            // ... do reconciliation work ...
            processed.incrementAndGet();
            if (processed.get() % 1000 == 0) {
                Thread.sleep(10); // yield to avoid starving other operations
            }
        }
        return processed.get();
    }

    @Override
    public Long reduce(List<AsyncResult<Integer>> results) throws Exception {
        long total = 0;
        for (AsyncResult<Integer> r : results) {
            if (r.getException() != null) throw r.getException();
            total += r.getResult();
        }
        return total;
    }

    @Override
    public boolean cancel() throws Exception {
        cancelled = true;
        return true;
    }

    @Override
    public String name() { return "reconciliation-job"; }

    @Override
    public String description() { return "Full-cluster reconciliation of all entries"; }
}
```

**Invoking a DurableTask:**
```java
AsyncFuture<Long> future = gigaSpace.execute(new ReconciliationTask());
// DurableTask is long-running by design — don't block on it with future.get(), even with a
// timeout (a short one fires spuriously; a long one defeats the point of a "bound"). Use the
// listener pattern instead (see AsyncFuture Patterns).
future.setListener(new AsyncFutureListener<Long>() {
    @Override
    public void onResult(AsyncResult<Long> result) {
        if (result.getException() != null) {
            log.error("Reconciliation failed", result.getException());
        } else {
            log.info("Reconciliation processed {} entries", result.getResult());
        }
    }
});

// To cancel before completion:
future.cancel(true);
```

---

## Routing vs Broadcast

| | `gigaSpace.execute(task, routingKey)` | `gigaSpace.execute(distributedTask)` |
|---|---|---|
| Sent to | Single partition owning the routing key | All partitions |
| Use case | Single-entity processing | Cluster-wide aggregation |
| Returns | `AsyncFuture<R>` | `AsyncFuture<T>` (reduced) |

---

## AsyncFuture Patterns

**Prefer the timeout variant by default.** A plain `future.get()` blocks the caller indefinitely if
a partition is slow, hung, or unreachable — always bound it with `get(timeout, unit)` unless a
listener-based callback (non-blocking) is a better fit.

```java
// Timeout — preferred default
Double result = future.get(5, TimeUnit.SECONDS);

// Non-blocking with callback
future.setListener(new AsyncFutureListener<Double>() {
    @Override
    public void onResult(AsyncResult<Double> result) {
        if (result.getException() != null) {
            log.error("Task failed", result.getException());
        } else {
            processResult(result.getResult());
        }
    }
});

// Blocking get — avoid; no bound on how long the caller can be stuck waiting
Double result = future.get();
```

---

## Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| `private GigaSpace gigaSpace` (non-transient) | `@TaskGigaSpace private transient GigaSpace gigaSpace` |
| Hold non-serializable objects in task fields | Task class + all non-transient fields must be `Serializable` |
| Use `readMultiple(..., Integer.MAX_VALUE)` inside task | Use `SpaceIterator` with bounded batch size |
| Forget `@SupportCodeChange` on DurableTask | Required for hot code reload without GSC restart |
| Block inside `execute()` with no cancel check | Check `cancelled` flag periodically in long loops |
| `reduce()` returns a bare count (e.g. `Long`, `Integer`) | Reduce into structured data (e.g. `Map<Category, Long>`) so callers don't need a follow-up query |
| Unbounded `future.get()` on a DistributedTask | Blocks the caller indefinitely if a partition is slow or unreachable — use `future.get(timeout, TimeUnit)` |
