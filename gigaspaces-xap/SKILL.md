---
name: gigaspaces-xap
description: >
  Expert GigaSpaces XAP 17.2.1 Java code generation and guidance. Use this skill for ANY task involving GigaSpaces XAP Java development — including writing Space POJOs, querying with SQLQuery or templates, event-driven processing with Polling/Notify containers, colocated task execution (Task/DistributedTask/DurableTask), space-based remoting, custom aggregators, transactions, Change API, Processing Unit design, Spring Boot integration, Maven project setup, and OpenTelemetry distributed tracing with Zipkin. Trigger whenever the user mentions GigaSpaces, XAP, IMDG, GigaSpace API, SpaceDocument, DistributedTask, DurableTask, Processing Unit, space-based architecture, SpaceRouting, partitioning, custom aggregator, SQLQuery, ZipkinTracerBean, OpenTelemetry, OTel span, distributed tracing, or any GigaSpaces-related code, pattern, or concept. Default XAP target version is 17.2.1 unless the user specifies otherwise.
---

# GigaSpaces XAP 17.2.1 – Java Code Generation Skill

GigaSpaces XAP (eXtreme Application Platform) is an in-memory data grid (IMDG) and Space-Based Architecture (SBA) platform for building high-throughput, low-latency distributed applications.

**Default target version: XAP 17.2.1** (Maven artifact version: `17.2.1`)

---

## Reference Files

**MANDATORY**: You MUST use the `Read` tool to read the relevant reference file(s) listed below **before generating any code or answering any question**. Do not skip this step. Select the file(s) based on the Quick Decision Guide below, then read them with absolute paths under `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/`.

| Absolute Path | Covers |
|------|--------|
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/maven-pom.md` | Maven pom.xml templates, repositories, Spring Boot, JDBC artifact, version strings |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/pojo-model.md` | @SpaceClass annotations, POJO design rules, SpaceDocument, @SupportCodeChange |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/space-operations.md` | GigaSpace API: write/read/take/change/count, SQLQuery, SpaceIterator, projections, transactions |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/event-containers.md` | Polling Container, Notify Container, FIFO, @TransactionalEvent |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/task-execution.md` | Task, DistributedTask, DurableTask, @TaskGigaSpace, AsyncFuture, routing vs broadcast |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/custom-aggregators.md` | AbstractPathAggregator, Externalizable, index optimization, skipFullScanSupported, @SupportCodeChange |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/remoting.md` | Executor-Based Remoting, @RemotingService, @ExecutorProxy, broadcast reducers |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/processing-unit.md` | PU packaging, Spring Boot main class, pu.xml, sla.xml, embedded vs remote space, local cache/view |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/sql-jdbc.md` | JDBC driver, SQL syntax, DYNAMIC_FILTER hint, EXPLAIN ANALYZE, Spring JdbcTemplate |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/billbuddy-domain.md` | Complete BillBuddy training domain with working examples of all major XAP patterns |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/not-in-aggregator.md` | NOT IN custom aggregator: index-bucket skipping, Person domain example, when/when-not to use |
| `/Users/yuvaldori/Workspace/gs-dev-skill/gigaspaces-xap/references/opentelemetry-tracing.md` | OpenTelemetry distributed tracing: ZipkinTracerBean, span creation, multi-thread patterns, Zipkin setup, common mistakes |

## Quick Decision Guide

```
User wants to...
  ├── Set up Maven / pom.xml            → maven-pom.md
  ├── Define a space-stored class        → pojo-model.md
  ├── Read/write/query the space         → space-operations.md
  ├── React to space events              → event-containers.md
  ├── Run logic colocated with data      → task-execution.md
  ├── Aggregate data server-side         → custom-aggregators.md
  ├── Expose a service via the space     → remoting.md
  ├── Configure / deploy a PU            → processing-unit.md
  ├── Query via SQL / JDBC               → sql-jdbc.md
  ├── See a full working example         → billbuddy-domain.md
  ├── Query NOT IN on an indexed field   → not-in-aggregator.md
  └── Add OpenTelemetry / tracing spans  → opentelemetry-tracing.md
```

---

## Output Behavior Rules

**Snippet** (user asks about a concept or isolated pattern):
- Single class or short example with inline comments explaining each XAP-specific choice
- Include only the relevant import block

**Complete project** (user asks for "working example", "runnable project", "full implementation"):
1. `pom.xml` — with XAP 17.2.1 Maven deps (read `maven-pom.md` first)
2. Model module: `@SpaceClass` POJOs with `@SpaceId`, `@SpaceRouting`, `@SpaceIndex`
3. `pu.xml` or Spring Boot `@Configuration` class — prefer annotations; use `pu.xml` only where annotations are insufficient
4. `application.yml` or `application.properties` for Spring Boot connection settings
5. Runnable main class **or** JUnit test demonstrating the feature end-to-end

---

## Code Style Rules

1. **Annotations over pu.xml** — Use `@EventDriven`, `@Polling`, `@Notify`, `@RemotingService`, `@Transactional` etc. in Java. Fall back to pu.xml only for namespace-level features (SLA, embedded space creation, WAN gateway).
2. **Spring Boot where applicable** — Client applications, REST gateways, and feeder services should use `@SpringBootApplication`; colocated PU logic uses the PU Spring context.
3. **Routing and partitioning** — Every `@SpaceClass` definition must show `@SpaceRouting` on the correct getter. If routing ≠ ID, explain why.
4. **Default topology** — Remote/clustered space proxy. Mention embedded space when the context is a colocated Processing Unit or a unit test.
5. **Java version** — Target Java 17 (`maven.compiler.source=17`).

---

## Anti-Patterns to Flag

Always flag these when you see them in user code or questions:

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `readMultiple(template, Integer.MAX_VALUE)` unbounded | OOM; scans all partitions | Add limit or use `SpaceIterator` with batch size |
| Non-`Serializable` space entry | Task serialization fails at send | Implement `Serializable` (or `Externalizable` for performance) |
| Non-`Externalizable` custom aggregator | Slow serialization across partitions | Implement `Externalizable` in every aggregator |
| Client-side aggregation via `readMultiple` | Transfers all data over network | Use `DistributedTask` or built-in aggregators |
| `GigaSpace` held as non-transient task field | Serialization error | Always mark `@TaskGigaSpace private transient GigaSpace gigaSpace` |
| `GigaSpace` bean missing `tx-manager` when using `@TransactionalEvent` | Startup fails: "GigaSpace is not transactional" | `<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager"/>` — the transaction manager must be set on the bean itself, not just declared |
| Missing `@SpaceRouting` | All objects land on partition 0 | Annotate the correct routing getter |
| Missing `@SpaceId` | XAP cannot manage entry identity | Every `@SpaceClass` needs exactly one `@SpaceId` getter |
| Template query with all nulls | Matches everything — scatter-gather across all partitions | Use SQLQuery with at least a routing condition |
| Writing all entries via embedded (local) proxy in partitioned PU | Entries whose routing key belongs to another partition throw a routing exception | Implement `ClusterInfoAware`, filter by `Math.abs(key.hashCode()) % numPartitions == partitionIndex`; see `processing-unit.md` |
| Using `clustered=true` GigaSpace bean inside a PU to seed data | Semantically wrong — the embedded space IS the local partition, not a cluster gateway | Use `ClusterInfoAware` to write only the partition-local subset |

---

## Core Principles (always apply)

1. **Default constructor required** — Every space class must have a no-arg constructor.
2. **`@SpaceId` is mandatory** — Exactly one getter per class.
3. **`@SpaceRouting` controls partitioning** — Defaults to the SpaceId field; always be explicit.
4. **Null = wildcard in templates** — Use `SQLQuery` for range/complex queries.
5. **Prefer batch operations** — `writeMultiple`, `readMultiple`, `takeMultiple` are 10–50× faster.
6. **Indexes are write-overhead tradeoffs** — Add `@SpaceIndex` only on fields used in filters.
7. **Tasks must be serializable** — Task class + all non-transient fields must implement `Serializable`.
8. **Aggregators must be `Externalizable`** — Custom aggregators are sent to every partition.
9. **Transactions scope** — Wrap read-modify-write; avoid transactions on pure reads.
10. **`@SupportCodeChange`** — Required on DurableTask and custom aggregators for hot-redeploy.

---

## Imports Cheat Sheet

```java
// Core space access
import org.openspaces.core.GigaSpace;
import org.openspaces.core.GigaSpaceConfigurer;
import org.openspaces.core.space.EmbeddedSpaceConfigurer;
import org.openspaces.core.space.SpaceProxyConfigurer;

// POJO annotations
import com.gigaspaces.annotation.pojo.*;
import com.gigaspaces.metadata.index.SpaceIndexType;
import com.gigaspaces.annotation.SupportCodeChange;

// Querying
import com.j_spaces.core.client.SQLQuery;
import com.gigaspaces.client.iterator.SpaceIterator;

// Tasks
import org.openspaces.core.executor.Task;
import org.openspaces.core.executor.DistributedTask;
import org.openspaces.core.executor.DurableTask;
import org.openspaces.core.executor.TaskGigaSpace;
import com.gigaspaces.async.AsyncFuture;
import com.gigaspaces.async.AsyncResult;

// Events
import org.openspaces.events.EventDriven;
import org.openspaces.events.polling.Polling;   // note: sub-package, not events.Polling
import org.openspaces.events.notify.Notify;     // note: sub-package, not events.Notify
import org.openspaces.events.EventTemplate;
import org.openspaces.events.TransactionalEvent;
import org.openspaces.events.adapter.SpaceDataEvent;

// Remoting
import org.openspaces.remoting.RemotingService;
import org.openspaces.remoting.ExecutorProxy;

// Transactions
import org.springframework.transaction.annotation.Transactional;
import org.openspaces.core.transaction.manager.DistributedJiniTxManagerConfigurer;

// Aggregation
import com.gigaspaces.query.aggregators.AbstractPathAggregator;
import com.gigaspaces.query.aggregators.SpaceEntriesAggregatorContext;
import com.gigaspaces.query.aggregators.AggregationSet;
import org.openspaces.core.aggregators.GigaSpaceAggregation;

// Change API
import com.gigaspaces.client.ChangeSet;
import com.gigaspaces.client.ChangeResult;

// Cluster awareness (partition-local PU seeding)
import org.openspaces.core.cluster.ClusterInfo;
import org.openspaces.core.cluster.ClusterInfoAware;

// Notify container type control (separate from @Notify — use @NotifyType to select write/update/take)
import org.openspaces.events.notify.NotifyType;

// IdQuery — lookup by @SpaceId; in com.gigaspaces.QUERY (not com.gigaspaces.client)
import com.gigaspaces.query.IdQuery;
```
