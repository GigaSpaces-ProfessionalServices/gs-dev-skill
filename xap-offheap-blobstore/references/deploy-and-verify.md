# Deploying an Off-Heap Space and Verifying In-Place Reads

## Deploy with properties (no pu.xml)

The default space template reads `space.mx.*` deploy properties, so a plain space deploy is enough:

```bash
./gs.sh space deploy --partitions=2 --ha \
    -p space.mx.off-heap.enabled=true \
    -p space.mx.off-heap.threshold=200m \
    -p space.mx.persistent=false \
    -p space.mx.cache-entries-percentage=0 \
    offheapDemo
```

| Property | Meaning |
|---|---|
| `space.mx.off-heap.enabled=true` | Store entries in an off-heap blobstore |
| `space.mx.off-heap.threshold` | How much off-heap memory the blobstore may use |
| `space.mx.persistent=false` | Keep entries in memory only. An off-heap blobstore is never persistent |
| `space.mx.cache-entries-percentage` | Size of the blobstore's **on-heap internal cache**, as a percentage of the heap. **Default: 20.** `0` switches it off, so every read goes to the off-heap block |

**Which setting decides whether reads hit the block:** `cache-entries-percentage`. Entries held in
the internal cache are already-built objects on the heap and are served from there; the block is not
read at all. That is correct and fast. **Don't set it to 0 in production just to get in-place
reads.** A warm cache beats reading the block. Use `0` for testing and benchmarking, so that every
read exercises the block path.

## Deploy as a processing unit (pu.xml)

You need a `pu.xml` when the space also needs something the properties can't express, such as a
space filter:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:os-core="http://www.openspaces.org/schema/core"
       xmlns:blob-store="http://www.openspaces.org/schema/off-heap-blob-store"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.openspaces.org/schema/core http://www.openspaces.org/schema/core/openspaces-core.xsd
       http://www.openspaces.org/schema/off-heap-blob-store http://www.openspaces.org/schema/off-heap-blob-store/openspaces-off-heap-blob-store.xsd">

    <!-- memory-threshold = space.mx.off-heap.threshold -->
    <blob-store:off-heap-blob-store id="offHeapBlobStore" memory-threshold="200m"/>

    <os-core:embedded-space id="space" space-name="mySpace">
        <!-- persistent = space.mx.persistent ; cache-entries-percentage = space.mx.cache-entries-percentage -->
        <os-core:blob-store-data-policy blob-store-handler="offHeapBlobStore"
                                        persistent="false"
                                        cache-entries-percentage="0"/>
    </os-core:embedded-space>
</beans>
```

The PU jar needs `xap-openspaces` only at `provided` scope. The grid supplies it.

## Check what you deployed

In the container log of each space instance, look for these lines (for
both the property deploy and the `pu.xml` deploy above):

```
created blob-store handler class=com.gigaspaces.blobstore.offheap.OffHeapBlobStoreHandler persistentBlobStore=false
BlobStore space data internal cache size=0
Space started [... policy=blob-store, persistency-mode=blob-store ...]
```

The `internal cache size` figure is a **number of entries**, not the percentage you set. With
`cache-entries-percentage` left at its default, a GSC with a 512 MB heap logged
`internal cache size=20971`. Any non-zero value means the cache is on.

## Grid sizing note (lab / local dev)

One container holds one space instance. `--partitions=2 --ha` needs 4 containers.
`gs.sh demo` occupies its 4 containers with its own `demo` space, so start an agent with enough
containers instead: `./gs.sh host run-agent --auto --gsc=6`.

## Requirements

- GigaSpaces **18.0.1+** for the in-place reads. 18.0.1 is built for **Java 25**, so set
  `maven.compiler.release` to `25`.
- Maven: `org.gigaspaces:xap-openspaces:${gigaspaces.version}` from
  `https://maven-repository.openspaces.org`.

---

## Measuring "entries built"

There is **no product API** that reports how many entries a space decoded from the block. The
training lab counts a server-side debug line:

- The logger is `com.gigaspaces.cache`, at level `FINE`.
- The space writes one line per built entry, containing `Blobstore- entry loaded from off heap`.
- The space is identified in the line as `container [<container>:<space>]`.

The lab's `BuiltEntriesTask` is a `DistributedTask` sent to every primary. It does three things:

1. Sets that logger to `FINE`.
2. Detaches the logger from its parent handlers, so the container log doesn't fill up.
3. Attaches a counting `Handler` that filters on the space name, and returns the counts summed in
   `reduce`.

Treat it as a **diagnostic aid only**:

- It depends on the text of an internal debug line, which can change in any release.
- The logger is container-wide. Two programs counting the same space at once reset each other.
- Successive tasks may load in different class loaders. Find the handler again by class **name**
  and read it through a JDK interface such as `LongSupplier`, not by casting to your own class.
- Always send a STOP action (logger level back to `null`, parent handlers back on) when you finish.

For a production check, compare operation latency on a large, wide type before and after the
upgrade instead. Don't ship the log-counting handler.

## Verification procedure

1. **Run a control first.** Use an operation that must build every entry: an iteration or
   `readMultiple` without a projection, or `maxEntry`. Its built count must equal the number of
   entries. If it shows 0, entries are coming from the internal cache. Redeploy with
   `cache-entries-percentage=0` to observe the block path.
2. **Run the operation you care about.** 0 built entries (or exactly the number of matches, for a
   whole-entry read) means the values were read in place.
3. **Check the container log lines** listed above.
4. **Compare against expected results computed on the client.** A wrong aggregation returns a
   plausible number, so you need the expected value next to it.
