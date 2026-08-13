# XAP Processing Unit & Cluster Topologies

## Processing Unit (PU)

A Processing Unit is the unit of deployment in XAP. It is a Spring application context packaged as a JAR with an SLA descriptor. Prefer annotation-based configuration; use `pu.xml` only where annotations are insufficient (e.g., embedded space creation, SLA, WAN gateway).

```
my-pu/
├── META-INF/
│   └── spring/
│       ├── pu.xml        ← Spring context (minimal or omit if using pure annotations)
│       └── sla.xml       ← SLA: partitions, HA, backup count, max-per-machine
└── com/example/...       ← compiled classes (POJOs, event handlers, remoting services)
```

---

## Space Topology Decision

| Topology | When to use | How to configure |
|----------|-------------|-----------------|
| **Embedded space** | Colocated PU (data + logic on same JVM). Max throughput; no network. | `EmbeddedSpaceConfigurer` or `<os-core:embedded-space>` in pu.xml |
| **Remote space proxy** | Client applications, feeders, REST services, external processors | `SpaceProxyConfigurer` or `<os-core:space-proxy>` |
| **Local cache** | Read-heavy client; caches remote reads locally; eventually consistent | `LocalCacheSpaceConfigurer` wrapping the proxy |
| **Local view** | Client needs a read-only filtered subset of data locally | `LocalViewSpaceConfigurer` with SQLQuery predicates |

---

## pu.xml — Minimal Skeleton

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:os-core="http://www.openspaces.org/schema/core"
       xmlns:os-events="http://www.openspaces.org/schema/events"
       xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
               http://www.springframework.org/schema/context/spring-context.xsd
           http://www.openspaces.org/schema/core
               http://www.openspaces.org/schema/core/openspaces-core.xsd
           http://www.openspaces.org/schema/events
               http://www.openspaces.org/schema/events/openspaces-events.xsd
           http://www.openspaces.org/schema/remoting
               http://www.openspaces.org/schema/remoting/openspaces-remoting.xsd
           http://www.springframework.org/schema/tx
               http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- Embedded space (server-side PU) -->
    <os-core:embedded-space id="space" space-name="mySpace"/>

    <!-- OR: Remote proxy (client/feeder) -->
    <!-- <os-core:space-proxy id="space" name="mySpace" /> -->

    <!-- Distributed transaction manager (required for @Transactional and @TransactionalEvent) -->
    <os-core:distributed-tx-manager id="transactionManager"/>
    <tx:annotation-driven transaction-manager="transactionManager"/>

    <!-- IMPORTANT: tx-manager MUST be set on the GigaSpace bean itself when using
         @TransactionalEvent polling containers — omitting it causes startup failure:
         "GigaSpace is not transactional" -->
    <os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager"/>

    <!-- Scan for @EventDriven, @RemotingService, @Component beans -->
    <context:component-scan base-package="com.example"/>

    <!-- Enable @ExecutorProxy injection in annotated beans -->
    <os-remoting:annotation-support/>

</beans>
```

---

## sla.xml — SLA Descriptor

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:os-sla="http://www.openspaces.org/schema/sla"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.openspaces.org/schema/sla
               http://www.openspaces.org/schema/sla/openspaces-sla.xsd">

    <!-- 2 partitions, 1 backup each = 4 GSC instances -->
    <os-sla:sla cluster-schema="partitioned"
                number-of-instances="2"
                number-of-backups="1"
                max-instances-per-machine="1"/>
</beans>
```

---

## Spring Boot Main Class (Client / REST Application)

Use for feeder clients, REST gateways, or standalone test drivers that connect to a **remote** space.

```java
package com.example.rest;

import org.openspaces.core.GigaSpace;
import org.openspaces.core.GigaSpaceConfigurer;
import org.openspaces.core.space.SpaceProxyConfigurer;
import org.openspaces.core.transaction.manager.DistributedJiniTxManagerConfigurer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@SpringBootApplication(exclude = {
        SecurityAutoConfiguration.class,
        DataSourceAutoConfiguration.class
})
@Configuration
@EnableTransactionManagement
public class Main {

    @Value("${space.name}")
    private String spaceName;

    @Value("${space.locators}")  // e.g. "localhost:4174"
    private String locators;

    @Value("${space.group:gigaspaces-XAP}")
    private String group;

    @Bean
    public PlatformTransactionManager transactionManager() throws Exception {
        return new DistributedJiniTxManagerConfigurer().transactionManager();
    }

    @Bean
    public GigaSpace gigaSpace() throws Exception {
        return new GigaSpaceConfigurer(
                new SpaceProxyConfigurer(spaceName)
                        .lookupLocators(locators)
                        .lookupGroups(group)
        ).transactionManager(transactionManager()).gigaSpace();
    }

    public static void main(String[] args) {
        SpringApplication.run(Main.class, args);
    }
}
```

**`application.properties`:**
```properties
space.name=mySpace
space.locators=localhost:4174
space.group=gigaspaces-XAP
server.port=8080
```

---

## Spring Boot with @ImportResource (Hybrid: Boot + pu.xml)

When existing `pu.xml` configuration is needed alongside Spring Boot:

```java
@SpringBootApplication
@ImportResource("META-INF/spring/pu.xml")
@EnableTransactionManagement
public class Main {
    // GigaSpace bean comes from pu.xml
    public static void main(String[] args) {
        SpringApplication.run(Main.class, args);
    }
}
```

---

## Embedded Space in Unit Tests

```java
import org.openspaces.core.GigaSpace;
import org.openspaces.core.GigaSpaceConfigurer;
import org.openspaces.core.space.EmbeddedSpaceConfigurer;
import org.junit.Before;
import org.junit.After;

public class MyServiceTest {

    private GigaSpace gigaSpace;

    @Before
    public void setUp() {
        // Embedded space: no GigaSpaces server required
        gigaSpace = new GigaSpaceConfigurer(
                new EmbeddedSpaceConfigurer("testSpace")).gigaSpace();
    }

    @After
    public void tearDown() {
        // Destroy space after test
        ((EmbeddedSpaceConfigurer) gigaSpace.getSpace()).destroy();
    }
}
```

---

## Local Cache / Local View

```java
// Local Cache — caches ALL reads; invalidated on space updates
import org.openspaces.core.space.cache.LocalCacheSpaceConfigurer;

GigaSpace localCache = new GigaSpaceConfigurer(
        new LocalCacheSpaceConfigurer(remoteSpaceProxy)).gigaSpace();

// Local View — maintains a filtered read-only subset
import org.openspaces.core.space.cache.LocalViewSpaceConfigurer;
import com.j_spaces.core.client.SQLQuery;

LocalViewSpaceConfigurer localViewConfigurer = new LocalViewSpaceConfigurer(remoteSpaceProxy)
        .addViewQuery(new SQLQuery<>(Trade.class, "status = ?").setParameter(1, TradeStatus.OPEN))
        .addViewQuery(new SQLQuery<>(Account.class, ""));

GigaSpace localView = new GigaSpaceConfigurer(localViewConfigurer).gigaSpace();
```

---

## Writing Entries from a Colocated PU (Partitioned Space)

**Critical constraint:** The embedded `GigaSpace` proxy inside a PU is partition-local. It only accepts entries whose routing key hashes to the current partition. Writing an entry that belongs to a different partition throws a routing exception.

### Wrong approaches

```java
// ❌ WRONG: clustered proxy from inside the PU
// Semantically broken — the embedded space IS the local partition, not a gateway.
// A clustered proxy writes back into the cluster through the space proxy layer,
// which is unnecessary overhead and architecturally incorrect for colocated logic.
<os-core:giga-space id="clusteredGigaSpace" space="space" clustered="true"/>
```

```java
// ❌ WRONG: writing all entries through the embedded (local) proxy
// Entries whose routing key belongs to another partition throw a RoutingException.
gigaSpace.writeMultiple(allEntries);
```

### Correct approach — `ClusterInfoAware`

Implement `ClusterInfoAware` so XAP injects the partition topology before `@PostConstruct` fires. Use the partition index and count to filter the dataset so each instance writes only the entries that belong to it — directly into the local embedded space.

```java
import org.openspaces.core.cluster.ClusterInfo;
import org.openspaces.core.cluster.ClusterInfoAware;

@Component
public class DataSeeder implements ClusterInfoAware {

    private ClusterInfo clusterInfo;

    @Resource
    private GigaSpace gigaSpace;   // partition-local embedded proxy — correct here

    @Override
    public void setClusterInfo(ClusterInfo clusterInfo) {
        this.clusterInfo = clusterInfo;
    }

    @PostConstruct
    public void seed() {
        // clusterInfo is null in single-instance (non-partitioned) mode
        int numPartitions = (clusterInfo != null && clusterInfo.getNumberOfInstances() != null)
                ? clusterInfo.getNumberOfInstances() : 1;
        // instanceId is 1-based; convert to 0-based partition index
        int partitionIndex = (clusterInfo != null && clusterInfo.getInstanceId() != null)
                ? clusterInfo.getInstanceId() - 1 : 0;

        List<MyEntry> local = new ArrayList<>();
        for (MyEntry entry : allCandidates()) {
            // XAP routing formula — same hash XAP uses internally for @SpaceRouting fields
            if (Math.abs(entry.getRoutingKey().hashCode()) % numPartitions == partitionIndex) {
                local.add(entry);
            }
        }
        if (!local.isEmpty()) {
            gigaSpace.writeMultiple(local.toArray(new MyEntry[0]));
        }
        log.info("Seeded {}/{} entries on partition {}/{}",
                local.size(), totalCount, partitionIndex + 1, numPartitions);
    }
}
```

**Key points:**
- `clusterInfo.getNumberOfInstances()` = number of primary partitions (excludes backups)
- `clusterInfo.getInstanceId()` = 1-based; subtract 1 to get the 0-based partition index
- XAP's routing hash for any `Object` routing key: `Math.abs(routingKey.hashCode()) % numPartitions`
- When `clusterInfo` is null (single-instance dev mode), default `numPartitions=1` and `partitionIndex=0` so all entries are written — correct fallback behavior

---

## Cluster Schema Reference

| Schema | Description |
|--------|-------------|
| `partitioned` | Hash-partitioned; `@SpaceRouting` determines partition |
| `sync_replicated` | Synchronously replicated — all nodes hold identical data |
| `async_replicated` | Asynchronously replicated |
| (none) | Single instance (development / testing only) |
