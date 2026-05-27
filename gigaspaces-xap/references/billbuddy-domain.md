# BillBuddy — Training Domain Model (XAP 17.2.1)

This is the canonical domain used throughout GigaSpaces training labs. Use these classes as reference when explaining XAP patterns.

The BillBuddy application models a payment processing platform: **Users** pay **Merchants** via **Payments**. A **Contract** defines the fee % for each merchant. A **ProcessingFee** is auto-generated when a payment is audited.

---

## Domain Classes

### Payment.java
```java
package com.gs.billbuddy.model;

import com.gigaspaces.annotation.pojo.*;
import com.gigaspaces.metadata.index.SpaceIndexType;
import java.io.Serializable;
import java.util.Date;

@SpaceClass
public class Payment implements Serializable {

    private String paymentId;
    private Integer payingAccountId;       // @SpaceIndex — who paid
    private Integer receivingMerchantId;   // @SpaceRouting — partitioned by merchant
    private String description;
    private Double paymentAmount;
    private TransactionStatus status;
    private Date createdDate;

    // XAP requires default constructor
    public Payment() {}

    @SpaceId(autoGenerate = true)          // UUID auto-generated
    public String getPaymentId() { return paymentId; }
    public void setPaymentId(String paymentId) { this.paymentId = paymentId; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public Integer getPayingAccountId() { return payingAccountId; }
    public void setPayingAccountId(Integer id) { this.payingAccountId = id; }

    @SpaceRouting                          // route by merchant so all their payments colocate
    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public Integer getReceivingMerchantId() { return receivingMerchantId; }
    public void setReceivingMerchantId(Integer id) { this.receivingMerchantId = id; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public TransactionStatus getStatus() { return status; }
    public void setStatus(TransactionStatus status) { this.status = status; }

    @SpaceIndex(type = SpaceIndexType.ORDERED)
    public Date getCreatedDate() { return createdDate; }
    public void setCreatedDate(Date createdDate) { this.createdDate = createdDate; }

    public Double getPaymentAmount() { return paymentAmount; }
    public void setPaymentAmount(Double paymentAmount) { this.paymentAmount = paymentAmount; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

### User.java
```java
@SpaceClass
public class User implements Serializable {
    private Integer userAccountId;
    private String name;
    private Address address;
    private Double balance;
    private Double creditLimit;
    private AccountStatus status;

    public User() {}

    @SpaceId(autoGenerate = false)  // application assigns the ID
    @SpaceRouting                   // route = ID; all user data on one partition
    public Integer getUserAccountId() { return userAccountId; }
    public void setUserAccountId(Integer id) { this.userAccountId = id; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    @SpaceIndex(type = SpaceIndexType.ORDERED)
    public Double getCreditLimit() { return creditLimit; }
    public void setCreditLimit(Double creditLimit) { this.creditLimit = creditLimit; }

    // balance, address, status — standard getters/setters
}
```

### Merchant.java
```java
@SpaceClass
public class Merchant implements Serializable {
    private Integer merchantAccountId;
    private String name;
    private Address address;
    private CategoryType category;
    private Double feeAmount;
    private AccountStatus status;

    public Merchant() {}

    @SpaceId(autoGenerate = false)
    @SpaceRouting
    public Integer getMerchantAccountId() { return merchantAccountId; }
    public void setMerchantAccountId(Integer id) { this.merchantAccountId = id; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public CategoryType getCategory() { return category; }
    public void setCategory(CategoryType category) { this.category = category; }

    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Double getFeeAmount() { return feeAmount; }
    public void setFeeAmount(Double feeAmount) { this.feeAmount = feeAmount; }
}
```

### Contract.java
```java
@SpaceClass
public class Contract implements Serializable {
    private Integer merchantAccountId;  // same as Merchant ID → colocated on same partition
    private Double transactionPrecentFee;

    public Contract() {}

    @SpaceId(autoGenerate = false)
    @SpaceRouting
    public Integer getMerchantAccountId() { return merchantAccountId; }
    public void setMerchantAccountId(Integer id) { this.merchantAccountId = id; }

    public Double getTransactionPrecentFee() { return transactionPrecentFee; }
    public void setTransactionPrecentFee(Double fee) { this.transactionPrecentFee = fee; }
}
```

### ProcessingFee.java
```java
@SpaceClass
public class ProcessingFee implements Serializable {
    private String id;
    private String dependentPaymentId;
    private Integer payingAccountId;
    private Double amount;
    private String description;
    private Date createdDate;
    private TransactionStatus status;

    public ProcessingFee() {}

    @SpaceId(autoGenerate = true)
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    @SpaceRouting
    @SpaceIndex(type = SpaceIndexType.EQUAL)
    public Integer getPayingAccountId() { return payingAccountId; }
    public void setPayingAccountId(Integer id) { this.payingAccountId = id; }

    // Other getters/setters...
}
```

---

## Enums

```java
public enum TransactionStatus { NEW, AUDITED, PROCESSED, CANCELLED, EXPIRED }
public enum AccountStatus     { ACTIVE, SUSPENDED, CLOSED }
public enum CategoryType      { FOOD, TRAVEL, ENTERTAINMENT, RETAIL, UTILITIES, HEALTHCARE }
```

---

## Processing Pipeline

```
PaymentFeeder (client)
  → gigaSpace.write(new Payment(status=NEW))
    ↓
AuditPaymentPollingEventContainer (PU, colocated)
  - EventTemplate: SQLQuery: status = NEW
  - Takes payment, audits it
  - Returns payment with status = AUDITED
    ↓
ProcessingFeePollingEventContainer (PU, colocated)
  - EventTemplate: SQLQuery: status = AUDITED
  - Reads Merchant + Contract for this payment
  - Calculates fee = contract.transactionPrecentFee * payment.amount
  - Writes ProcessingFee entry
  - Updates Merchant.feeAmount via gigaSpace.write(merchant)
  - Returns payment with status = PROCESSED
```

### ProcessingFeePollingEventContainer (Full Example)

```java
@EventDriven
@Polling(gigaSpace = "gigaSpace")
@TransactionalEvent
public class ProcessingFeePollingEventContainer {

    @Resource
    private GigaSpace gigaSpace;

    @EventTemplate
    SQLQuery<Payment> template() {
        SQLQuery<Payment> q = new SQLQuery<>(Payment.class, "status = ?");
        q.setParameter(1, TransactionStatus.AUDITED);
        return q;
    }

    @SpaceDataEvent
    public Payment processPayments(Payment payment) {
        // Read Merchant (colocated — same partition as payment due to @SpaceRouting)
        Merchant merchantTemplate = new Merchant();
        merchantTemplate.setMerchantAccountId(payment.getReceivingMerchantId());
        Merchant merchant = gigaSpace.read(merchantTemplate);

        // Read Contract with projection (only need the fee %)
        SQLQuery<Contract> contractQuery =
                new SQLQuery<>(Contract.class, "merchantAccountId = ?")
                        .setProjections("transactionPrecentFee");
        contractQuery.setParameter(1, payment.getReceivingMerchantId());
        Contract contract = gigaSpace.read(contractQuery);

        Double feeAmount = contract.getTransactionPrecentFee() * payment.getPaymentAmount();

        // Update merchant balance
        merchant.setFeeAmount(merchant.getFeeAmount() + feeAmount);
        gigaSpace.write(merchant);

        // Write ProcessingFee
        ProcessingFee fee = new ProcessingFee();
        fee.setDependentPaymentId(payment.getPaymentId());
        fee.setPayingAccountId(merchant.getMerchantAccountId());
        fee.setAmount(feeAmount);
        fee.setCreatedDate(new Date());
        fee.setStatus(TransactionStatus.PROCESSED);
        gigaSpace.write(fee);

        payment.setStatus(TransactionStatus.PROCESSED);
        return payment; // written back to space
    }
}
```

---

## CountPaymentsByCategoryTask (DistributedTask Example)

```java
public class CountPaymentsByCategoryTask
        implements DistributedTask<Integer, Long>, Serializable {

    private final CategoryType categoryType;

    public CountPaymentsByCategoryTask(CategoryType categoryType) {
        this.categoryType = categoryType;
    }

    @TaskGigaSpace
    private transient GigaSpace gigaSpace;

    @Override
    public Integer execute() throws Exception {
        // Runs on ONE partition
        Merchant merchantTemplate = new Merchant();
        merchantTemplate.setCategory(categoryType);
        Merchant[] merchants = gigaSpace.readMultiple(merchantTemplate, Integer.MAX_VALUE);

        int count = 0;
        for (Merchant m : merchants) {
            Payment paymentTemplate = new Payment();
            paymentTemplate.setReceivingMerchantId(m.getMerchantAccountId());
            count += gigaSpace.count(paymentTemplate);
        }
        return count;
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

// Client usage:
AsyncFuture<Long> future = gigaSpace.execute(new CountPaymentsByCategoryTask(CategoryType.FOOD));
Long count = future.get();
```

---

## Recommended pom.xml for BillBuddy-style Multi-Module Project

```xml
<properties>
    <gigaspaces.version>17.2.1-ga</gigaspaces.version>
    <spring.version>6.2.1</spring.version>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>

<repositories>
    <repository>
        <id>org.openspaces</id>
        <url>https://maven-repository.openspaces.org</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>org.gigaspaces</groupId>
        <artifactId>xap-openspaces</artifactId>
        <version>${gigaspaces.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-orm</artifactId>
        <version>${spring.version}</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
        <version>2.1.1</version>
    </dependency>
</dependencies>
```
