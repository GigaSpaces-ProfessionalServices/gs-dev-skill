# XAP Event Containers (XAP 17.3.0)

| Container | Semantics | Use when |
|-----------|-----------|----------|
| **Polling Container** | Point-to-point (queue): **takes** entry from space; exactly one listener fires per event | Pipeline stages, work queues, stream processing |
| **Notify Container** | Publish/subscribe: notifies all listeners; entry **stays** in space | Fan-out, audit, cache invalidation, monitoring |

---

## Polling Container (Annotation Style)

```java
package com.example.processor;

import org.openspaces.events.*;
import org.openspaces.events.polling.*;
import org.openspaces.events.adapter.SpaceDataEvent;
import org.openspaces.core.GigaSpace;
import org.springframework.transaction.annotation.Transactional;
import com.example.model.Trade;
import com.j_spaces.core.client.SQLQuery;
import jakarta.annotation.Resource;

@EventDriven
@Polling(
    gigaSpace      = "gigaSpace",     // bean name of the GigaSpace to poll
    concurrentConsumers    = 2,        // min concurrent threads
    maxConcurrentConsumers = 4         // max concurrent threads
)
@TransactionalEvent  // take + event handler run in one TX; auto-rollback on exception
public class TradeProcessor {

    @Resource
    private GigaSpace gigaSpace;

    // Template defining which entries trigger this container
    @EventTemplate
    public SQLQuery<Trade> template() {
        SQLQuery<Trade> q = new SQLQuery<>(Trade.class, "status = ?");
        q.setParameter(1, TradeStatus.PENDING);
        return q;
    }

    // Called for each taken entry; return value is written back to space
    @SpaceDataEvent
    public Trade process(Trade trade) {
        // process the trade...
        trade.setStatus(TradeStatus.PROCESSED);
        return trade; // written to space; null = not written back
    }
}
```

### Polling Container: Single-Field Filter

```java
@EventTemplate
public SQLQuery<Trade> matchTemplate() {
    SQLQuery<Trade> q = new SQLQuery<>(Trade.class, "status = ?");
    q.setParameter(1, TradeStatus.PENDING);
    return q;
}
```

### Polling Container: Multiple Entries per Trigger (batch)

```java
@EventDriven
@Polling(receiveOperationHandler = "batchReceiver")
public class BatchProcessor {

    @Bean
    public MultipleObjectsMessageReceiveOperationHandler batchReceiver() {
        MultipleObjectsMessageReceiveOperationHandler handler =
                new MultipleObjectsMessageReceiveOperationHandler();
        handler.setMaxEntries(50); // take up to 50 per polling cycle
        return handler;
    }

    @EventTemplate
    public Trade template() { return new Trade(); }

    // Receives Trade[] when batchReceiver is configured
    @SpaceDataEvent
    public void process(Trade[] batch) {
        for (Trade t : batch) {
            // process each
        }
    }
}
```

---

## Notify Container (Annotation Style)

```java
package com.example.audit;

import org.openspaces.events.*;
import org.openspaces.events.notify.*;
import org.openspaces.events.adapter.SpaceDataEvent;
import org.openspaces.core.GigaSpace;
import com.example.model.Payment;
import com.j_spaces.core.client.SQLQuery;
import jakarta.annotation.Resource;

// Notification types are controlled by a SEPARATE @NotifyType annotation.
// @Notify does NOT have notifyWrite/notifyUpdate/notifyTake attributes.
@EventDriven
@Notify(gigaSpace = "gigaSpace")
@NotifyType(write = true, update = true)   // write=new entries, update=field changes
// Other @NotifyType flags: take, leaseExpire, unmatched, matchedUpdate, rematchedUpdate
public class PaymentAuditListener {

    @Resource
    private GigaSpace gigaSpace;

    @EventTemplate
    public SQLQuery<Payment> template() {
        SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "status = ?");
        q.setParameter(1, TransactionStatus.PROCESSED); // notify only for PROCESSED payments
        return q;
    }

    @SpaceDataEvent
    public void onPayment(Payment payment) {
        // Entry is NOT removed from space (unlike polling container)
        AuditRecord record = new AuditRecord(payment.getPaymentId(), new Date());
        gigaSpace.write(record);
    }
}
```

---

## FIFO Ordering

```java
// Enable FIFO at the class level
@SpaceClass(fifoSupport = FifoSupport.OPERATION)
public class OrderedEvent implements Serializable { ... }

// Enable FIFO on the polling container
@EventDriven
@Polling(fifo = true)
@TransactionalEvent
public class FifoProcessor {
    @EventTemplate
    public OrderedEvent template() { return new OrderedEvent(); }

    @SpaceDataEvent
    public void process(OrderedEvent event) { ... }
}
```

---

## pu.xml — Polling Container (use when annotation style is insufficient)

Prefer `<os-core:sql-query>` over a POJO-bean `<os-core:template>` — same reasoning as
`space-operations.md`'s SQLQuery-over-template guidance: a bean template can only express
equality-per-property and can't be null-safe on primitives, while `sql-query` supports the full
WHERE-clause grammar (ranges, `IN`, `IS NULL`, etc.) and takes an explicit projections attribute.
Namespace declarations (`xmlns:os-events`, `xmlns:os-core`) go on the root `<beans>` element — see
`processing-unit.md` for a full pu.xml header.

```xml
<os-events:polling-container id="tradeProcessor" giga-space="gigaSpace">
    <os-events:tx-support tx-manager="transactionManager"/>
    <os-core:sql-query class="com.example.model.Trade" where="status = 'PENDING'"/>
    <os-events:listener>
        <os-events:annotation-adapter>
            <os-events:delegate>
                <bean class="com.example.processor.TradeEventHandler"/>
            </os-events:delegate>
        </os-events:annotation-adapter>
    </os-events:listener>
</os-events:polling-container>
```

`<os-core:sql-query>` is a direct child of `<os-events:polling-container>` (and
`<os-events:notify-container>`) as an alternative to `<os-core:template>` — attributes are `where`
(required), `class` or `class-name`, and optional `projections` (comma-separated property names).
There's no `?` parameter binding in static XML config — embed the literal value directly in `where`,
as shown above.

---

## BillBuddy Polling Pipeline

The training app uses two chained polling containers:

```
PaymentFeeder
    → writes Payment(status=NEW)
        → AuditPaymentPollingContainer    takes NEW → audits → writes back AUDITED
            → ProcessingFeePollingContainer  takes AUDITED → calculates fee → writes PROCESSED
```

Key pattern: **each stage filters by `status`**, so containers don't compete.

```java
// Stage 1: AuditPaymentPollingEventContainer
@EventTemplate
SQLQuery<Payment> auditTemplate() {
    SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "status = ?");
    q.setParameter(1, TransactionStatus.NEW);
    return q;
}

@SpaceDataEvent
public Payment audit(Payment payment) {
    payment.setStatus(TransactionStatus.AUDITED);
    return payment; // written back; picked up by stage 2
}

// Stage 2: ProcessingFeePollingEventContainer
@EventTemplate
SQLQuery<Payment> processingTemplate() {
    SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "status = ?");
    q.setParameter(1, TransactionStatus.AUDITED);
    return q;
}

@SpaceDataEvent
public Payment processAndChargeFee(Payment payment) {
    // read merchant, compute fee, write ProcessingFee, update merchant balance
    payment.setStatus(TransactionStatus.PROCESSED);
    return payment;
}
```

---

## Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| No `@TransactionalEvent` on pipeline processor | Add `@TransactionalEvent` — ensures take + write are atomic |
| `gigaSpace` bean missing `tx-manager` when using `@TransactionalEvent` | XAP requires the GigaSpace bean itself to reference the transaction manager: `<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager"/>` — the event container will fail at startup with `GigaSpace is not transactional` if omitted |
| Template with all-null fields | Always filter by at least one field (e.g., `status`) to avoid consuming ALL entries |
| Both polling + notify containers competing for same entries | Polling takes entries (destructive); notify observes (non-destructive) — they serve different purposes |
| Throwing unchecked exception without `@TransactionalEvent` | Entry is consumed but not processed; add `@TransactionalEvent` for auto-rollback |
| `concurrentConsumers > numberOfPartitions` | Excess threads sit idle waiting; size to partition count |
| `@EventDriven`/`@Polling`/`@Notify` bean never fires, no error at all | `pu.xml` is missing `<os-events:annotation-support/>` — `<context:component-scan>` alone discovers the class but doesn't turn it into a real event container; see `processing-unit.md`'s pu.xml skeleton |
| Standalone Notify/Polling Spring Boot app (no web server) exits right after "Started..." log line, container never processes anything | Nothing keeps the JVM alive once `main()` returns — block it, e.g. `Thread.currentThread().join()` after `SpringApplication.run(...)` |
