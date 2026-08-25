# WAN Gateway: Replication Filter (Selective Replication)

Discard specific writes from replicating to a target site based on the object's own field values —
e.g. only replicate EU customer records to an EU site. This runs on the **source** space, inspecting
each entry as it's about to replicate, before it ever leaves for the gateway.

## Concept

`IReplicationFilter` is attached directly to the space (via `<os-core:space-replication-filter>`),
not to the gateway PU. It sees every replicated write — not just gateway-bound ones, also intra-cluster
replication (primary→backup) unless you explicitly scope it — and decides per-entry whether to let it
through or discard it.

## Filter implementation

```java
package com.example.wan.filter;

import com.j_spaces.core.IJSpace;
import com.j_spaces.core.cluster.IReplicationFilter;
import com.j_spaces.core.cluster.IReplicationFilterEntry;
import com.j_spaces.core.cluster.ReplicationPolicy;

public class RegionReplicationFilter implements IReplicationFilter {

    @Override
    public void init(IJSpace space, String filterId, ReplicationPolicy policy) {
        // one-time setup; no space/gateway access needed for a simple field check
    }

    @Override
    public void process(int direction, IReplicationFilterEntry entry, String replicationTargetName) {
        // process() fires for EVERY replication channel this space has, not just the WAN
        // gateway — including primary-to-backup sync, where replicationTargetName is a raw
        // container/instance identifier (e.g. "wanSpaceUS_container2_1:wanSpaceUS"), nothing
        // like a gateway name. Only the WAN gateway channel itself is named "gateway:<GatewayName>".
        // Verified by logging replicationTargetName at runtime — confirms both facts: the
        // gateway channel really is "gateway:EMEA", and every backup-sync channel is something
        // else entirely. An under-scoped check here doesn't just miss the gateway target, it can
        // discard entries meant for your own backup replicas — a correctness bug on the local
        // cluster, not just a WAN replication miss.
        if (!replicationTargetName.equals("gateway:EMEA")) {
            return; // not the WAN gateway target we're filtering; let backups and anything else through unmodified
        }
        if (!entry.getClassName().equals("com.example.model.User")) {
            return; // only filtering User objects; every other type passes through
        }
        Object region = entry.getFieldValue("region");
        if (!"EU".equals(region)) {
            entry.discard(); // never replicates to this target
        }
    }

    @Override
    public void close() {
        // release any resources acquired in init()
    }
}
```

`IReplicationFilterEntry.discard()` is the only mechanism — there's no way to *modify* the entry
before it replicates, only pass it through unchanged or drop it entirely for this specific target.

## Wiring — attached to the space PU, only on the filtering side

```xml
<bean id="filter" class="com.example.wan.filter.RegionReplicationFilter" />

<os-core:embedded-space id="space" space-name="wanSpaceUS" mirrored="false" gateway-targets="gatewayTargets">
    <os-core:space-replication-filter>
        <os-core:output-filter ref="filter" />
    </os-core:space-replication-filter>
</os-core:embedded-space>
<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager" />

<os-gateway:targets id="gatewayTargets" local-gateway-name="US">
    <os-gateway:target name="EMEA" />
</os-gateway:targets>
```

`<os-core:output-filter>` filters what leaves this space (the direction relevant to WAN Gateway).
There's also an `<os-core:input-filter>` for filtering incoming replicated writes, which
`IReplicationFilter` also handles via the `direction` parameter — not covered here since it's less
commonly needed for WAN scenarios.

## Structural consequence: this breaks module symmetry

Unlike `active-active.md`'s baseline (identical bean set on both sides → one shared module deployed
twice), a filter attached to only one side is a **structural** difference — an entire extra bean and
XML element present on one side and not the other, not just a different property value. This means
the space PU needs **two separate modules** (one with the filter, one without), even though the
gateway PU can still be the shared module from `active-active.md` (the filter doesn't touch the
gateway PU at all — it's purely a space-side concern).

## Deploy

Gateway PU deploys exactly as in `active-active.md`. Space PU deploys as two distinct jars, not one
shared jar with `-p` overrides:

```bash
gs.sh --server=us-manager   service deploy --zones=US-space   US-space   SpacePU-US.jar    # has the filter
gs.sh --server=emea-manager service deploy --zones=EMEA-space EMEA-space SpacePU-EMEA.jar  # no filter
```

## Verify

Write a mix of values to the filtering site — some that should pass, some that shouldn't — then
confirm the target site only received the subset that should have:

```bash
gs.sh --server=emea-manager space query wanSpaceEMEA \
  com.example.model.User --max-results=50 --columns=userId,region
```

Every result should have `region=EU`; every other object type written alongside `User` should have
replicated in full (the filter above only inspects `User`).

## Pitfalls specific to this scenario

- **`replicationTargetName` is not a stable, predictable format across channels — verify it at
  runtime rather than assuming.** Confirmed by logging it directly: the WAN gateway channel is
  named `"gateway:EMEA"`, but the primary-to-backup sync channel for the same space is something
  entirely different (a raw container/instance identifier, no relation to any gateway or site
  name). Two distinct failure modes follow from this:
  - **Too narrow** (e.g. matching only `"EMEA"` instead of `"gateway:EMEA"`): the check silently
    never matches, `process()` still gets called every time but always falls through as if
    unfiltered, and every write replicates through with no error anywhere — the same
    "looks configured, silently does nothing" failure mode that shows up elsewhere in XAP (see
    `gigaspaces-xap`'s missing-`annotation-support` pitfall).
  - **Too broad** (e.g. matching on a substring or prefix instead of the exact gateway target
    name): risks also matching a backup-sync channel's target name and discarding entries meant
    for your own backup replicas — a correctness bug on the local cluster itself, worse than
    simply failing to filter WAN traffic.
  Always verify with an actual mixed write, check the target site's contents, and don't trust that
  "no errors" means the filter is doing what you think.
- `entry.getFieldValue(...)` returns the raw stored value — cast/compare against the actual type
  (an enum, a `String`, etc.), not its `toString()`.
- The filter instance is shared across every replicated write from that space — keep `process()`
  stateless and side-effect-free beyond logging; don't accumulate per-call state across invocations.
