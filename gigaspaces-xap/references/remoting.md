# XAP Space-Based Remoting (XAP 17.3.0)

Space-Based Remoting exposes server-side services over the space transport layer, providing automatic load-balancing, high availability, and partition-aware routing.

---

## Executor-Based Remoting (Recommended)

The client invocation sends a Task to the server-side service via the space. The service must be **colocated with the space** in a Processing Unit.

### Step 1 — Define the Service Interface

```java
package com.example.service;

import java.util.List;

public interface ITradeService {
    Trade processBreak(String tradeId);
    List<Trade> getOpenTrades(String accountId);
    Map<String, Long> countByStatus();
}
```

### Step 2 — Implement on the Server (Processing Unit)

```java
package com.example.service;

import org.openspaces.remoting.RemotingService;
import org.openspaces.core.GigaSpace;
import jakarta.annotation.Resource;
import com.j_spaces.core.client.SQLQuery;

@RemotingService   // registers this bean as a remoting endpoint in the space
public class TradeServiceImpl implements ITradeService {

    @Resource
    private GigaSpace gigaSpace;

    @Override
    public Trade processBreak(String tradeId) {
        SQLQuery<Trade> q = new SQLQuery<>(Trade.class, "tradeId = ?");
        q.setParameter(1, tradeId);
        Trade trade = gigaSpace.read(q);
        if (trade == null) return null;
        trade.setStatus(TradeStatus.BREAK_RESOLVED);
        gigaSpace.write(trade);
        return trade;
    }

    @Override
    public List<Trade> getOpenTrades(String accountId) {
        SQLQuery<Trade> q = new SQLQuery<>(Trade.class, "accountId = ? AND status = ?");
        q.setParameter(1, accountId);
        q.setParameter(2, TradeStatus.OPEN);
        return Arrays.asList(gigaSpace.readMultiple(q, 1000));
    }

    @Override
    public Map<String, Long> countByStatus() {
        // Aggregate across this partition
        // ...
        return statusMap;
    }
}
```

### Step 3 — Register in pu.xml (server side)

```xml
<!-- Component scan picks up @RemotingService -->
<context:component-scan base-package="com.example.service"/>

<!-- Enable annotation-based remoting -->
<os-remoting:annotation-support/>

<!-- REQUIRED: executor-based service exporter — without this, startup fails with
     "No service exporters are defined within the context" even if annotation-support is present -->
<os-remoting:service-exporter id="serviceExporter"/>
```

### Step 4 — Inject on the Client (@ExecutorProxy)

```java
package com.example.client;

import org.openspaces.remoting.ExecutorProxy;
import org.openspaces.core.GigaSpace;
import jakarta.annotation.Resource;
import org.springframework.stereotype.Component;

@Component
public class TradeReportService {

    @Resource
    private GigaSpace gigaSpace;

    // Routes to a single partition (non-broadcast)
    @ExecutorProxy(gigaSpace = "gigaSpace")
    private ITradeService tradeService;

    // Routes to ALL partitions; results reduced by the reducer class
    @ExecutorProxy(
        gigaSpace = "gigaSpace",
        broadcast = true,
        remoteResultReducerType = CountByStatusReducer.class
    )
    private ITradeService broadcastTradeService;

    public Trade resolveBreak(String tradeId) {
        return tradeService.processBreak(tradeId); // routed to correct partition
    }

    public Map<String, Long> clusterWideCount() {
        return broadcastTradeService.countByStatus(); // sent to all partitions, reduced
    }
}
```

---

## Broadcast Reducer

When using `broadcast = true`, implement a reducer to merge per-partition results:

```java
package com.example.client;

import org.openspaces.remoting.RemoteResultReducer;
import java.util.List;
import java.util.Map;
import java.util.HashMap;

public class CountByStatusReducer implements RemoteResultReducer<Map<String,Long>, Map<String,Long>> {

    @Override
    public Map<String, Long> reduce(List<SpaceRemotingResult<Map<String,Long>>> results, ...)
            throws Exception {
        Map<String, Long> merged = new HashMap<>();
        for (SpaceRemotingResult<Map<String,Long>> r : results) {
            if (r.getException() != null) throw r.getException();
            r.getResult().forEach((status, count) ->
                    merged.merge(status, count, Long::sum));
        }
        return merged;
    }
}
```

---

## BillBuddy Remoting Examples

### Pattern 1: Distributed Executor (DistributedTask as service)

```java
// Client: run a DistributedTask across all partitions
AsyncFuture<Long> future = gigaSpace.execute(new CountPaymentsByCategoryTask(CategoryType.FOOD));
Long total = future.get(5, TimeUnit.SECONDS); // always bound the wait
```

### Pattern 2: @RemotingService + @ExecutorProxy with Broadcast

```java
// Service interface
public interface ICountPaymentsByCategoryService {
    int findPaymentCountByCategory(CategoryType categoryType);
}

// Server implementation (PU colocated with space)
@RemotingService
public class CountPaymentByCategoryService implements ICountPaymentsByCategoryService {

    @Resource
    private GigaSpace gigaSpace;

    @Override
    public int findPaymentCountByCategory(CategoryType categoryType) {
        SQLQuery<Merchant> merchantQuery = new SQLQuery<>(Merchant.class, "category = ?");
        merchantQuery.setParameter(1, categoryType);
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
        return count; // result from THIS partition only
    }
}

// Client: inject with reducer to sum all partition results
@ExecutorProxy(
    gigaSpace = "gigaSpace",
    broadcast = true,
    remoteResultReducerType = CountPaymentByCategoryReducer.class
)
private ICountPaymentsByCategoryService categoryService;

// Usage
int total = categoryService.findPaymentCountByCategory(CategoryType.FOOD);
```

### Pattern 3: Top-N Reducer

```java
public class CategoryTop5PaymentReducer
        implements RemoteResultReducer<Payment[], Payment[]> {

    @Override
    public Payment[] reduce(List<SpaceRemotingResult<Payment[]>> results, ...) {
        List<Payment> all = new ArrayList<>();
        for (SpaceRemotingResult<Payment[]> r : results) {
            if (r.getResult() != null) all.addAll(Arrays.asList(r.getResult()));
        }
        // Sort and return top 5
        return all.stream()
                .sorted(Comparator.comparingDouble(Payment::getPaymentAmount).reversed())
                .limit(5)
                .toArray(Payment[]::new);
    }
}
```

---

## pu.xml for Client-Side Remoting

```xml
<!-- Client PU or Spring Boot app pu.xml -->
<os-core:space-proxy id="space" name="mySpace"/>
<os-core:giga-space id="gigaSpace" space="space"/>

<!-- Enable @ExecutorProxy injection -->
<os-remoting:annotation-support/>
```

---

## Routing vs Broadcast

| | Default (`broadcast=false`) | Broadcast (`broadcast=true`) |
|---|---|---|
| Routes to | Single partition (by method arg hash or explicit routing) | All partitions |
| Result | Single service response | Reduced via `remoteResultReducerType` |
| Use case | Account-specific queries | Cluster-wide aggregation |

---

## Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| `<os-remoting:annotation-support/>` alone without `<os-remoting:service-exporter>` | Add `<os-remoting:service-exporter id="serviceExporter"/>` to pu.xml — annotation-support detects the beans but a service exporter is still required to register them; omitting it causes "No service exporters are defined within the context" at startup |
| Broadcast without a reducer | All partition results arrive as a list; specify `remoteResultReducerType` |
| `@RemotingService` on a non-singleton Spring bean | Remoting services must be Spring singletons |
| Hold `GigaSpace` proxy (not @Resource injected) in remoting service | Always `@Resource`-inject the colocated space; never connect from server back to remote |
| Use remoting for high-frequency per-entry operations | Use event containers or direct space API for throughput-critical paths; remoting adds task overhead |
