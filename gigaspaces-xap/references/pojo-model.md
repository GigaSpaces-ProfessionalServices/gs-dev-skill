# XAP POJO Model & Annotations (XAP 17.2.1)

## Class-Level Annotations

```java
@SpaceClass                           // marks this as a space-stored type
@SpaceClass(persist = true)           // enable persistence (write-behind / mirror)
@SpaceClass(fifoSupport = FifoSupport.OPERATION)  // FIFO per-operation ordering
@SpaceClass(fifoSupport = FifoSupport.ALL)         // FIFO all operations
@SpaceClass(includeProperties = IncludeProperties.EXPLICIT) // only annotated fields stored
@SpaceClass(replicate = true)         // replicate in partial-replication mode
@SpaceClass(inheritIndexes = false)   // do not inherit superclass indexes
```

Compound indexes (multiple fields together):
```java
@CompoundSpaceIndexes({
    @CompoundSpaceIndex(paths = {"firstName", "lastName"}),
    @CompoundSpaceIndex(paths = {"department", "salary"})
})
@SpaceClass
public class Employee { ... }
```

---

## Property-Level Annotations (on getters, not fields)

### Identity & Routing

```java
@SpaceId(autoGenerate = false)  // application-assigned ID (Integer, String, etc.)
public Integer getId() { return id; }

@SpaceId(autoGenerate = true)   // XAP auto-generates a UUID string
public String getId() { return id; }

// @SpaceRouting: which field routes entries to partitions.
// Default = @SpaceId field. Override when routing key != ID.
@SpaceRouting
public Integer getMerchantId() { return merchantId; }

// Common pattern: SpaceId is auto-generated UUID; routing is a business key
@SpaceId(autoGenerate = true)
public String getPaymentId() { return paymentId; }

@SpaceRouting          // route by merchant so all payments for a merchant land together
@SpaceIndex(type = SpaceIndexType.EQUAL)
public Integer getReceivingMerchantId() { return receivingMerchantId; }
```

### Indexing

```java
@SpaceIndex(type = SpaceIndexType.EQUAL)          // equality lookups (default)
@SpaceIndex(type = SpaceIndexType.ORDERED)        // range queries (>, <, BETWEEN)
@SpaceIndex(type = SpaceIndexType.EQUAL_AND_ORDERED) // both — highest index cost
```

### Storage & Exclusion

```java
@SpaceExclude          // exclude from space storage (transient field)
@SpaceStorageType(StorageType.COMPRESSED)   // compress large blobs
@SpaceStorageType(StorageType.BINARY)       // store as raw bytes
@SpaceDynamicProperties                     // Map<String,Object> for schema-free properties
@SpaceFifoGroupingProperty(path = "groupId") // group FIFO ordering per group key
@SpaceLeaseExpiration  // field holds the entry's lease expiration (ms since epoch)
```

### Version (Optimistic Locking)

```java
@SpaceVersion
public int getVersion() { return version; }
```

---

## Minimal Valid POJO Pattern

Every space class requires:
1. `@SpaceClass`
2. No-arg default constructor  
3. Exactly one `@SpaceId` getter
4. `@SpaceRouting` (explicit; defaults to `@SpaceId` field if omitted)
5. Implements `Serializable`

```java
package com.example.model;

import com.gigaspaces.annotation.pojo.*;
import com.gigaspaces.metadata.index.SpaceIndexType;
import java.io.Serializable;
import java.util.Date;

@SpaceClass
public class Trade implements Serializable {

    private String tradeId;         // @SpaceId (auto-generated UUID)
    private String accountId;       // @SpaceRouting (partition by account)
    private String symbol;
    private Double price;
    private Integer quantity;
    private TradeStatus status;
    private Date tradeDate;

    // XAP requires default constructor
    public Trade() {}

    public Trade(String accountId, String symbol) {
        this.accountId = accountId;
        this.symbol = symbol;
    }

    @SpaceId(autoGenerate = true)
    public String getTradeId() { return tradeId; }
    public void setTradeId(String tradeId) { this.tradeId = tradeId; }

    // Route by account — all trades for an account land on the same partition
    @SpaceRouting
    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public String getAccountId() { return accountId; }
    public void setAccountId(String accountId) { this.accountId = accountId; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public String getSymbol() { return symbol; }
    public void setSymbol(String symbol) { this.symbol = symbol; }

    @SpaceIndex(type = SpaceIndexType.ORDERED)
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }

    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public TradeStatus getStatus() { return status; }
    public void setStatus(TradeStatus status) { this.status = status; }

    @SpaceIndex(type = SpaceIndexType.ORDERED)
    public Date getTradeDate() { return tradeDate; }
    public void setTradeDate(Date tradeDate) { this.tradeDate = tradeDate; }

    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

---

## SpaceDocument (Schema-Free)

Use `SpaceDocument` when the schema is not known at compile time.

```java
import com.gigaspaces.document.SpaceDocument;

// Write
SpaceDocument doc = new SpaceDocument("com.example.DynamicEvent");
doc.setProperty("id",        UUID.randomUUID().toString());
doc.setProperty("source",    "feed-A");
doc.setProperty("timestamp", new Date());
doc.setProperty("payload",   Map.of("key", "value"));
gigaSpace.write(doc);

// Query
SQLQuery<SpaceDocument> q = new SQLQuery<>("com.example.DynamicEvent", "source = ?");
q.setParameter(1, "feed-A");
SpaceDocument[] results = gigaSpace.readMultiple(q, 100);
for (SpaceDocument d : results) {
    String id = d.getProperty("id");
}
```

---

## @SupportCodeChange (Hot Redeploy)

Used on Tasks and aggregators colocated with the space to force class reload without restarting GSCs:

```java
import com.gigaspaces.annotation.SupportCodeChange;

@SupportCodeChange(id = "1")  // increment id on each deployment to invalidate cache
public class MyTask implements DistributedTask<Integer, Long>, Serializable {
    // ...
}
```

---

## BillBuddy Domain: Partitioning Choices

| Class | @SpaceId | @SpaceRouting | Rationale |
|-------|----------|---------------|-----------|
| `User` | `userAccountId` (no autoGenerate) | same as ID | Account-centric queries colocated |
| `Merchant` | `merchantAccountId` | same as ID | All merchant data on one partition |
| `Payment` | `paymentId` (autoGenerate=true) | `receivingMerchantId` | All payments for a merchant colocated with merchant |
| `Contract` | `merchantAccountId` | same as ID | Contract joins with Merchant on same partition |
| `ProcessingFee` | auto | `payingAccountId` | Colocated with Merchant |

---

## Rules Summary

1. Annotate **getters**, not fields — XAP reads metadata from getter annotations.
2. Never annotate both a getter AND its backing field for the same property.
3. `@SpaceId` is mandatory — without it XAP cannot manage object identity.
4. `@SpaceRouting` defaults to `@SpaceId` if not set — always be explicit for clarity.
5. Every `@SpaceIndex` adds write overhead — index only query-filter fields.
6. `Serializable` required on every space class — even if not used in tasks.
