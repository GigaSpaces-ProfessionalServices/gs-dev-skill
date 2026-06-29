# OpenTelemetry Distributed Tracing in GigaSpaces XAP

This reference covers how to instrument GigaSpaces Processing Units with OpenTelemetry spans, using Zipkin as the tracing backend. Based on the working demo at `~/Workspace/OpenTelemetryDemo`.

**XAP Version:** 17.2.2  
**OTel API Version:** `io.opentelemetry:opentelemetry-api:1.40.0`  
**Zipkin:** v2 JSON API (`/api/v2/spans`)

---

## Key Constraint: No OTel SDK in PUs

The OTel **SDK** (`SdkTracerProvider`) cannot be bundled inside a GigaSpaces PU — it has a hard dependency on `opentelemetry-api-incubator` which is not on the GSC classpath and cannot be added without modifying GigaSpaces internals.

**Use instead:**
- `opentelemetry-api` at `provided` scope (already on the GSC classpath)
- `ZipkinTracerBean` from `org.gigaspaces:xap-reporter` — this is GigaSpaces' own OTel SDK integration that registers with `GlobalOpenTelemetry`

---

## Maven Dependencies

```xml
<!-- OTel API is already on the GSC classpath — compile only, do NOT bundle -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.40.0</version>
    <scope>provided</scope>
</dependency>

<!-- ZipkinTracerBean: registers GigaSpaces' OTel SDK with GlobalOpenTelemetry -->
<dependency>
    <groupId>org.gigaspaces</groupId>
    <artifactId>xap-reporter</artifactId>
    <version>17.2.1-SNAPSHOT</version>
    <scope>compile</scope>
</dependency>
```

---

## Step 1: Enable Tracing in the GigaSpaces Grid

Edit `$GS_HOME/bin/setenv-overrides.sh`:

```bash
export GS_OPTIONS_EXT="\
  -Dcom.gs.tracing.enabled=true \
  -Dcom.gs.tracing.exporter=zipkin \
  -Dcom.gs.tracing.zipkin.endpoint=http://localhost:9411/api/v2/spans"
```

Restart the grid after editing. Official docs: https://docs.gigaspaces.com/latest/admin/admin-distributed-tracing.html

---

## Step 2: Initialize ZipkinTracerBean (TraceHelper Pattern)

Each PU needs a singleton that initializes `ZipkinTracerBean` once on startup and registers it with `GlobalOpenTelemetry`. The standard pattern used across all PUs in the demo:

```java
import com.gigaspaces.tracing.ZipkinTracerBean;
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import java.util.concurrent.Callable;

public class TraceHelper {

    private static final String ZIPKIN_URL =
            System.getProperty("otel.zipkin.endpoint", "http://localhost:9411/");

    private static ZipkinTracerBean tracingBean;
    private static TraceHelper helper;

    protected TraceHelper() {
        tracingBean = new ZipkinTracerBean("my-service-name")
                .setStartActive(true)
                .setZipkinUrl(ZIPKIN_URL);
        tracingBean.start(); // registers with GlobalOpenTelemetry
    }

    public static synchronized TraceHelper getInstance() {
        if (helper == null)
            helper = new TraceHelper();
        return helper;
    }

    public static void close() {
        if (tracingBean != null) {
            try { tracingBean.destroy(); } catch (Exception ignored) {}
        }
    }

    // Convenience wrapper: creates a named span, runs callable, handles errors
    public static <T> T wrap(String name, Callable<T> c) throws Exception {
        Tracer tracer = GlobalOpenTelemetry.getTracer("my-service-name");
        Span span = tracer.spanBuilder(name).startSpan();
        try (Scope scope = span.makeCurrent()) {
            return c.call();
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.toString());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Initialize in `@PostConstruct`:**
```java
@PostConstruct
public void construct() {
    TraceHelper.getInstance(); // must be called before any span creation
    tracer = GlobalOpenTelemetry.getTracer("my-service-name");
    // ...
}

@PreDestroy
public void destroy() {
    TraceHelper.close();
    // ...
}
```

---

## Step 3: Creating Spans

### Basic span around an operation

```java
Tracer tracer = GlobalOpenTelemetry.getTracer("my-service-name");

Span span = tracer.spanBuilder("operation-name")
        .setAttribute("key", "value")
        .startSpan();
try (Scope scope = span.makeCurrent()) {
    // your operation here
    gigaSpace.write(obj);
} finally {
    span.end();
}
```

### Using the wrap() helper for Callable operations

```java
TraceHelper.wrap("feeder-write", () -> {
    gigaSpace.write(pru);
    return null;
});
```

### Span with attributes (feeder example)

```java
Span span = tracer.spanBuilder("feeder-write")
        .setAttribute("id", id)
        .startSpan();
try (io.opentelemetry.context.Scope scope = span.makeCurrent()) {
    gigaSpace.write(pru);
} finally {
    span.end();
}
```

### Span around a ChangeSet operation (streamer example)

```java
Span span = tracer.spanBuilder("streamer-change")
        .setAttribute("id", id)
        .startSpan();
ChangeResult<MyType> result;
try (io.opentelemetry.context.Scope scope = span.makeCurrent()) {
    result = gigaSpace.change(idQuery, changeSet);
} finally {
    span.end();
}
```

### Span around aggregation queries (IMPORTANT: must be inside active span)

`DefaultGigaSpace.wrap()` checks `Span.current().isValid()` before creating a child span — the aggregation call **must** be inside an active span to produce traces:

```java
Span span = tracer.spanBuilder("aggregation-loop")
        .setAttribute("thread_id", threadId)
        .setAttribute("aggprop", aggProp)
        .startSpan();
AggregationResult result;
try (io.opentelemetry.context.Scope scope = span.makeCurrent()) {
    result = gigaSpace.aggregate(query, new AggregationSet().add(new MyAggregator("field")));
} finally {
    span.end();
}
```

---

## Step 4: Multi-threaded Span Pattern

When submitting tasks to an executor, create the span in the submitting thread, then obtain the `Tracer` again inside the worker thread from `GlobalOpenTelemetry`:

```java
// In @PostConstruct — submitting thread
for (int i = 0; i < concurrentThreads; i++) {
    Span span = tracer.spanBuilder("submit-task")
            .setAttribute("thread_id", i)
            .startSpan();
    int finalI = i;
    TraceHelper.wrap("aggregate", () -> {
        executorService.submit(new MyTask(finalI));
        return 0;
    });
    span.end();
}

// Inside the worker Runnable
private class MyTask implements Runnable {
    public void run() {
        // Get tracer fresh — GlobalOpenTelemetry is initialized by TraceHelper
        Tracer taskTracer = GlobalOpenTelemetry.getTracer("my-service-name");
        while (running) {
            Span span = taskTracer.spanBuilder("my-operation").startSpan();
            try (Scope scope = span.makeCurrent()) {
                // work
            } finally {
                span.end();
            }
        }
    }
}
```

---

## Alternative: Direct Zipkin JSON (No SDK at all)

If `ZipkinTracerBean` is not available (e.g. older XAP version), send spans directly via HTTP POST to Zipkin's v2 JSON API:

```java
// POST to http://localhost:9411/api/v2/spans
// Content-Type: application/json
// Body: [{"traceId":"<32 hex chars>","id":"<16 hex chars>","name":"op-name",
//          "timestamp":<epoch micros>,"duration":<micros>,
//          "localEndpoint":{"serviceName":"my-service"},
//          "tags":{"key":"value"}}]
```

See `ZipkinSender.java` in the demo for a self-contained implementation with no extra dependencies.

---

## GigaSpaces System Properties for Tracing

| Property | Description | Example |
|---|---|---|
| `com.gs.tracing.enabled` | Enable/disable OTel tracing | `true` |
| `com.gs.tracing.exporter` | Exporter type | `zipkin` |
| `com.gs.tracing.zipkin.endpoint` | Zipkin spans endpoint | `http://localhost:9411/api/v2/spans` |
| `otel.zipkin.endpoint` | Endpoint read by `ZipkinTracerBean` | `http://localhost:9411/` |

---

## Running the Demo

Full working demo at `~/Workspace/OpenTelemetryDemo`.

```bash
# 1. Build
cd ~/Workspace/OpenTelemetryDemo && mvn clean package

# 2. Start Zipkin
docker run -d -p 9411:9411 --name zipkin openzipkin/zipkin

# 3. Enable OTel in GigaSpaces (edit setenv-overrides.sh, then restart grid)

# 4. Deploy PUs in order
scripts/deploy-processor.sh   # creates space
scripts/deploy-loader.sh      # loads data, exits when done
scripts/deploy-feeder.sh      # continuous writer (after loader finishes)
scripts/deploy-streamer.sh    # continuous ChangeSet updater
scripts/deploy-aggregator.sh  # aggregation queries — primary tracing target

# 5. View traces: http://localhost:9411
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bundling `opentelemetry-api` as `compile` in PU | Use `provided` — it's on the GSC classpath |
| Bundling OTel SDK (`SdkTracerProvider`) | Don't — use `ZipkinTracerBean` instead |
| Calling `GlobalOpenTelemetry.getTracer()` before `ZipkinTracerBean.start()` | Always call `TraceHelper.getInstance()` in `@PostConstruct` before using the tracer |
| Running aggregation outside an active span | `DefaultGigaSpace.wrap()` only creates child spans when `Span.current().isValid()` — wrap the `gigaSpace.aggregate()` call in a span |
| Multiple `ZipkinTracerBean` instances per PU | Use singleton pattern (`TraceHelper`) — only one per PU |
