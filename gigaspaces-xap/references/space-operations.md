# XAP Space Operations (XAP 17.2.1)

## Connecting to the Space

### Embedded Space (in-process, unit tests or colocated PU)
```java
GigaSpace gigaSpace = new GigaSpaceConfigurer(
        new EmbeddedSpaceConfigurer("mySpace")).gigaSpace();
```

### Remote Proxy (client apps, feeders, REST services)
```java
GigaSpace gigaSpace = new GigaSpaceConfigurer(
        new SpaceProxyConfigurer("mySpace")
                .lookupLocators("localhost:4174")
                .lookupGroups("gigaspaces-XAP")
).gigaSpace();
```

### Spring Bean (Processing Unit pu.xml)
```xml
<os-core:embedded-space id="space" space-name="mySpace"/>
<os-core:giga-space id="gigaSpace" space="space"/>
<!-- or for remote: -->
<os-core:space-proxy id="space" name="mySpace"/>
```

### Spring @Bean (Spring Boot client)
```java
@Bean
public GigaSpace gigaSpace() throws Exception {
    PlatformTransactionManager ptm = new DistributedJiniTxManagerConfigurer().transactionManager();
    return new GigaSpaceConfigurer(
            new SpaceProxyConfigurer("mySpace").lookupLocators("localhost:4174")
    ).transactionManager(ptm).gigaSpace();
}
```

---

## Write Operations

```java
// Write single entry — update if exists (by @SpaceId), insert if new
gigaSpace.write(object);

// Write with lease (TTL in ms) — auto-expiry
gigaSpace.write(object, 60_000); // expires in 60 seconds

// Write-once — fails if entry already exists
gigaSpace.write(object, LeaseContext.INFINITE, 0, WriteModifiers.WRITE_ONLY);

// Update-only — fails if entry doesn't exist
gigaSpace.write(object, LeaseContext.INFINITE, 0, WriteModifiers.UPDATE_ONLY);

// Batch write — always prefer over single writes for throughput
Object[] batch = new Object[]{trade1, trade2, trade3};
gigaSpace.writeMultiple(batch);
```

---

## Read Operations

### Template-Based (null = wildcard)
```java
Trade template = new Trade();
template.setStatus(TradeStatus.OPEN); // only non-null fields are matched

Trade result = gigaSpace.read(template);                   // first match
Trade[] results = gigaSpace.readMultiple(template, 1000);  // up to 1000 matches
```

### SQLQuery (prefer for complex predicates)
```java
SQLQuery<Trade> query = new SQLQuery<>(Trade.class, "symbol = ? AND price > ?");
query.setParameter(1, "AAPL");
query.setParameter(2, 150.0);

Trade  single = gigaSpace.read(query);
Trade[] batch  = gigaSpace.readMultiple(query, 500);
```

### Read by ID (fastest — direct lookup)
```java
Trade trade = gigaSpace.readById(Trade.class, tradeId);

// With routing key (avoids broadcast when ID != routing field)
Trade trade = gigaSpace.readById(Trade.class, tradeId, routingValue);
```

### Read with Timeout (blocking read — waits for entry to appear)
```java
// Block up to 5 seconds for a matching entry
Trade trade = gigaSpace.read(template, 5_000);
```

### Projection (fetch only specific fields)
```java
SQLQuery<Contract> q = new SQLQuery<>(Contract.class, "merchantId = ?")
        .setProjections("transactionPrecentFee");
q.setParameter(1, merchantId);
Contract c = gigaSpace.read(q);
// Only transactionPrecentFee is populated
```

---

## Take Operations

Take **removes** the entry from the space atomically.

```java
Trade taken = gigaSpace.take(template);                   // remove first match
Trade taken = gigaSpace.takeById(Trade.class, tradeId);   // remove by ID
Trade[] taken = gigaSpace.takeMultiple(template, 100);    // remove up to 100

// Non-blocking take
Trade taken = gigaSpace.take(template, 0);  // timeout = 0 = no wait
```

---

## Change API (Partial Update)

Use `change()` instead of read-then-write to minimize data transfer.

```java
import com.gigaspaces.client.ChangeSet;
import com.gigaspaces.client.ChangeResult;
import com.gigaspaces.query.IdQuery;   // note: com.gigaspaces.QUERY, not com.gigaspaces.client

// --- Change by @SpaceId (most efficient — use when you have the entry's ID) ---
// IdQuery routes directly to the owning partition using the routing value (3rd arg).
// Always supply the routing value when @SpaceRouting != @SpaceId to avoid a broadcast.
IdQuery<Order> idQuery = new IdQuery<>(Order.class, order.getOrderId(), order.getCustomerId());
ChangeResult<Order> result = gigaSpace.change(idQuery,
        new ChangeSet().set("status", OrderStatus.PAID).set("updatedAt", new Date()));
// result.getNumberOfChangedEntries() == 0 means the entry was not found (already removed/changed)

// --- Change by SQLQuery (use when you don't have the ID or need a conditional filter) ---
ChangeSet changeSet = new ChangeSet().increment("balance", 100.0);
gigaSpace.change(new SQLQuery<>(User.class, "userId = ?").setParameter(1, userId), changeSet);

// Set a field value
ChangeSet setStatus = new ChangeSet().set("status", TradeStatus.CLOSED);
gigaSpace.change(template, setStatus);

// Multiple changes in one operation
ChangeSet multi = new ChangeSet()
        .increment("balance", -payment.getAmount())
        .set("lastUpdated", new Date());

// With timeout (ms)
ChangeResult<User> r = gigaSpace.change(
        new SQLQuery<>(User.class, "userId = ?").setParameter(1, userId),
        multi,
        10_000L);
System.out.println("Changed " + r.getNumberOfChangedEntries() + " entries");
```

**Choosing the right query type for `change()`:**

| Situation | Use |
|---|---|
| You have the entry's `@SpaceId` value | `IdQuery` — direct lookup, no scan |
| You need a condition (`status = X`) across many entries | `SQLQuery` |
| You have the ID but `@SpaceRouting` ≠ `@SpaceId` | `IdQuery(type, id, routingValue)` — avoids broadcast |

---

## Count & Clear

```java
int count = gigaSpace.count(template);
int count = gigaSpace.count(new SQLQuery<>(Payment.class, "status = ?").setParameter(1, s));

// Clear all matching entries
gigaSpace.clear(template);
gigaSpace.clear(new SQLQuery<>(Payment.class, "status = 'EXPIRED'"));
```

---

## SpaceIterator (Large Result Sets)

Never use `readMultiple(query, Integer.MAX_VALUE)` for large datasets — use iterator.

```java
import com.gigaspaces.client.iterator.SpaceIterator;
import com.gigaspaces.client.iterator.SpaceIteratorConfiguration;

SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "status = ?");
query.setParameter(1, TransactionStatus.PENDING);

SpaceIteratorConfiguration config = new SpaceIteratorConfiguration().setBatchSize(200);

try (SpaceIterator<Payment> iterator = gigaSpace.iterator(query, config)) {
    while (iterator.hasNext()) {
        Payment payment = iterator.next();
        // process
    }
} // auto-closes and releases server-side resources
```

---

## Transactions

### Setup (Spring @Configuration or pu.xml)
```java
@Bean
public PlatformTransactionManager transactionManager() throws Exception {
    return new DistributedJiniTxManagerConfigurer().transactionManager();
}
```

### Usage
```java
import org.springframework.transaction.annotation.Transactional;

@Service
public class PaymentService {

    @Autowired
    private GigaSpace gigaSpace;

    // @Transactional: read-modify-write atomically
    // If exception thrown, take is rolled back (entry returned to space)
    @Transactional
    public void processPayment(String paymentId) {
        SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "paymentId = ?");
        q.setParameter(1, paymentId);

        Payment payment = gigaSpace.take(q); // removed from space
        if (payment == null) return;

        payment.setStatus(TransactionStatus.PROCESSED);
        gigaSpace.write(payment);            // written back with new status
        // Both take and write commit together — or both roll back on exception
    }
}
```

---

## Aggregation API (Server-Side)

```java
import com.gigaspaces.query.aggregators.AggregationSet;
import com.gigaspaces.query.aggregators.AggregationResult;

SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "status = ?");
q.setParameter(1, TransactionStatus.PROCESSED);

AggregationResult result = gigaSpace.aggregate(q, new AggregationSet()
        .sum("paymentAmount")
        .count()
        .max("paymentAmount")
        .groupBy("category", new AggregationSet().sum("paymentAmount").count())
);

Double total  = (Double) result.get("sum(paymentAmount)");
Long   count  = (Long)   result.get("count");
```

---

## Notification (Event-Driven Callback)

For one-off async callbacks (not polling containers):

```java
import com.j_spaces.core.client.NotifyModifiers;
import org.openspaces.core.GigaSpace;

gigaSpace.addListener(template, (source, remoteEvent) -> {
    System.out.println("Entry written: " + source);
}, NotifyModifiers.NOTIFY_WRITE);
```

---

## Anti-Patterns

| ❌ Anti-pattern | ✅ Fix |
|---|---|
| `readMultiple(t, Integer.MAX_VALUE)` | Set a bounded limit; use `SpaceIterator` for large datasets |
| Read-then-write without `@Transactional` | Wrap with `@Transactional` — race condition without it |
| Template with all null fields | Use `SQLQuery` or set at least the routing field |
| `gigaSpace.write(obj)` in a tight loop | Use `gigaSpace.writeMultiple(batch)` |
| `readById(Class, id)` without routing value for non-ID routing | Pass routing value as 3rd arg to avoid broadcast |
