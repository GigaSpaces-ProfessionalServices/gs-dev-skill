# WAN Gateway: Conflict Resolution

Reconcile a record that both sites modified **independently before the gateway link ever
existed** (or while it was down) — once connected, each side's accumulated redo log drains into the
other, and any record both sides touched surfaces as a genuine conflict that needs a policy
decision, not just "last write wins" by default.

## Concept

`ConflictResolver.onDataConflict(sourceGatewayName, conflict)` is invoked once per conflicting
entry, told which remote gateway the conflicting write came *from*, and handed a `DataConflict`
containing one or more `DataConflictOperation`s to resolve. The handler decides, per operation,
whether to `override()` (apply the incoming write) or leave it alone / `abortAll()` (discard the
incoming side entirely).

This is wired via `<os-gateway:error-handling>` on the **sink**, so it only fires for conflicts on
*incoming* replicated data — it has nothing to do with the replication filter (`replication-filter.md`),
which runs on the outgoing side before data ever leaves.

## Handler implementation

```java
package com.example.wan.conflict;

import com.gigaspaces.cluster.replication.gateway.conflict.ConflictResolver;
import com.gigaspaces.cluster.replication.gateway.conflict.DataConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.DataConflictOperation;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryAlreadyInSpaceConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryLockedUnderTransactionConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryNotInSpaceConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryVersionConflict;

public class SourcePriorityConflictHandler extends ConflictResolver {

    @Override
    public void onDataConflict(String sourceGatewayName, DataConflict conflict) {
        // Policy: US always wins. Conflicts sourced FROM EMEA are discarded outright;
        // conflicts sourced FROM US override whatever the local (EMEA) value was.
        // Branch on sourceGatewayName, not on any field of the conflicting object itself —
        // the handler already tells you which site the incoming write originated from.
        if ("EMEA".equals(sourceGatewayName)) {
            conflict.abortAll();
            return;
        }
        if ("US".equals(sourceGatewayName)) {
            for (DataConflictOperation operation : conflict.getOperations()) {
                if (!operation.hasConflict()) {
                    continue;
                }
                // Override for every conflict cause this policy cares about. Leaving a
                // cause un-handled here means that operation is left unresolved (effectively
                // rejected) — enumerate every cause you actually intend to override.
                if (operation.getConflictCause() instanceof EntryNotInSpaceConflict
                        || operation.getConflictCause() instanceof EntryAlreadyInSpaceConflict
                        || operation.getConflictCause() instanceof EntryVersionConflict
                        || operation.getConflictCause() instanceof EntryLockedUnderTransactionConflict) {
                    operation.override();
                }
            }
        }
    }
}
```

This handler is identical on both sites' gateway PUs — it branches internally on
`sourceGatewayName`, so there's no per-site XML difference needed for it, unlike
`replication-filter.md`'s filter (which only exists on one side).

## Wiring — `<os-gateway:error-handling>` on the sink

```xml
<bean id="conflictResolver" class="com.example.wan.conflict.SourcePriorityConflictHandler" />

<os-gateway:sink id="sink" local-gateway-name="${localGatewayName}" gateway-lookups="gatewayLookups"
                 start-embedded-lus="true" local-space-url="${localSpaceUrl}">
    <os-gateway:sources>
        <os-gateway:source name="${remoteGatewayName}"/>
    </os-gateway:sources>
    <os-gateway:error-handling conflict-resolver="conflictResolver"
                                max-retries-on-tx-lock="5"
                                tx-lock-retry-interval="100" />
</os-gateway:sink>
```

`max-retries-on-tx-lock`/`tx-lock-retry-interval` govern retries specifically for
`EntryLockedUnderTransactionConflict` — a transient conflict cause, not a genuine data conflict — so
they exist independently of whatever the resolver decides to do once retries are exhausted.

## Reproducing a genuine conflict (for testing this deliberately)

A real conflict needs both sides to have written the *same* record before the gateway link exists,
which means deploying in a specific order rather than the usual single-step deploy of everything
together:

```
1. Deploy BOTH sites' space PUs only — no gateway PUs yet.
2. Write to both sites independently (same record identities, different values) — no link exists,
   so each accumulates its own version with no replication happening at all.
3. Deploy BOTH sites' gateway PUs. The moment they connect, each side's already-built-up redo
   log drains into the other — this is when conflicts actually surface and resolve.
```

```bash
# 1+2: space PUs deployed, writes happen here, gateway PUs not deployed yet
gs.sh --server=us-manager   service deploy --zones=US-space   -p localSpaceName=wanSpaceUS   ... US-space   SpacePU.jar
gs.sh --server=emea-manager service deploy --zones=EMEA-space -p localSpaceName=wanSpaceEMEA ... EMEA-space SpacePU.jar
# write to both sites here, independently

# 3: NOW deploy the gateway PUs — conflicts surface and resolve on connection
gs.sh --server=us-manager   service deploy --zones=US-gateway   ... US-gateway   GatewayPU.jar
gs.sh --server=emea-manager service deploy --zones=EMEA-gateway ... EMEA-gateway GatewayPU.jar
```

## Verify

Check both gateway GSCs' own log output for the resolution outcome (grep for `resolution` in
whatever way your deployment exposes GSC logs — e.g. the GSC's log file on disk, or your log
aggregation/monitoring setup) — with the policy above, US should show `ABORT` for every
EMEA-sourced conflict, EMEA should show `OVERRIDE` for every US-sourced one.

Then confirm every conflicting record on both sides now matches whichever side's policy said should
win.

## Pitfalls specific to this scenario

- **A conflict only surfaces this way if the gateway link genuinely didn't exist while both sides
  wrote.** If you deploy in the usual single-step order (space + gateway together, as in
  `active-active.md`), one side's write typically replicates before the other side ever gets a
  chance to write the same identity — you won't reliably reproduce a real conflict, and testing a
  conflict handler that way risks false confidence that it works when it was never actually
  exercised.
- **Un-handled `DataConflictOperation` causes are effectively rejected, not passed through.** If a
  new conflict cause is introduced (e.g. a XAP version bump adds one) and the `instanceof` chain
  above doesn't enumerate it, that operation silently falls through `onDataConflict` unresolved —
  audit the cause list against the current XAP version rather than assuming it's exhaustive forever.
- Branch on `sourceGatewayName`, the parameter `onDataConflict` is called with — not on some field
  inside the conflicting object itself. The gateway has already resolved which site the incoming
  write came from; re-deriving that from the object's own data is redundant and error-prone.
