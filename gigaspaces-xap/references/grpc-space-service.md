# XAP gRPC SpaceService

**Requires GigaSpaces 18.0.1 or later** (not available in 17.3.0). The REST v3 process hosts a gRPC
server next to its HTTP server. Its one service, `com.gigaspaces.rest.grpc.SpaceService`, exposes the
everyday space operations to any language with gRPC support (Python, Go, C#, Node.js, C++, Java).

- The client needs **no GigaSpaces jar and no lookup settings**: only the `.proto` files and `host:port`.
- Every call takes **two network hops** (client → REST v3 → partitions) and two conversions
  (protobuf ↔ `SpaceDocument`). It is always slower than the native `GigaSpace` client.

| | |
|---|---|
| Server side | an agent started with `--restv3` (e.g. `gs.sh host run-agent --auto --restv3`) |
| gRPC port | 9091 (`GS_REST_V3_GRPC_PORT`); HTTP stays on 9090 (`GS_REST_V3_PORT`) |
| Contract | `http://<host>:9090/api/v3/grpc/proto.zip` (`common.proto`, `space_crud.proto`, `space_admin.proto`, `space_tasks.proto`, `space_service.proto`) |
| Java package of generated code | `com.gigaspaces.rest.grpc.proto` |
| Reflection | on, so `grpcurl -plaintext host:9091 list` works without the `.proto` files |
| Health | standard `grpc.health.v1.Health`, `Check` → `SERVING` |
| JVM | REST v3 needs JDK 25; JVM options go in `GS_RESTV3_OPTIONS` |

---

## When to Use gRPC (decide this first)

| Factor | Native `GigaSpace` | JDBC v3 | REST v3 HTTP | gRPC SpaceService |
|---|---|---|---|---|
| Client language | Java | Java (+ BI tools via gateway) | any | any with gRPC |
| Path to data | direct to partitions | direct to partitions | via REST v3 | via REST v3 |
| GigaSpaces jars on client | yes | yes | no | no |
| Queries | ids, templates, parameterized `SQLQuery`, projections, aggregations, iterators | full SQL incl. JOIN, GROUP BY, `EXPLAIN` | — | ids, templates, a WHERE clause, **no parameters, no projections** |
| Transactions, events, notify, remoting, colocated tasks | ✅ | ❌ | ❌ | ❌ (can only execute an already-registered durable task) |
| Value types | any Java type | SQL types | JSON | string, int64, double, bool |
| Speed | fastest | — | 2 hops | 2 hops, **1.2x to 4.8x slower than native (2.2x average)** |

**Use gRPC when:** the client is **not Java** and does request/response work on entries (by id, by
template, by a simple condition); you want a typed, versioned contract shared across languages; or
the client must not carry GigaSpaces jars or join the lookup service.

**Do not use gRPC when:** the client is Java on a latency-critical path (use native); you need
JOIN/GROUP BY/SQL tooling (use JDBC v3, see `sql-jdbc.md`); you need transactions, event containers,
notifications, iterators, aggregations, remoting or colocated tasks (all native-only); or the entries
have nested objects, collections, `byte[]` or `BigDecimal` that the client must write back.

**gRPC vs REST v3 HTTP:** same server process. gRPC gives binary encoding, one multiplexed HTTP/2
connection, per-call deadlines and generated typed clients. HTTP needs only `curl` and has Swagger.
Use gRPC from a program and HTTP from a person or a shell script.

---

## Java Client Setup

### Maven (client module)

The client depends on gRPC and protobuf only. Download `proto.zip` from the target server into
`src/main/proto` and generate the stubs at build time.

```xml
<properties>
    <!-- Match the protobuf version shipped with GigaSpaces (lib/required/protobuf-java.jar);
         18.0.1 ships 4.34.2 -->
    <protobuf.version>4.34.2</protobuf.version>
    <grpc.version>1.85.1</grpc.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>${protobuf.version}</version>
    </dependency>
    <!-- @javax.annotation.Generated on generated stubs -->
    <dependency>
        <groupId>org.apache.tomcat</groupId>
        <artifactId>annotations-api</artifactId>
        <version>6.0.53</version>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals><goal>compile</goal><goal>compile-custom</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

If the same module also uses the native client (`xap-openspaces`), pin `protobuf-java` in
`dependencyManagement` so the stubs and GigaSpaces jars share one protobuf version.

### Channel, Stub, Deadline

```java
import com.gigaspaces.rest.grpc.proto.SpaceServiceGrpc;
import com.gigaspaces.rest.grpc.proto.SpaceServiceGrpc.SpaceServiceBlockingStub;
import io.grpc.Grpc;
import io.grpc.InsecureChannelCredentials;
import io.grpc.ManagedChannel;
import java.util.concurrent.TimeUnit;

// ONE channel per process: it is an HTTP/2 connection, thread-safe, multiplexed, self-reconnecting.
// Use TlsChannelCredentials outside a lab.
ManagedChannel channel = Grpc.newChannelBuilderForAddress("rest-host", 9091,
        InsecureChannelCredentials.create()).build();
SpaceServiceBlockingStub stub = SpaceServiceGrpc.newBlockingStub(channel);

// A deadline on EVERY call: a blocking call with no deadline waits forever on a hung server.
// withDeadlineAfter is per call. Set it on each call, not once on a long-lived stub.
stub.withDeadlineAfter(5, TimeUnit.SECONDS).count(/* ... */);

// on shutdown
channel.shutdown();
if (!channel.awaitTermination(5, TimeUnit.SECONDS)) channel.shutdownNow();
```

### The `Value` Type (four kinds only)

```java
import com.gigaspaces.rest.grpc.proto.SpaceDocument;
import com.gigaspaces.rest.grpc.proto.Value;

static Value of(Object v) {
    Value.Builder b = Value.newBuilder();
    switch (v) {
        case null -> { }                              // null = value not set
        case String s  -> b.setStringValue(s);
        case Integer i -> b.setInt64Value(i);
        case Long l    -> b.setInt64Value(l);
        case Double d  -> b.setDoubleValue(d);
        case Boolean x -> b.setBoolValue(x);
        default -> throw new IllegalArgumentException("no gRPC Value kind for " + v.getClass());
    }
    return b.build();
}

// A document = type name + map of Values
SpaceDocument doc = SpaceDocument.newBuilder().setTypeName("Product")
        .putProperties("productId", of("p1"))
        .putProperties("category", of("books"))
        .putProperties("price", of(9.5))
        .putProperties("stock", of(100))
        .build();
```

---

## Operations

The service has 14 calls: `Write`, `WriteMultiple`, `RegisterType`, `ReadById`, `ReadByIds`,
`ReadMultiple`, `TakeById`, `TakeByIds`, `TakeMultiple`, `Clear`, `Count`, `Change`,
`QueryExpression`, `ExecuteDurableTask`. Every request carries `space_name`.

### Register a Document Type

```java
import com.gigaspaces.rest.grpc.proto.RegisterTypeRequest;

// Property types are Java type names. Registration is NOT idempotent: a second call fails
// with INVALID_ARGUMENT "already registered". A client that registers at start-up must catch it.
stub.withDeadlineAfter(5, TimeUnit.SECONDS).registerType(RegisterTypeRequest.newBuilder()
        .setSpaceName("mySpace").setTypeName("Product")
        .putProperties("productId", "String")
        .putProperties("category", "String")
        .putProperties("price", "Double")
        .putProperties("stock", "Integer")
        .addId("productId")
        .setRoutingProperty("productId")   // optional; defaults to the id
        .putIndexes("category", "EQUAL")
        .build());
```

A Java service's POJO type is already registered. A gRPC client names it by its **fully qualified
class name** (`com.example.model.Order`).

### Write, Read, Take

```java
import com.gigaspaces.rest.grpc.proto.*;

// write = insert-or-update; it REPLACES the whole entry
stub.withDeadlineAfter(5, TimeUnit.SECONDS).write(
        WriteRequest.newBuilder().setSpaceName("mySpace").setDocument(doc).build());

// prefer batches: the hop is paid per call, not per entry
stub.withDeadlineAfter(5, TimeUnit.SECONDS).writeMultiple(WriteMultipleRequest.newBuilder()
        .setSpaceName("mySpace").addAllDocuments(docs).build());

// readById: a miss is NOT an error; check hasDocument()
ReadByIdResponse r = stub.withDeadlineAfter(5, TimeUnit.SECONDS).readById(ReadByIdRequest.newBuilder()
        .setSpaceName("mySpace").setTypeName("Order").setId(of("o1"))
        .setRouting(of("c1"))              // optional; without it the call goes to every partition
        .build());
if (r.hasDocument()) { /* ... */ }

// readByIds returns ONLY the hits, with no placeholder for a miss. Match results by id property, not position.
// One routing value applies to all ids; per-id routing is not supported.
ReadByIdsResponse many = stub.withDeadlineAfter(5, TimeUnit.SECONDS).readByIds(ReadByIdsRequest.newBuilder()
        .setSpaceName("mySpace").setTypeName("Product")
        .addIds(of("p1")).addIds(of("p2")).build());
```

### Template Queries and SQL Expressions

```java
// Template (readMultiple / takeMultiple / count / clear): exact-value match only
ReadMultipleResponse books = stub.withDeadlineAfter(5, TimeUnit.SECONDS).readMultiple(
        ReadMultipleRequest.newBuilder().setSpaceName("mySpace")
                .setTemplate(SpaceDocument.newBuilder().setTypeName("Product")
                        .putProperties("category", of("books")))
                .setMaxEntries(500)        // ALWAYS set: 0 / unset = unlimited
                .build());

// queryExpression: a SQLQuery-style WHERE clause for ranges, OR, IN, LIKE.
// NO ? parameters: quote every value yourself (see Anti-Patterns)
QueryExpressionResponse q = stub.withDeadlineAfter(5, TimeUnit.SECONDS).queryExpression(
        QueryExpressionRequest.newBuilder().setSpaceName("mySpace").setTypeName("Product")
                .setExpression("category = " + sqlLiteral(category) + " AND price >= 60")
                .setMaxEntries(500)
                .build());

static String sqlLiteral(String v) { return "'" + v.replace("'", "''") + "'"; }
```

There is no projection, ORDER BY, JOIN, aggregation or iterator. A query returns whole documents
in **one** response message. For large results, page by key (`productId > 'last-seen'` with
`maxEntries`).

### Change (prefer over read-modify-write)

```java
// criteria_type "id" (criteria = id value) or "sql" (criteria = WHERE clause; "" = all entries of type)
ChangeResponse c = stub.withDeadlineAfter(5, TimeUnit.SECONDS).change(ChangeRequest.newBuilder()
        .setSpaceName("mySpace").setTypeName("Product")
        .setCriteriaType("id").setCriteria(of("p1"))
        .setCriteriaRouting(of("p1"))      // optional, only when routing property ≠ id
        .addOperations(ChangeOperation.newBuilder().setOp("increment").setPath("stock").setValue(of(5)))
        .addOperations(ChangeOperation.newBuilder().setOp("set").setPath("price").setValue(of(9.99)))
        .build());
long changed = c.getNumberOfChangedEntries();   // 0 = nothing matched; not an error
```

Ops: `set`, `unset`, `increment`, `decrement`, `addToCollection`, `addAllToCollection`,
`removeFromCollection`, `putInMap`, `removeFromMap` (`putInMap`/`removeFromMap` also take `key`).

### Python

```python
import grpc
from space import common_pb2, space_crud_pb2, space_service_pb2_grpc  # generated by grpc_tools.protoc

def value(v):
    if isinstance(v, bool):  # check bool BEFORE int: True is an int in Python
        return common_pb2.Value(bool_value=v)
    if isinstance(v, int):
        return common_pb2.Value(int64_value=v)
    if isinstance(v, float):
        return common_pb2.Value(double_value=v)
    return common_pb2.Value(string_value=str(v))

with grpc.insecure_channel("rest-host:9091") as channel:
    stub = space_service_pb2_grpc.SpaceServiceStub(channel)
    r = stub.ReadById(space_crud_pb2.ReadByIdRequest(
            space_name="mySpace", type_name="Product", id=value("p1")), timeout=5)
    if r.HasField("document"):
        print({k: getattr(v, v.WhichOneof("kind")) for k, v in r.document.properties.items()})
```

Generate with: `python -m grpc_tools.protoc -I proto --python_out=gen --grpc_python_out=gen proto/space/*.proto`
(the output directory must exist).

---

## Type Mapping (POJO types shared with Java services)

| Java property type | Client receives | Client sends | Notes |
|---|---|---|---|
| `String` | string | string | |
| `int`/`long`/`short`/`byte` (+ boxed) | int64 | int64 | range-checked: out of range → `INVALID_ARGUMENT` |
| `double`/`float` (+ boxed) | double | double | |
| `boolean`/`Boolean` | bool | bool | |
| enum | string (constant name) | string (constant name) | unknown name → `INVALID_ARGUMENT` |
| `java.util.Date`, `java.sql.*` | ISO-8601 string, UTC | ISO-8601 string | |
| `null` | value not set | value not set | |
| **nested object, collection, map, `byte[]`, `BigDecimal`** | **description string** (`"SpaceDocument [...]"`) | **cannot be sent** | read-only text; writing it back → `INVALID_ARGUMENT` |

**Design rule:** a type shared with gRPC clients should be **flat**: strings, numbers, booleans,
enums, dates. Put anything else in properties the gRPC side never writes, or in a separate type.
`write` replaces the whole entry, so a gRPC read-modify-write **silently nulls** every property the
client could not represent. Use `change` for updates from gRPC clients.

---

## Status Codes (verified on 18.0.1)

| Status | Cause |
|---|---|
| `INVALID_ARGUMENT` | value does not convert or fit; type registered twice; negative `maxEntries`; bad doc in `writeMultiple` (the message names the position) |
| `NOT_FOUND` | unknown **space** |
| `INTERNAL` | unknown **type**; bad SQL expression (description contains the server stack trace, so log it and don't show it to end users); anything unmapped |
| `PERMISSION_DENIED` | security denial |
| `FAILED_PRECONDITION` | protective-mode rejection |
| `DEADLINE_EXCEEDED` | client deadline passed; the server **may still have completed** the operation |
| `RESOURCE_EXHAUSTED` | response larger than the client's max inbound message size (4 MB default) |
| `UNAVAILABLE` | REST v3 down or unreachable. This is the one to retry on |

Batch semantics: `writeMultiple` converts **all** documents before writing any (one bad document →
nothing written), but there is **no transaction** over gRPC. A failure during the space write itself
is not rolled back.

---

## Security and Operations

- Secured grid: send `authorization` metadata on every call, `Basic <base64(user:password)>` or
  `Bearer <token>` (Java: `stub.withCallCredentials(...)`). Grid privileges apply per operation;
  `registerType` also needs `PROVISION_PU`. Calls are audited. Use TLS: Basic over plaintext is
  readable on the wire. *(Derived from the 18.0.1 server code; not yet live-tested.)*
- REST v3 is now **in the data path**: size its heap/CPU for gRPC load (`GS_RESTV3_OPTIONS`; 512 MB
  heap by default), and monitor it. An unbounded query is fully materialized in REST v3's heap before
  the client rejects it.
- For availability, run REST v3 on more than one manager host behind a load balancer or gRPC
  client-side load balancing. *(Not live-tested.)*
- Each call is traced as an OpenTelemetry span (`xap-grpc`) with status code and duration.
- The `.proto` files are versioned with the **server**. Re-download `proto.zip` and regenerate on
  every GigaSpaces upgrade.

### Measured cost (2 Linux hosts, 0.2 ms ping, no TLS, 104 runs)

- gRPC slower in every run: **1.2x–4.8x**, **2.2x average**; single thread 2.4x; ~110–165 µs added per
  single-entry call at p50.
- At 50 threads the gap narrows to 1.9x; batches narrow it further (`writeMultiple` of 50: 1.3x).

---

## Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| Concatenate user input into `queryExpression` / SQL `change` (no `?` support, so this is SQL injection) | Quote every value (`'` doubled) or allow only known values |
| Leave `maxEntries` unset (protobuf 0 = unlimited) | Set it on every query; page by key for large results |
| Raise `maxInboundMessageSize` to "fix" `RESOURCE_EXHAUSTED` | Bound the query with `maxEntries`; raise the limit only for a known size |
| Read a document, modify a field, `write` it back | `change` only the fields you mean to change |
| Write back a property received as a description string | Model shared types flat; keep complex properties out of gRPC writes |
| Create a channel per call | One `ManagedChannel` per process, shared by all threads |
| Calls without a deadline | `withDeadlineAfter(...)` / `timeout=` on every call |
| Match `readByIds` results by position | Match by the id property; misses are dropped |
| `Count`/`Clear` with no template | Always set the template type name; an empty `ClearRequest` **empties the whole space** |
| Treat `INTERNAL` as "server down, retry" | `INTERNAL` also means unknown type or bad SQL; retry only on `UNAVAILABLE` |
| Retry a timed-out non-idempotent call (`change` with `increment`) | Retry only operations safe to repeat (`write` of the same document) |
| Single-entry calls in a loop | `writeMultiple`, `readByIds`, `takeByIds`: one hop for many entries |
| gRPC from a latency-critical Java service | Native `GigaSpace` client (see `space-operations.md`) |
| Ship `.proto` files from an older server version | Re-download `proto.zip` and regenerate after every upgrade |
| Suggest gRPC on XAP 17.3.0 or earlier | It requires 18.0.1+; on older versions use REST v3 HTTP or the native/JDBC clients |
