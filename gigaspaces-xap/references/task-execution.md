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
Double result = future.get();
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
import com.example.model.CategoryType;
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
        // Runs on ONE partition — query against local data only
        Payment template = new Payment();
        // If Payment is routed by merchantId, this counts payments for
        // merchants in this category that happen to be on this partition
        return gigaSpace.count(template); // scoped to this partition
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
Long count = future.get();
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
Long totalProcessed = future.get();

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

```java
// Blocking get
Double result = future.get();

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

// Timeout
Double result = future.get(5, TimeUnit.SECONDS);
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
