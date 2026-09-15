# LRMI Network Mapping (NAT Address Translation)

## What it is

LRMI (GigaSpaces' internal RMI-like transport) always instantiates a network mapper on startup.
Unless `com.gs.transport_protocol.lrmi.network-mapper` names a different implementation class,
that's always `DefaultNetworkMapper` (confirmed: `LRMIRuntime` logs `"Creating default network
mapper"` at debug level whenever that property is unset — this is the normal case, not an opt-in
feature). `DefaultNetworkMapper` reads a line-based address-translation table and rewrites any
matching exported `ServerAddress` (the host:port a stub advertises to remote callers) from its real
bind address to a different address before handing it to a consumer.

## File format and location

The distribution ships a template at `$GS_HOME/config/network_mapping.config`:

```
#Line separated mapping, each line has the following format:
#<original ip>:<original port>,<mapped ip>:<mapped port> for instance:
#10.0.0.1:4162,212.321.1.1:3000
```

The property that names this file, `com.gs.transport_protocol.lrmi.network-mapping-file`, defaults
to the literal string `config/network_mapping.config` — but it's loaded via
`Thread.currentThread().getContextClassLoader().getResourceAsStream(...)`, **not** as a plain
filesystem path. That means it's resolved as a classpath resource: a value that looks like a normal
relative or absolute filesystem path will not necessarily do what it appears to (a classloader
resource lookup is relative to a classpath root, never the OS filesystem root, regardless of a
leading `/`). The safe, confirmed-working way to configure a mapping is to overlay the *contents* of
the shipped `$GS_HOME/config/network_mapping.config` in place, since `$GS_HOME/config` is already on
the classpath by default — not to redirect the property to some other file location.

Format: each non-comment line must split into exactly two comma-separated `host:port` pairs (each
side split again on `:`) or the mapper throws `IllegalArgumentException` at startup with the exact
message shown in the shipped template's comment.

The mapping file belongs on the consuming machine — the one dialing out to the private-to-public
mapped address — not on the machine whose address is being mapped. Each machine in the network
keeps its own translation file, with one line per peer it needs to reach through a NAT'd address;
mapping is configured per-consumer, not globally.

## Custom mapping logic

An alternative to the file-based mapper: implement `com.gigaspaces.lrmi.INetworkMapper` (a single
`ServerAddress map(ServerAddress serverAddress)` method, called on each new connection from proxy to
service) in a class placed in `$GS_HOME/lib/platform/ext`, and point
`com.gs.transport_protocol.lrmi.network-mapper` at its class name. The class needs a no-argument
constructor — the runtime instantiates it that way.

## Unconditional rewrite affects internal callers too

**This is the important part, and it's exactly what makes the mechanism unsafe in some topologies.**
`DefaultNetworkMapper`'s translation applies uniformly to *every* consumer of a mapped address —
there is no concept of "this caller is remote and needs translation" vs. "this caller is local and
doesn't." **Confirmed**: mapping a container's real bridge IP to a different external-facing
address (to make a Docker-published port advertise something a host-side client could dial) broke
that same component's *own* internal self-referential admin/discovery client identically — a
Manager process trying to reach its own exported `ServiceGridRegistrar` via the translated address,
which isn't reachable from inside the same container that exported it. Symptom: a tight, permanent
`Connection refused` retry loop (50,000+ occurrences within 30 seconds) and `gs.sh service deploy`
failing with `Cluster manager information has not been discovered yet`. Emptying the mapping file
back to just its comment template dropped the error count to baseline transient-startup-noise
levels (single digits) immediately.

**Practical rule**: only map an address via this mechanism when nothing that shares that address's
identity ever needs to reach it from the *same* side the mapping was written for. This is safe on
genuinely separate physical/virtual hosts (a real Manager machine behind a real NAT device doesn't
loop back through its own external-facing address to reach itself). It is **not** safe for a
single-container/single-host setup where one process needs to reach its own advertised address
directly — in that case, either don't map that specific address, or restructure so the
self-referential component and the mapped-address boundary are on different hosts entirely (see
this skill's own lab: splitting a combined Manager+GSC container into two separate containers made
this mapping unnecessary in the first place, since the host machine could already route directly to
each container's real bridge address without any translation).

## When you don't actually need this mechanism

Before reaching for network mapping, confirm the "external" client genuinely cannot route to the
server's real bind address at all. On a Docker host specifically, the bridge network's subnet is
often directly routable from the host itself (`ip route` will show a route like `172.x.x.x/24 dev
br-...`) — in that case a host-side client can reach a container's real address on any port that a
firewall rule permits, with no NAT/address-translation needed, and `network_mapping.config` would be
solving a problem that doesn't exist on that host. Confirm this before adding mapping-file
complexity to a lab or a real deployment.

## Pitfalls index

| Symptom | Cause | Fix |
|---|---|---|
| A Manager's own internal discovery/admin client loops forever with `Connection refused` against an address that a plain external TCP client can reach fine | `network_mapping.config` maps that same address to something unreachable from inside the process that also needs to reach it via its own real identity | Don't map an address that any self-referential internal consumer also needs to dial directly; empty the mapping file to confirm this is the cause before changing anything else |
| Setting `com.gs.transport_protocol.lrmi.network-mapping-file` to an absolute path and the mapping never seems to apply | The property is loaded via `ClassLoader.getResourceAsStream`, a classpath lookup, not a filesystem path | Overlay the contents of the default `$GS_HOME/config/network_mapping.config` in place instead of redirecting the property elsewhere |
| Assuming a NAT/mapping-file setup is required to let an external client reach a containerized GigaSpaces cluster | The container network may already be directly routable from the client's host (common on a single Docker host) | Check `ip route` for the bridge subnet before adding mapping-file complexity; if it's already routable, only a firewall/port rule is needed, not address translation |
