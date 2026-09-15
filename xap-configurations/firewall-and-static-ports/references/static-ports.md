# Static Ports for Firewall/NAT Traversal

## Source documents

- GigaSpaces' "GigaSpaces Over a Firewall" admin-guide page (17.3.0). Its "Required Configuration
  Modifications" section edits `bin/gs.sh`, `bin/gs-ui.sh`, and `bin/setenv.sh` directly, and
  references `gsm.sh`/`gsc.sh`/`startJiniTX_Mahalo.sh` webster overrides — standalone-service
  scripts from the pre-Manager architecture. No mention of `GS_MANAGER_SERVERS`, ZooKeeper, or the
  Manager's REST/WebUI ports. Does specify: disable multicast, statically pin every listener port,
  plan a contiguous port range per IP address, don't expose the JMX/RMI-registry port.

## Per-component port plan

The firewall doc's own port table (GSM: Lookup Service + LRMI + Webster + JMX; GSC: LRMI + Webster
+ JMX; Mahalo/transaction manager: LRMI + Webster + JMX) is **stale on one specific point**: in the
modern Manager+ZooKeeper architecture, **only the Manager runs a Webster instance — a GSC does
not.** Confirmed by setting `com.gigaspaces.start.httpPort` in `GS_GSC_OPTIONS` and checking the
GSC's actual listening ports: the port never opened. The Manager's own log is explicit about who
owns it: `Created Webster on http://<manager-ip>:<httpPort>/ [roots=.../lib;.../lib/required;
.../deploy]`, and a GSC's PU-deployment log confirms it fetches the PU from the *Manager's* Webster
(`Downloading from GSM [http://<manager-ip>:<httpPort>//<pu-name>]`), not one of its own. Don't set
`httpPort` on `GS_GSC_OPTIONS` — it's a no-op there.

**Scope of that claim: confirmed for the PU-deployment path only** — both observations above are
one-directional (GSC fetching from the Manager), so they don't rule out a GSC serving content
outward in some other, untested interaction, possibly via a different, unexamined property. Treat
"a GSC doesn't run a Webster" as confirmed for PU-deployment specifically, not as a blanket
architectural fact.

Still holds from the firewall doc, and carries forward here: the LRMI transport range can be a
single value shared cluster-wide via `GS_OPTIONS_EXT` — including for components that share a host.
XAP allocates sequentially within that shared range as each additional component starts (see
"Sequential port allocation on a shared host" below), so co-located components don't need distinct
per-component ranges; what matters is sizing the range generously enough to cover every listener
every co-located component will actually open. The JMX/RMI-registry port works the same way: per
the firewall doc, it's assigned by the RMIRegistry mechanism starting from `com.gigaspaces.system.registryPort`
(default `10098`), and "each component opens the next available port" — the same sequential
fallback as Webster/LRMI, not a value that needs to be distinct per component. How far that search
extends is controlled by `com.gigaspaces.system.registryRetries` (default `20`).

**Confirmed**: a combined single-container `host run-agent --manager --gsc=1` topology sharing one
`GS_OPTIONS_EXT`-wide LRMI range produced a permanent, tight `Connection refused` retry loop
(50,000+ occurrences inside a minute — not transient startup noise, which tops out around 10-20
occurrences on an already-validated multi-Manager cluster) and outright blocked PU deployment
(`Cluster manager information has not been discovered yet`). Splitting the Manager and GSC into
separate containers, each with its own dedicated LRMI range via `GS_MANAGER_OPTIONS`/`GS_GSC_OPTIONS`,
fixed it immediately — but root cause wasn't conclusively isolated to the range being *shared* per
se: two candidate causes, an undersized range for the combined listener count of both components,
or the `host run-agent` CLI orchestrator process (also reached by `GS_OPTIONS_EXT`) contending for
the same range, weren't distinguished from each other. Don't treat this result as proof that a
shared range
is inherently unsafe for co-located components — size it generously and keep it in `GS_OPTIONS_EXT`
unless a specific range-exhaustion symptom appears.

| Purpose | Property | Where to set it |
|---|---|---|
| Multicast off, discovery port, explicit unicast discovery port | `com.gs.multicast.enabled=false`, `com.gs.multicast.discoveryPort`, `com.sun.jini.reggie.initialUnicastDiscoveryPort` | `GS_OPTIONS_EXT` — genuinely shared cluster-wide; only the Manager actually hosts the Lookup Service these ports govern, but every process needs multicast disabled consistently |
| Webster (HTTPD) | `com.gigaspaces.start.httpPort` | `GS_MANAGER_OPTIONS` **only** — the Manager is the only component that runs one; setting it in `GS_GSC_OPTIONS` has no effect (see above) |
| LRMI transport range | `com.gs.transport_protocol.lrmi.bind-port=<low>-<high>` | `GS_OPTIONS_EXT` — one shared range is sufficient, including for components on the same host; size it for the combined number of listeners every co-located component will open |
| JMX/RMI registry | `com.gigaspaces.system.registryPort` (default `10098`), `com.gigaspaces.system.registryRetries` (default `20`) | `GS_OPTIONS_EXT` — one shared starting port is sufficient; the RMIRegistry mechanism assigns each additional co-located component the next available port, up to `registryRetries` attempts |

`lrmi.bind-port` should always be set as a `<low>-<high>` range, not a single port — a component's
JVM opens multiple LRMI listeners, not just one.

## Goal this section's port list is scoped to

**State this explicitly, since it drives every "Yes"/"No" below and wasn't specified in advance:
an external client, outside the firewall, discovers and performs read/write operations against an
already-deployed space.** A different goal changes which ports need to cross — e.g. PU deployment
triggered from outside the firewall, or remote JMX-based monitoring, aren't the goal tested here
and may need different ports open. Review the goal before applying the table below to a deployment
with different requirements.

## Which ports actually need to cross the firewall, for that goal

**Confirmed against a real enforced `iptables` boundary** (not merely "we didn't publish this
port," which a directly-routable container/VLAN network can silently defeat — see the Pitfalls
index below):

| Port | Needs to cross the firewall for the stated goal? |
|---|---|
| Unicast discovery (`com.gs.multicast.discoveryPort`) | **Yes, confirmed** — this is how the external client finds the Lookup Service at all. Manager only (embeds the LUS). |
| LRMI transport range (`com.gs.transport_protocol.lrmi.bind-port`) | **Yes, confirmed, the entire range, on every component** (Manager and each GSC) — required to complete discovery itself, not only to move write/read payloads; see below |
| JMX/RMI registry (`com.gigaspaces.system.registryPort`) | **No, confirmed** — the SpaceProxy client doesn't use JMX/RMI-registry for discovery or read/write operations; direct-connecting to it from outside an enforced firewall rule that dropped everything except the two ports above was refused, and the client's operations were unaffected |
| Webster (`com.gigaspaces.start.httpPort`) | **Not confirmed either way for this goal.** Webster is confirmed used during PU deployment (a GSC's deployment log shows `Downloading from GSM [http://<manager-ip>:<httpPort>//<pu-name>]`), but that traffic was GSC→Manager, both inside the firewalled zone in this lab's topology — the deploy command itself was also triggered from inside that zone (`docker exec` into the GSC container), not by anything outside the firewall. Whether Webster needs to cross the *external* boundary depends on where deployment is triggered from, which this lab didn't test. Manager-only regardless — a GSC doesn't run its own Webster (see above). |
| REST v3 / SpaceDeck (`GS_REST_V3_PORT`, env var not a system property, default `9090`) | **Yes, confirmed, if remote REST v3/SpaceDeck access is also a goal** — a client outside an enforced firewall permitting only the discovery port, the LRMI range, and this port successfully loaded the Swagger UI, the SpaceDeck UI, and a live API call (`/api/v3/spaces`) that returned real grid data. Manager only. See below — the restv3 process's *own* LRMI range is a separate matter and does **not** belong on this list. |

**Confirmed the LRMI range is required to complete discovery, not just to move data**: with the
discovery port and Webster both left open but the LRMI range specifically blocked, a client fails
outright at proxy construction — `FinderException`, `Number of Lookup Services: 0` — despite
successfully reaching the discovery service moments earlier. Registering a found lookup service as
usable requires an LRMI callback to that service's own exported stub; reaching the discovery port
alone only gets a client to the point of knowing a lookup service exists, not to a working
reference. With the LRMI range open again, the identical client builds a working proxy, writes a
`SpaceDocument`, and reads it back successfully — confirming the range is genuinely exercised
end-to-end, not just during connection setup.

For the stated goal, firewall rules confirmed sufficient are inbound TCP on the discovery port and
the full LRMI range, for each IP address hosting a GigaSpaces component. Whether Webster also needs
to be opened depends on where PU deployment is triggered from — not established either way here.

## REST v3 and SpaceDeck

If remote REST v3 API access or SpaceDeck (the Manager's web console) is also a goal — a genuinely
different goal from the SpaceProxy client access above, since it's a separate client type — the
Manager needs `--restv3 --webui` (`host run-agent --manager --restv3 --webui`) and its own port
plan:

- **`GS_REST_V3_PORT`** (env var, not a system property; default `9090`) is the only port an
  external REST v3/SpaceDeck client needs. **Confirmed**: a client outside an enforced firewall
  permitting only the discovery port, the LRMI range, and this one port successfully loaded the
  Swagger UI, the SpaceDeck UI, and a live API call (`GET /api/v3/spaces`) that returned real grid
  data — not just a static page.
- **SpaceDeck's UI is served by the restv3 process itself, from its own root path, on the same
  port as the REST v3 API** — not a separate port or process. Confirmed by the restv3 process's own
  log (`UI serving at root path is ENABLED (GS_WEBUI_ENABLED=true)`, `SpaceDeckConfigProperties`
  loaded) and by fetching the root path directly (real HTML, not an error page). `--webui` also
  attempts to start a separate `WEBUI` GSA service, which failed in the image tested
  (`WEBUI was not created - service factory was not found`) — a known issue, not a stable
  architectural fact to plan around; SpaceDeck worked regardless, served by restv3.
- **The restv3 process is a separate JVM from the core Manager process** — `GS_MANAGER_OPTIONS`
  doesn't reach it. Confirmed: without a dedicated setting, its LRMI listener fell back to an
  unpinned port (observed as `8201`, outside any configured range). Set the LRMI range via
  `GS_OPTIONS_EXT` instead — session-wide, it reaches every `gs.sh`-launched process including
  restv3, so the same shared range already used for the Manager and GSCs covers it too, picking up
  the next available port via the same sequential-allocation behavior described above. This
  specific case (restv3 against a shared `GS_OPTIONS_EXT` range) wasn't itself tested here — only
  "no dedicated setting" vs. `GS_MANAGER_OPTIONS` were — but it follows from `GS_OPTIONS_EXT`'s
  confirmed scope.
- **That pinned range does not need to cross the external firewall for the client-access goal.**
  It's restv3's own *inbound* LRMI listener — exercised by whatever restv3 needs internally
  (plausibly self-registration with the Lookup Service), never by an HTTP/REST client, which only
  ever speaks HTTP to `GS_REST_V3_PORT`. **Confirmed**: blocking it while leaving the discovery port
  and `GS_REST_V3_PORT` open had no effect on any client-facing behavior, including the live
  data-returning API call above. Pin it anyway for consistency (avoids the unpinned fallback), but
  don't add it to the external-firewall allow-list — doing so would be exactly backwards from what a
  REST API is for: abstracting the LRMI-based admin/discovery machinery behind plain HTTP so a
  client never needs raw LRMI reachability at all.

## Sequential port allocation on a shared host

Per the firewall doc: "each additional component started on the same machine opens a sequentially
higher Webster and LRMI port, beginning from the low port in the defined range." This still holds —
one range, shared across every process on a host via `GS_OPTIONS_EXT`, is sufficient; size it for
the combined number of listeners every component on that host will actually open (a Manager process
alone typically needs fewer than a GSC hosting multiple space instances). The JMX/RMI-registry port
gets the same treatment via its own mechanism: the firewall doc separately notes it's "assigned by
the RMIRegistry mechanism... each component opens the next available port."

## Pitfalls index

| Symptom | Cause | Fix |
|---|---|---|
| The Manager's own admin/REST discovery loops forever with `Connection refused` against its own advertised address; `gs.sh service deploy` fails with `Cluster manager information has not been discovered yet` | A narrow `com.gs.transport_protocol.lrmi.bind-port` range on `GS_OPTIONS_EXT`, shared by multiple co-located components (Manager, GSC, and the CLI orchestrator) — not conclusively isolated to range size vs. the orchestrator's own transient LRMI export contending for it | Widen the shared `GS_OPTIONS_EXT` range to cover the combined listener count of every co-located component; if that alone doesn't resolve it, isolate further by giving each component its own range via `GS_MANAGER_OPTIONS`/`GS_GSC_OPTIONS` |
| "The client connects even though I didn't publish/open that port" | The underlying network (a Docker bridge subnet, a flat VLAN) is directly routable regardless of what's been explicitly published/opened — not publishing a port is not the same as firewalling it | Verify with an actually-enforced firewall/security-group rule (e.g. `iptables`) that drops traffic to the port in question, then re-test; don't treat "unpublished" as proof of "unreachable" |
| A same-host test client's connection isn't blocked by a `DOCKER-USER`-chain `iptables` rule that looks correct | `DOCKER-USER` hooks the `FORWARD` chain, which only governs traffic passing *through* the host between two other parties. A process running on the Docker host itself talking to a directly-routable container/bridge address is host-originated traffic, routed via `OUTPUT`/`POSTROUTING` — `DOCKER-USER` never sees it | Hook the same rule set to `OUTPUT` instead when the test client runs on the same host as the containers. (A real firewall between genuinely separate hosts doesn't have this distinction — all such traffic is `FORWARD`-chain traffic to it.) |
| Far-side client (genuinely outside the firewall) can't resolve a locator even though the port is open | `GS_LOOKUP_LOCATORS`/`bin/setenv.sh`'s unicast discovery port on the client side must match the *same* value configured as `com.gs.multicast.discoveryPort` on the server side | Confirm both sides use the identical port number, not just that the port is open |
| Set `com.gigaspaces.start.httpPort` in `GS_GSC_OPTIONS`, expecting a per-GSC Webster port, but the port never opens | A GSC doesn't run a Webster instance at all in this architecture — only the Manager does | Only set `httpPort` on `GS_MANAGER_OPTIONS`; don't plan a Webster port per GSC |
| Client reaches the discovery port fine (no timeout, `Connected to LUS using locator ...`) but proxy construction still fails with `FinderException`/`Number of Lookup Services: 0` | The LRMI transport range is blocked (firewall rule, security group, or simply never opened) even though the discovery port itself is open | Open the entire LRMI range, not just the discovery port — a found lookup service still needs an LRMI callback to its own exported stub to be usable, so the discovery port alone is not sufficient |
| The restv3 process's LRMI listener binds to an unpinned/unexpected port even though `GS_MANAGER_OPTIONS` sets `lrmi.bind-port` | restv3 is a separate JVM from the core Manager process — `GS_MANAGER_OPTIONS` doesn't reach it | Set `lrmi.bind-port` via `GS_OPTIONS_EXT` instead, which reaches every `gs.sh`-launched process including restv3 |
| Planning to open restv3's own LRMI range in the firewall because "REST v3 needs LRMI too" | Confusing the restv3 process's own internal LRMI usage (talking to the Manager/GSC, entirely inside the firewall) with what an external REST client needs (only `GS_REST_V3_PORT`, plain HTTP) | Don't open restv3's LRMI range externally; pin it for consistency, but only `GS_REST_V3_PORT` needs to cross the firewall |
