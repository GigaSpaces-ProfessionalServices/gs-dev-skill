---
name: firewall-and-static-ports
description: >
  Field-validated guidance for running GigaSpaces XAP 17.3.0 across a firewall or through NAT:
  disabling multicast, statically pinning every component's listener ports (Lookup Service,
  Webster, LRMI transport range, JMX/RMI-registry), how a single shared LRMI/JMX range set via
  GS_OPTIONS_EXT is sufficient even for components co-located on one host (XAP allocates
  sequentially within it), which ports actually need to be opened in a firewall versus which stay
  admin-only, and the LRMI network-mapping-file mechanism for
  translating an internal bind address to an externally-reachable one. Verified against a real
  two-container Docker cluster with an actually-enforced iptables firewall boundary, not just
  "the port wasn't published." Use when the user mentions GigaSpaces over a firewall, NAT, static
  ports, com.gs.transport_protocol.lrmi.bind-port, com.gs.multicast.discoveryPort,
  com.gigaspaces.start.httpPort, com.gigaspaces.system.registryPort,
  com.sun.jini.reggie.initialUnicastDiscoveryPort, network_mapping.config, DefaultNetworkMapper,
  com.gs.transport_protocol.lrmi.network-mapping-file, Webster port, JMX port exposure, port
  ranges, or exposing/publishing GigaSpaces ports through Docker, a corporate firewall, or a cloud
  security group.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# Firewall & Static Ports — Configuration Skill

Guidance for making a GigaSpaces XAP 17.3.0 cluster reachable across a firewall or NAT boundary:
disabling multicast, statically pinning every listener port, and knowing which of those ports
actually need to cross the boundary. Where a claim is marked **Confirmed**, it's been reproduced
against a real two-container (Manager + GSC) XAP 17.3.0 cluster on Docker with a real `iptables`
firewall rule enforcing the port boundary — not just inferred from a port being left unpublished,
which this skill's own lab proved is *not* sufficient evidence on a host where the container
network happens to be directly routable (see `references/static-ports.md`).

**Default target version: XAP 17.3.0.**

**Goal this skill's port guidance is scoped to, stated explicitly**: an external client, outside
the firewall, discovers and performs read/write operations against an already-deployed space. A
different goal — PU deployment triggered from outside the firewall, remote JMX-based monitoring —
was not tested and may need different ports open. See `references/static-ports.md` for which port
claims are confirmed for this goal versus not established either way.

**This skill covers making the server side of the cluster reachable across a boundary.** For a
*client* that already has connectivity but can't find/connect to a space (locators, groups,
`CannotFindSpaceException`), use the sibling `remote-proxy-connectivity` skill instead. For the
Manager/GSC setup and port table this skill builds on (`GS_MANAGER_OPTIONS`, `GS_GSC_OPTIONS`,
`GS_MANAGER_SERVERS`, the Manager's REST/ZooKeeper ports), see the sibling
`environment-variables-and-manager` skill — this skill doesn't repeat that material, only the ports
specific to crossing a firewall.

**MANDATORY**: Read the relevant reference file(s) below **before answering any firewall, NAT, or
static-port configuration question**, using paths relative to this skill's own directory
(`references/<file>.md`).

## Reference Files

| Reference Path | Covers |
|---|---|
| `references/static-ports.md` | The full per-component static-port plan (discovery, Webster, LRMI range, JMX), how the LRMI/JMX range is set via the shared `GS_OPTIONS_EXT` — including for co-located components, via XAP's own sequential port allocation — and when isolating with `GS_MANAGER_OPTIONS`/`GS_GSC_OPTIONS` actually helps, which ports must actually be opened in a firewall vs. which stay admin-only, `setenv`/`GS_LOOKUP_LOCATORS` requirements on the far side of the firewall, and where the firewall doc's own port table no longer matches the modern Manager+ZooKeeper architecture |
| `references/network-mapping.md` | The LRMI `network_mapping.config`/`DefaultNetworkMapper` NAT mechanism: file format and classpath-resource loading, per-machine placement, custom mapping logic, and the unconditional-rewrite behavior that can break a component's own internal self-discovery when misapplied |

## Quick Decision Guide

```
User's question is about...
  ├── Which ports need pinning, and to what                                    → static-ports.md
  ├── Which ports actually need to be opened in the firewall/security group    → static-ports.md
  ├── Manager and GSC seem to be fighting over the same ports / the Manager won't discover its own registrar → static-ports.md (GS_OPTIONS_EXT pitfall)
  ├── A server's internal bind address differs from what an external client should dial (NAT) → network-mapping.md
  ├── Deploying network_mapping.config and something breaks (discovery loops, self-connect failures) → network-mapping.md
  └── A client has network access but still can't find/connect to the space     → remote-proxy-connectivity skill instead
```

## Four core properties from the firewall doc

The "Use case" column exists because whether a port needs to be opened depends on the goal — see
`static-ports.md`'s "Goal this section's port list is scoped to." Don't open a port because it's in
this table alone; open it because the deployment's actual goal matches the use case listed.

| Purpose | Property | Mandatory per the doc? | Use case (open it when...) | Notes |
|---|---|---|---|---|
| Disable multicast | `-Dcom.gs.multicast.enabled=false` | Optional | Any remote SpaceProxy/Java client — not needed for a REST v3/SpaceDeck-only client, which never does LUS discovery | The practical default for anything firewalled regardless — see `remote-proxy-connectivity`'s `locators-and-groups.md` for why relying on multicast is fragile even without a firewall in play |
| Unicast discovery / LUS | `-Dcom.gs.multicast.discoveryPort=<port>` | Mandatory | Remote SpaceProxy/Java client discovery — not needed for a REST v3/SpaceDeck-only client | Same value also goes on `initialUnicastDiscoveryPort` (next row) — see `static-ports.md` for why both are set together |
| Explicit unicast discovery | `-Dcom.sun.jini.reggie.initialUnicastDiscoveryPort=<port>` | Mandatory | Paired with the discovery port above — same use case | Defaults to the value above when unset (`0`) — setting both explicitly is the documented, unambiguous form |
| Webster (HTTPD) | `-Dcom.gigaspaces.start.httpPort=<port>` | Mandatory | PU deployment (GSC pulling a package from the Manager) — **not confirmed** as needed even for remote deployment specifically; not needed for a remote SpaceProxy client's read/write operations or for REST v3/SpaceDeck | **Confirmed** still in active use for PU deployment (not legacy), and Manager-only — see `static-ports.md` |

**Not in this table**: `com.gs.transport_protocol.lrmi.bind-port` (the actual data-transport range —
the single most consequential property here) and `com.gigaspaces.system.registryPort` (JMX). Both
get their own detailed treatment in `static-ports.md`, including a combined single-container
Manager+GSC test's `GS_OPTIONS_EXT`-range failure and what it does and doesn't imply about sizing a
shared LRMI range for co-located components.

## Troubleshooting

Both reference files end with their own **Pitfalls index** (symptom → cause → fix). Check the
table in the relevant file before diagnosing — the GS_OPTIONS_EXT/LRMI-range conflict and the
network-mapper self-reference behavior both look like generic "discovery is broken" symptoms, but
have a specific, already-documented cause.
