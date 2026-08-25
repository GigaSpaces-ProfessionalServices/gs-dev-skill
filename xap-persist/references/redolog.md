# Redolog Internals: SQLite Storage, Flush, and Replay

**Read this warning before using anything in this file beyond a one-off diagnostic.** Every
technique here — flushing the redolog, deserializing it, replaying it — works only by reaching into
`com.gigaspaces.internal.*` classes: `SqliteRedoLogFileStorage`, `DBSwapRedoLogFileConfig`,
`IReplicationOrderedPacket`, `IInternalRemoteJSpaceAdmin`, and friends. These are GigaSpaces'
**private implementation classes, not public API** — no compatibility guarantee applies to them
across versions, including patch versions. Code built on this file's patterns can break on any XAP
upgrade with no deprecation warning first. There does not appear to be a public/supported API for
directly inspecting or replaying redolog content — if you find one, prefer it over everything below.

## What the redolog is

The redolog is the pending-replication backlog on a primary space instance — operations that have
happened locally but haven't yet been acknowledged by whatever they're replicating to (a mirror, a
backup, a WAN gateway target). It exists specifically so those operations survive a restart before
they've been durably replicated elsewhere. Configuring it to swap to SQLite storage (rather than
staying purely in memory) is what makes it possible to inspect its contents directly, which is the
whole reason this lab's techniques exist.

## Configuring SQLite-backed redolog storage

```java
public class CustomSpaceConfig extends EmbeddedSpaceBeansConfig {
    @Override
    protected void configure(EmbeddedSpaceFactoryBean factoryBean) {
        super.configure(factoryBean);
        factoryBean.setMirrored(true); // redolog only accumulates when there's something to replicate to
        Properties properties = new Properties();
        properties.setProperty("cluster-config.groups.group.repl-policy.swap-redo-log.storage-type", "sqlite");
        properties.setProperty("cluster-config.groups.group.repl-policy.redo-log-memory-capacity", "20");
        properties.setProperty("cluster-config.groups.group.repl-policy.redo-log-capacity", "10000");
        factoryBean.setProperties(properties);
    }
}
```

`redo-log-memory-capacity` is how many packets stay in memory before swapping to SQLite;
`redo-log-capacity` is the hard cap (memory + disk combined) before the space starts rejecting
writes or blocking, depending on other cluster settings. A `mirrored="true"` space with no mirror
actually deployed (the deliberate setup in this lab) accumulates redolog indefinitely, since nothing
ever acknowledges it — that's how to generate one on demand for testing, but exactly the failure
mode `gigaspaces-xap`'s `mirror-persistence.md` warns about for a *real* deployment with no
exception handler.

**If the actual goal is bounding the redolog's memory footprint for a more stable process
(shutdown or otherwise) — this property alone is the supported answer.** It's continuous and
automatic: XAP swaps to SQLite storage on its own once the in-memory count crosses this threshold,
no manual flush call needed. Don't reach for the manual `FlushRedoLogTask` approach below for this;
it calls an internal, unsupported API (see the warning at the top of this file) and isn't the
right tool for an operational concern like this one — it's for on-demand diagnostic inspection of
what's already on disk, a different use case.

## Deploy and generate a redolog

```bash
./gs.sh host run-agent --auto --gsc=2
./gs.sh pu deploy --partitions=1 --ha my-app-space-pu <path>/my-app-space/target/my-app-space-1.0-SNAPSHOT.jar
# Run a writer client against it — see space-operations.md for basic write patterns.
# Because there's no mirror deployed, every write accumulates in the redolog with nothing draining it.
```

**The space's own name is not the same as the PU/app-name you deployed it under** —
`pu deploy --partitions=1 --ha my-app-space-pu <jar>` deploys under app-name `my-app-space-pu`, but
the actual space name (set in the PU's own config, e.g. `redolog` in this lab) is what `space`
subcommands need. `./gs.sh space info --type-stats my-app-space-pu` fails with `404 Not Found` —
use the real space name, findable via `pu list`'s `SPACE` column if it's not obvious from the code:

```bash
./gs.sh space info --type-stats redolog       # space's own object counts, by real space name
./gs.sh space list-instances redolog
./gs.sh space info-instance --replication-stats redolog~1_1   # per PRIMARY instance
```

Confirmed live: `--replication-stats` output has one block per replication channel (backup, mirror,
gateway — whichever apply). The redolog size for a specific target is `Redo Log Retained Size`
under *that target's own block* — e.g. with a `mirrored="true"` space and no mirror deployed, the
`[mirror-service_container:mirror-service]` block shows `Channel State: DISCONNECTED` and a nonzero
`Redo Log Retained Size`/`Redo Log Retained Weight`, while the `[..._container:<space-name>]`
backup-sync block (actively connected) shows `0` for both — the redolog is only piling up behind
the channel nothing is draining, exactly as expected.

## Flushing the redolog to disk (for inspection, not operational use)

This section exists so you can force the *entire* current in-memory redolog onto disk on demand —
e.g. mid-investigation, when you want to inspect everything that's there right now, rather than
waiting for `redo-log-memory-capacity` to naturally push older entries over. It is not an
operational tool for bounding memory or smoothing shutdown — see the note in the previous section
for the supported way to do that. Forcing a full flush requires a colocated `DistributedTask`
calling an internal admin API — there's no client-side equivalent, and no public API for this:

```java
public class FlushRedoLogTask implements DistributedTask<Integer, Integer> {
    @TaskGigaSpace
    private transient GigaSpace injectedGigaSpace;

    @Override
    public Integer execute() throws Exception {
        IInternalRemoteJSpaceAdmin admin =
                (IInternalRemoteJSpaceAdmin) injectedGigaSpace.getSpace().getAdmin();
        return admin.flushRedoLogToStorage(); // returns count of packets flushed
    }

    @Override
    public Integer reduce(List<AsyncResult<Integer>> results) throws Exception {
        int sum = 0;
        for (AsyncResult<Integer> r : results) {
            if (r.getException() != null) throw r.getException();
            sum += r.getResult();
        }
        return sum;
    }
}
```

Run it from a plain client via `gigaSpace.execute(new FlushRedoLogTask())` (see `task-execution.md`
in `gigaspaces-xap` for the general `DistributedTask` pattern this follows). After it returns, the
SQLite file is at:

```
$GS_HOME/work/redo-log/redolog/sqlite_storage_redo_log_redolog_container1
```
(`redolog` is the space name, `redolog_container1` the specific container — substitute your own
space/container names.) It's openable in any SQLite browser, but **its contents are XAP's internal
serialized packet format, not the original objects** — don't expect to read meaningful data by
inspecting the SQLite rows directly; deserializing them requires the same internal packet classes
used to write them (see below).

## Replaying a redolog into a space

Reading packets back out requires re-registering the same type descriptors the original space used
(`gs.getTypeManager().registerTypeDescriptor(...)`) before replay, and reconstructing objects from
each packet's raw `IEntryData` via `PojoIntrospector` — entirely internal-API territory, no
shortcut. At a high level:

```bash
./gs.sh host run-agent --auto --gsc=2
./gs.sh pu deploy --partitions=1 --ha my-app-space-pu <path>/my-app-space/target/my-app-space-1.0-SNAPSHOT.jar
# space is empty at this point

java -Dcom.gs.home="$BACKUP_HOME" -cp ... com.example.ProcessRedoLog redolog redolog_container1
# VM arg points at wherever the redolog SQLite files were backed up to, not the live $GS_HOME;
# program args are <space-name> <container-name>
```

Compare the replayed space's contents against what was originally written to confirm nothing was
lost or corrupted in the round-trip.

## Automatic flush on shutdown (no manual task needed, since XAP 16.4)

```
com.gs.redolog.flush.on.shutdown=true        # copies the redolog to $GS_HOME/work/redolog-backup automatically
com.gs.redolog.flush.notify.class=com.example.MyFlushNotifier   # implements RedologFlushNotifier; jar goes under $GS_HOME/platform/ext
com.gs.shutdownhook.timeout=<seconds>         # increase to give the flush time to finish before the shutdown hook is killed
```

This is the supported path for "don't lose the redolog on a planned shutdown" — reach for it before
building manual flush tooling like `FlushRedoLogTask` above, which exists for *inspecting* an
already-running space's redolog, not as a substitute for this.

## Pitfalls

- **Everything in this file beyond the "automatic flush on shutdown" section is unsupported-API
  territory.** Treat it as a diagnostic/forensic tool for a specific investigation, not something to
  build ongoing product functionality on top of — an XAP upgrade can silently break it with no
  warning, unlike a public API deprecation.
- **A `mirrored="true"` space with no mirror deployed is a deliberate test setup here, not a
  pattern to replicate.** In a real deployment this is exactly the failure mode
  `gigaspaces-xap`'s `mirror-persistence.md` warns about: no exception handler needed because there's
  no persistence target to fail, but the redolog still grows unbounded for the same underlying
  reason (nothing is draining it).
- **The SQLite file's contents are not the original objects.** Don't expect to query or read them
  directly for any purpose beyond confirming the file exists/has grown — actual content inspection
  needs the same internal deserialization path as replay.
