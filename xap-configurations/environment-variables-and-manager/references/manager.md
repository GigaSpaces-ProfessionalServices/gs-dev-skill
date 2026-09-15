# GigaSpaces Manager

The Manager stacks the LUS and GSM together with an embedded Apache ZooKeeper and a REST API, and
is the recommended way to run these services — not a standalone GSM/LUS pair (see
`environment-variables.md`'s note that `GS_GSM_OPTIONS`/`GS_LUS_OPTIONS` are ignored once a process
runs in Manager mode). Benefits over the standalone stack:

- Space leader election uses ZooKeeper instead of the LUS — more robust, stays consistent through
  network partitions.
- The GSM now uses ZooKeeper to elect a single active/managing GSM among the cluster — the rest
  stand by and only monitor until a new leader is elected. The older standalone stack ran GSMs
  "active-active" instead: every GSM was simultaneously active and could manage the grid, with no
  single leader. See "ZooKeeper client (GSM leader election)" below for how the election itself
  works.
- With MemoryXtend, the last primary is stored in ZooKeeper automatically, instead of requiring a
  shared NFS setup.

## Local development: a single standalone Manager

```bash
./gs.sh host run-agent --auto
```

This starts the LUS, ZooKeeper, GSM, and REST API together (visible in the Manager log,
`$GS_HOME/logs`). ZooKeeper's own files land in `$GS_HOME/work/manager/zookeeper`; the (V2) REST API
comes up on `localhost:8090`. Add `--restv3 --webui` to also start the newer V3 Manager API
alongside it, on `localhost:9090` by default — see the ports section below.

**`--auto` is for local development only, never for a shared or production setup.** It binds to
`localhost` by design and is not reachable from other machines — use it only on a developer's own
machine. For a Manager other machines need to actually reach, use one of the two setups below
instead.

## Shared server: a single instance reachable from other machines (non-HA)

For one Manager that other machines need to reach — a shared dev/test environment, not production
HA (see below for that) — set `GS_MANAGER_SERVERS` to this host's own address and start it
explicitly with `--manager` instead of `--auto`, plus `--restv3 --webui` for the newer management
API/console:

```bash
export GS_MANAGER_SERVERS=<this-host's-reachable-address>
./gs.sh host run-agent --manager --restv3 --webui
```

**Also set `GS_NIC_ADDRESS` to that same address if this host's own hostname doesn't already resolve
to one.** On a host where `hostname` resolves to a loopback address in `/etc/hosts` (the common
Debian/Ubuntu default, e.g. `127.0.1.1 <hostname>`), setting only `GS_MANAGER_SERVERS` leaves the
Lookup Service bound to that loopback address — nothing outside the machine can reach it, despite the
setup looking correct — because `GS_NIC_ADDRESS` (which controls the actual bind address; see
`environment-variables.md`'s multi-NIC section) defaults to `` `hostname` `` and resolves the same
way. Set `GS_NIC_ADDRESS` explicitly to the same reachable address as `GS_MANAGER_SERVERS` to fix
it. The legacy V2 REST endpoint (port `8090`) stays bound to `127.0.0.1` in this setup regardless of
`GS_NIC_ADDRESS`; only the V3 REST API (port `9090`, via `--restv3`) is reachable from another
machine.

Every other machine joining this fleet (GSCs, GSAs) needs `GS_MANAGER_SERVERS` set to that same
address too. This is still a single point of failure — no ZooKeeper quorum, no Manager failover —
use the 3-Manager cluster below for anything that needs actual HA.

## Production: a 3-Manager cluster for high availability

A production deployment needs a cluster of Managers across multiple hosts. Requirements:

- **Exactly 3 machines** (an odd number). With an even number of Managers, consistency can't be
  guaranteed across a network partition — 2 Managers doesn't give you HA over the old "2 LUS + 2
  GSM" model, it gives you a tie. 5 is theoretically possible in ZooKeeper terms but not currently
  supported by GigaSpaces (starting 5 Managers starts 5 embedded ZooKeeper instances, which isn't
  the supported topology).
- **One Manager per host.** Starting more than one Manager on the same host is not supported.

Setup:

1. Edit `$GS_HOME/bin/setenv-overrides.sh`/`.bat` and set `GS_MANAGER_SERVERS` to the list of
   Manager hosts, e.g.:
   ```bash
   export GS_MANAGER_SERVERS=alpha,bravo,charlie
   ```
2. Copy the modified `setenv-overrides.sh`/`.bat` to every machine that runs a GigaSpaces Agent
   (not just the three Manager hosts — every machine in the fleet needs to know where the Managers
   are).
3. Run `./gs.sh host run-agent --auto` (or `.bat` on Windows) on the three Manager machines
   themselves.

**Set `GS_MANAGER_SERVERS` on the cluster's own machines (Managers, GSCs, GSAs); a remote/independent
client outside the cluster should use `GS_LOOKUP_LOCATORS` instead — not both on the same process.**
`GS_MANAGER_SERVERS` already includes LUS information — don't also set `GS_LOOKUP_LOCATORS`.
**Confirmed** against a real local Manager (XAP 17.3.0): setting both doesn't produce vague
"confusing discovery behavior" — a client's `SpaceProxyConfigurer` fails immediately, before any
discovery attempt, with `java.lang.IllegalStateException: Ambiguous locators: Manager locators:
[...], explicit locators: [...]`. It's a fail-fast validation error, not a silent supersede or a
harmless redundancy — see the sibling `remote-proxy-connectivity` skill's `locators-and-groups.md`
for the client-side detail. **Confirmed** against the same real cluster: a GSC given only
`GS_MANAGER_SERVERS=manager1,manager2,manager3` (no `GS_LOOKUP_LOCATORS`, no explicit locators
anywhere) logged `GSC started successfully [groups=[xap-17.3.0],
locators=[jini://manager1:4174/, jini://manager2:4174/, jini://manager3:4174/]]` and registered with
all three GSMs — `GS_MANAGER_SERVERS` genuinely derives working LUS locators on its own, not just
per the docs. If a client is genuinely failing to connect and this conflict is ruled out, that's
client-side discovery territory — see the sibling `remote-proxy-connectivity` skill's
`locators-and-groups.md`, not this file.

If any Manager's ZooKeeper ports were changed from default, the extended syntax carries that
per-host (semicolons — quote the whole value on Unix/Linux so the shell doesn't treat them as
command separators):

```bash
GS_MANAGER_SERVERS="alpha;zookeeper=2000:3000;lus=4242,bravo;zookeeper=2100:3100,charlie;zookeeper=2200:3200"
```

## Configuration: ports

Every port below can be overridden via a system property (e.g. through `setenv-overrides`):

| Port | System Property / Env Var | Default |
|---|---|---|
| REST (V2 Manager API) | `com.gs.manager.rest.port` | 8090 |
| REST (V3 Manager API) | `GS_REST_V3_PORT` (env var, not a system property) | 9090 |
| Zookeeper | `com.gs.manager.zookeeper.discovery.port`, `com.gs.manager.zookeeper.leader-election.port`, `com.gs.zookeeper.client.port` | 2888, 3888, 2181 |
| Lookup Service | `com.gs.multicast.discoveryPort` | 4174 |

**Apache ZooKeeper requires that every Manager can reach every other Manager.** If you change
ZooKeeper's ports, apply the change consistently across all three Managers.

The V3 Manager API (`9090`) is a complete OpenAPI 3.1.0-based rewrite of the management interface —
config at `$GS_HOME/config/ui/rest-v3.yaml`, Swagger UI at
`http://<manager>:9090/api/v3/swagger-ui/index.html#` — and only comes up when the Manager is
started with `--restv3 --webui`. It runs *alongside* the older V2 API (`8090`), which is deprecated
but still the default when those flags aren't passed. See `environment-variables.md`'s
`GS_REST_V3_PORT` entry, and don't confuse the two when troubleshooting REST connectivity — a client
timing out against `9090` when only V2 was started is expected, not a bug.

**Confirmed** (found in the V2 API's own OpenAPI spec): the V2 API also exposes `DurableTask`
management for a space — `GET /spaces/{id}/listDurableTask` and
`GET /spaces/{id}/unregisterDurableTask/{uuid}` — letting an operator inspect or unregister a
`DurableTask` without touching application code. See the `gigaspaces-xap` skill's `task-execution.md`
for what a `DurableTask` is and how it's registered/run in the first place; this is the operational
(Manager-side) counterpart to that. Whether the V3 API exposes the same capability wasn't checked.

The Lookup Service port above is what a client's `GS_LOOKUP_LOCATORS`/`.lookupLocators()` should
point at when connecting directly to a Manager-based grid's embedded LUS (assuming
`GS_MANAGER_SERVERS` isn't already handling that for it — see above). For the client side of that
connection — reading the resulting `FinderException`, the silent-multicast-masking trap, and so on
— see the sibling `remote-proxy-connectivity` skill's `locators-and-groups.md`.

The ZooKeeper client port can also be set via the `GS_ZOOKEEPER_CLIENT_PORT` environment variable,
or through ZooKeeper's own client config file (see below). Where more than one of these is set,
priority (highest to lowest) is: Java system property, environment variable, config file.

## ZooKeeper configuration file (`zoo.cfg`)

The embedded ZooKeeper instance starts from a default config at `$GS_HOME/config/zookeeper/zoo.cfg`,
preset with:

| Property | Description | Value |
|---|---|---|
| `tickTime` | ZooKeeper's base time unit, in ms. | 1000 |
| `initLimit` | Ticks allowed for a follower to connect and sync to the leader. | 10 |
| `syncLimit` | Ticks allowed for a follower to stay synced with the leader. | 10 |
| `clientPort` | Port ZooKeeper listens on for client connections. | 2181 |
| `maxSessionTimeout` | Maximum session timeout the server allows a client to negotiate, in ms. | 60000 |
| `autopurge` | Automatic purging of snapshots and transaction logs. | enabled by `purgeInterval > 0` |
| `autopurge.purgeInterval` | Purge task interval, in hours (0 disables it). | 1 |
| `autopurge.snapRetainCount` | Snapshots (and their transaction logs) to retain; the rest are deleted. | 3 |

**To override it**, set the `GS_ZOOKEEPER_SERVER_CONFIG_FILE` environment variable or the
`com.gs.zookeeper.config-file` system property to point at a custom file. **Confirmed** against a
real 3-Manager cluster (XAP 17.3.0, Docker): mounting a custom file with `maxSessionTimeout=45000`
and `autopurge.snapRetainCount=5` via `GS_ZOOKEEPER_SERVER_CONFIG_FILE` and restarting all three
Managers produced `maxSessionTimeout set to 45000 ms` and `autopurge.snapRetainCount set to 5` in
every Manager's actual ZooKeeper startup log — the override genuinely takes effect, on all quorum
members, not just cosmetically accepted.

**Pitfall found during that same verification, not documented anywhere in the official docs**: a
custom `zoo.cfg` supplied this way **must explicitly set `dataDir` and `dataLogDir`**. The built-in
default file has these set internally (to paths under `$GS_HOME/work/manager/zookeeper/`), but that
default isn't inherited by a custom override file — omitting them fails the Manager's ZooKeeper
startup outright (`java.io.IOException: The 'dataDir' configuration is missing from the zoo.cfg
file`, thrown from `org.openspaces.zookeeper.grid.XapZookeeperContainer`) and the container
crash-loops retrying every few seconds rather than starting degraded. Set them explicitly, e.g.:
```
dataDir=${com.gs.work}/manager/zookeeper/data
dataLogDir=${com.gs.work}/manager/zookeeper/log
```
## ZooKeeper client (GSM leader election)

The Manager stack uses ZooKeeper leader election to pick one GSM as the active/managing GSM among
the cluster. Until a leader is elected (e.g. no quorum yet), the other GSMs only monitor the
cluster rather than managing it. Tunable via:

| System Property | Default |
|---|---|
| `com.gs.manager.leader-election.zookeeper.connection-timeout` | 5000 |
| `com.gs.manager.leader-election.zookeeper.session-timeout` | 15000 |
| `com.gs.manager.leader-election.zookeeper.retry-timeout` | `Integer.MAX_VALUE` |
| `com.gs.manager.leader-election.zookeeper.retry-interval` | 100 |

## Securing ZooKeeper (TLS)

For a production cluster where the ZooKeeper traffic between Managers needs encryption:

1. Generate a keystore/truststore pair (`keytool -genkeypair` → `-exportcert` → `-importcert`, PKCS12).
2. Configure `zookeeper-server.cfg` with `secureClientPort`, `ssl=true`, `sslQuorum=true`,
   `serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory`, and the keystore/
   truststore locations.
3. Configure the client side (`zoo-client.cfg`, next to `zoo.cfg` at
   `$GS_HOME/config/zookeeper/zoo-client.cfg` — introduced in 17.1.2; override its path via the
   `ZOOKEEPER_CLIENT_CONFIG_FILE` env var) with matching `secureClientPort`, `ssl=true`, and its own
   keystore/truststore.

This is enough setup work (cert generation, two config files, matching settings on every Manager)
that it's worth treating as its own task rather than a quick toggle — see GigaSpaces' ZooKeeper
configuration docs for the full property reference beyond what's captured here.

## Notes

- The Manager already bundles its own GSM and LUS — it doesn't run alongside a standalone GSM/LUS
  stack, it replaces one. A given cluster runs one topology or the other, not both at once; new
  deployments should default to the Manager over the legacy standalone stack.
- The Manager uses a different resource-selection strategy than the legacy stack when choosing
  where to deploy a PU instance, so instance distribution across GSCs may look different than
  before. Both approaches are "best-effort," but if the distribution matters, it's tunable via
  `-Dorg.jini.rio.monitor.serviceResourceSelector` (e.g.
  `=org.jini.rio.monitor.WeightedSelector`).
- A load balancer in front of the REST API is fine, but use sticky sessions — operations like
  upload/deploy take time to propagate across Managers, so a request that lands on a different
  Manager mid-operation can see stale state.

## Pitfalls index

| Symptom | Likely cause | Fix |
|---|---|---|
| Only 2 Managers configured "for HA," and the cluster doesn't stay consistent through a network partition | An even number of Managers can't guarantee quorum consensus during a partition | Use exactly 3 Managers (or accept a single standalone Manager for non-HA/dev use) |
| Tried running a 4th or 5th Manager for extra headroom | Not supported — GigaSpaces doesn't support more than 3, even though ZooKeeper itself theoretically could | Stick to 3 |
| Two Managers configured on the same host | Not supported | One Manager per host |
| A single Manager started with `--manager` and `GS_MANAGER_SERVERS` set is still unreachable from other machines | `GS_NIC_ADDRESS` wasn't set and defaulted to `hostname`, which resolves to a loopback address on this host (common Debian/Ubuntu default, e.g. `127.0.1.1`) — the Lookup Service bound to that loopback address instead of a reachable one | Set `GS_NIC_ADDRESS` explicitly to the same reachable address used for `GS_MANAGER_SERVERS` |
| Client throws `IllegalStateException: Ambiguous locators: Manager locators: [...], explicit locators: [...]` before any discovery attempt | `GS_LOOKUP_LOCATORS` is also still set alongside `GS_MANAGER_SERVERS` on that client — a fail-fast check, not a harmless redundancy | Remove `GS_LOOKUP_LOCATORS` once `GS_MANAGER_SERVERS` is configured; see `remote-proxy-connectivity`'s `locators-and-groups.md` for the client-side detail |
| `setenv-overrides` on one Manager sets custom ZooKeeper ports, but the cluster won't form | ZooKeeper requires every Manager to reach every other Manager — a port change on one Manager without matching changes/reachability on the others breaks that | Apply ZooKeeper port changes consistently across all three Managers, and confirm network reachability between them |
| A Manager set up with a custom `zoo.cfg` (via `GS_ZOOKEEPER_SERVER_CONFIG_FILE`) fails to start at all, restarting in a loop | The custom file is missing `dataDir`/`dataLogDir` — the built-in default file sets these, but a custom override file must set them explicitly, they aren't inherited | Add `dataDir`/`dataLogDir` to the custom file (e.g. under `${com.gs.work}/manager/zookeeper/`) |
| A load-balanced REST API client sees an upload/deploy appear to fail or go stale | No sticky sessions — the operation started on one Manager, a follow-up request landed on another before it finished propagating | Configure the load balancer for sticky sessions |
| PU instances land on different GSCs than expected after adopting the Manager | The Manager's resource-selector strategy differs from the legacy stack's | Not a bug; use `-Dorg.jini.rio.monitor.serviceResourceSelector` if a specific distribution is required |
| A REST client can't reach the V3 Manager API despite the (V2) REST port `8090` being reachable | V3 is a separate, newer API on its own port (`GS_REST_V3_PORT`, default `9090`) that only starts when the Manager is launched with `--restv3 --webui` — V2 on `8090` runs by default either way | Confirm which API version the client is targeting and that `--restv3 --webui` was actually passed; see `environment-variables.md`'s `GS_REST_V3_PORT` entry |
