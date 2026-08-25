---
name: xap-wan
description: >
  Expert guidance for GigaSpaces XAP WAN Gateway: multi-site replication topologies
  (active-passive, active-active), bootstrapping a new site from an existing one's data,
  selective replication filters, and cross-site conflict resolution. Use when the user mentions
  WAN Gateway, WAN replication, multi-site XAP, cross-site DR, gateway delegator, gateway sink,
  os-gateway, GatewaySinkFactoryBean, requires-bootstrap, AdminBootstrapInitiator,
  IReplicationFilter, ConflictResolver, onDataConflict, DataConflict, or active-active/
  active-passive space replication. Default XAP target version is 17.3.0 unless the user specifies
  otherwise.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# GigaSpaces XAP WAN Gateway Skill

WAN Gateway replicates space data between independently-clustered XAP sites — typically
geographically separate (e.g. US and EMEA) — for disaster recovery, data locality, or multi-region
active-active applications. Each site is a normal, fully independent XAP grid; the gateway is a
separate Processing Unit per site that ships changes across the link.

**Default target version: XAP 17.3.0**

---

## Reference Files

**MANDATORY**: Read the relevant reference file(s) below **before generating any code or answering
any question** about WAN Gateway. Select by scenario, then read using paths relative to this skill's
own directory (`references/<file>.md`).

| Reference Path | Covers |
|------|--------|
| `references/active-passive.md` | One-directional replication: delegator-only source site, sink-only target site |
| `references/active-active.md` | Bidirectional replication: both sites run a delegator and a sink, shared module + `-p` overrides |
| `references/bootstrap.md` | Bringing a new site online from an existing site's data: `requires-bootstrap`, `AdminBootstrapInitiator`, staged deploy order |
| `references/replication-filter.md` | Selectively discarding replicated writes per-object with `IReplicationFilter` |
| `references/conflict-resolution.md` | Reconciling records both sites modified independently, before the gateway link existed: `ConflictResolver`/`onDataConflict` |

## Quick Decision Guide

```
User wants to...
  ├── One site replicates to another, one-way (DR)        → active-passive.md
  ├── Both sites accept writes, replicate to each other    → active-active.md
  ├── Add a new site and seed it from an existing site      → bootstrap.md
  ├── Replicate only some objects/fields to a target site   → replication-filter.md
  └── Handle records both sites wrote before connecting     → conflict-resolution.md
```

A gateway PU can combine more than one of these — e.g. active-active *and* a filter on one
direction (see `replication-filter.md`, which is a filtered active-active topology). Read every
reference file relevant to the combination, not just one.

---

## Core Building Blocks (apply regardless of scenario)

```xml
xmlns:os-gateway="http://www.openspaces.org/schema/core/gateway"
...
http://www.openspaces.org/schema/core/gateway
http://www.openspaces.org/schema/core/gateway/openspaces-gateway.xsd
```

- **`<os-gateway:delegator>`** — send-only. Declared in the gateway PU; `<os-gateway:delegation
  target="..."/>` names which remote gateway to send to.
- **`<os-gateway:sink>`** — receive-only. Declared in the gateway PU; `<os-gateway:sources><os-gateway:source
  name="..."/></os-gateway:sources>` names which remote gateway(s) to accept from, and
  `local-space-url` names the local space to write incoming replicated data into.
- **`<os-gateway:lookups>`/`<os-gateway:lookup>`** — how the gateway's embedded LUS discovers the
  local and remote gateway processes (`gateway-name`, `host`, `discovery-port`,
  `communication-port`). Both the local and every remote gateway need an entry.
- **`<os-gateway:targets>`/`<os-gateway:target>`** — declared in the **space** PU (not the gateway
  PU), referenced from `<os-core:embedded-space gateway-targets="...">`. This is what actually
  makes writes to that space attempt delegation — the gateway PU's delegator only exists to carry
  them once the space marks them for replication.
- A site with **both** a delegator and a sink is a full peer (active-active). Delegator-only = pure
  source (active-passive's active side). Sink-only = pure target (active-passive's passive side).
- **Deploying**: `gs.sh --server=<manager-host> service deploy --zones=<zone> [-p key=value ...]
  <app-name> <jar>`. `-p` overrides feed the PU's `PropertyPlaceholderConfigurer` — this is how one
  shared module gets deployed twice (once per site) with site-specific values, instead of
  maintaining near-duplicate modules. See `active-active.md` for when to prefer this over separate
  per-site modules (`active-passive.md`'s topology is asymmetric enough that it isn't worth it).

---

## Anti-Patterns / Pitfalls (cross-cutting — apply to every scenario)

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `requires-bootstrap="${someProperty}"` directly on `<os-gateway:sink>` | Fails at XML-parse time, before Spring ever resolves the placeholder — Xerces validates the element's `requires-bootstrap` attribute as XSD `boolean` and rejects a raw `${...}` string outright | Declare the sink as a plain `org.openspaces.core.gateway.GatewaySinkFactoryBean` bean instead of the `<os-gateway:sink>` element when this value must vary by deploy-time `-p` override — a bean property stays an ordinary `String` until Spring's own type conversion runs, which happens *after* placeholder resolution; see `bootstrap.md` |
| A gateway process reachable on disjoint, non-routed subnets for its own site's manager vs. the remote gateway | `GS_NIC_ADDRESS` (or equivalent bind address) is JVM-wide — one address binds and advertises *every* LRMI export in that process, including the gateway's embedded LUS. It can't bind two different addresses for two different peers | Give the gateway process one address reachable by both its own site's manager and the remote gateway — route or peer the two sites' networks so that's possible, rather than trying to keep them segmented |
| Assuming a connected (`INTACT`) gateway sink means data is flowing | `requires-bootstrap="true"` genuinely blocks automatic replication — a sink can show connected with the underlying space still at 0 objects until an explicit bootstrap runs | Don't mistake a bootstrap-pending sink for a broken link; check `bootstrap.md` before assuming a gateway misconfiguration |

---

## Imports / Class Cheat Sheet

```java
// Gateway core (used from pu.xml or as plain beans)
import org.openspaces.core.gateway.GatewaySource;
import org.openspaces.core.gateway.GatewaySinkFactoryBean;   // plain-bean form — see bootstrap.md

// Replication filter (references/replication-filter.md)
import com.j_spaces.core.IJSpace;
import com.j_spaces.core.cluster.IReplicationFilter;
import com.j_spaces.core.cluster.IReplicationFilterEntry;
import com.j_spaces.core.cluster.ReplicationPolicy;

// Conflict resolution (references/conflict-resolution.md)
import com.gigaspaces.cluster.replication.gateway.conflict.ConflictResolver;
import com.gigaspaces.cluster.replication.gateway.conflict.DataConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.DataConflictOperation;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryNotInSpaceConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryAlreadyInSpaceConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryVersionConflict;
import com.gigaspaces.cluster.replication.gateway.conflict.EntryLockedUnderTransactionConflict;

// Admin API bootstrap trigger (references/bootstrap.md)
import org.openspaces.admin.Admin;
import org.openspaces.admin.AdminFactory;
import org.openspaces.admin.gateway.Gateway;
import org.openspaces.admin.gateway.GatewaySinkSource;
import org.openspaces.admin.gateway.BootstrapResult;
```
