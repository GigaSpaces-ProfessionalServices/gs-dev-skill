# GigaSpaces Environment Variables

How the GigaSpaces environment is configured, which variables are actually worth setting today, and
which ones are legacy carryovers from before the Manager existed.

## setenv / setenv-overrides

Environment configuration is maintained by `setenv` (`$GS_HOME/bin`), which every GigaSpaces script
invokes to load its configuration. **Never edit `setenv` directly** — it complicates upgrading XAP
later. Custom overrides belong in `setenv-overrides` (same directory), which `setenv` sources
automatically and which is specifically intended for this. A standalone GigaSpaces client can also
just source `setenv` directly to pick up the standard GigaSpaces libraries/classpath.

## Naming history: XAP_ prefix vs. GS_ prefix

Environment variables historically used an `XAP_` prefix (`XAP_LOOKUP_LOCATORS`, `XAP_LOOKUP_GROUPS`,
`XAP_MANAGER_SERVERS`, `XAP_NIC_ADDRESS`, `XAP_HOME`, etc.) before today's `GS_` prefix. **The current
variable names all use `GS_`** — that's the form every current script comment, example, and this
document uses, and the one to write in any new configuration.

If an old script or doc still uses an `XAP_`-prefixed variable, don't assume it's dead: GigaSpaces
resolves these by checking the `GS_` form first and falling back to the matching `XAP_` form if `GS_`
isn't set — so an old `XAP_`-prefixed setting left over from a previous setup still takes effect on
its own. The actual source of confusion is when **both** forms are set to different values at once —
`GS_` silently wins, with no warning that the `XAP_` value is being ignored. If a `GS_` setting isn't
behaving as expected, check for a leftover `XAP_`-prefixed variable with a conflicting value before
assuming the `GS_` setting itself is wrong. Simplest way to avoid this entirely: only ever set the
`GS_` form in new configuration, so there's never a `GS_`/`XAP_` pair to worry about precedence
between in the first place.

## Core variables

| Name | Description | Default |
|---|---|---|
| `JAVA_HOME` | Directory Java is installed in. | — |
| `GS_HOME` | The GigaSpaces home directory. | Auto-set via folder structure |
| `GS_LICENSE` | License key (Premium/Enterprise editions). | — |
| `GS_LOOKUP_GROUPS` | Lookup Service groups used for multicast discovery. | `xap-17.3` |
| `GS_LOOKUP_LOCATORS` | Lookup Service locators used for unicast discovery. Set this on a remote/independent client connecting to the cluster from outside it (see `GS_MANAGER_SERVERS` below for the server-side equivalent). | — |
| `GS_NIC_ADDRESS` | The network interface card GigaSpaces uses. | Omitted by default — all interfaces used |
| `GS_LOGS_CONFIG_FILE` | Location of the GigaSpaces logging configuration. | `$GS_HOME/config/log/xap_logging.properties` |

### GS_NIC_ADDRESS syntax on multi-NIC hosts

On a host with more than one network interface, `GS_NIC_ADDRESS` limits discovery to one of them.
Three forms, per the [multi-NIC configuration doc](https://docs.gigaspaces.com/latest/admin/network-multi-nic-advanced.html):

| Form | Example | Meaning |
|---|---|---|
| Explicit IP | `GS_NIC_ADDRESS=192.168.80.145` | Recommended — unambiguous, no name resolution involved. |
| Interface name, IP | `GS_NIC_ADDRESS="#eth0:ip#"` | `eth0` is the interface name; `ip` means the interface's registered IP address is used. |
| Interface name, hostname | `GS_NIC_ADDRESS="#eth0:host#"` | Same, but resolves to the interface's hostname instead of its IP. |
| Local interface | `GS_NIC_ADDRESS="#local:ip#"` | Equivalent to `InetAddress.getLocalHost().getHostAddress()`. |


## Manager & clustering

| Name | Description | Default | Notes                                                                                                                                                                                                                                                                                                                                                                                                   |
|---|---|---|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `GS_MANAGER_SERVERS` | List of Manager servers other machines connect to. | — | See `manager.md` for correct setup. Set this on the GigaSpaces server-side cluster itself (GSC/GSA/Manager fleet members) — it's the modern way to point a fleet at its Managers. A remote/independent client connecting from outside the cluster should use `GS_LOOKUP_LOCATORS` instead; don't set both on the same process (see `manager.md`'s Pitfalls index).                                     |
| `GS_REST_V3_PORT` | Port the V3 Manager API (OpenAPI 3.1.0-based management interface) listens on. | `9090` | Documented on the [REST Manager API V3 page](https://docs.gigaspaces.com/latest/admin/admin-rest-manager-api.html). Distinct from the older V2 Manager API, which listens on `8090` (`com.gs.manager.rest.port` — see `manager.md`'s ports table); V3 runs alongside V2 when the Manager is started with `--restv3`/`--webui`. `GS_RESTV3_OPTIONS` below sets this process's JVM options, not its port. |

## Process-specific Java options

Each of these appends JVM options/system properties to one specific GigaSpaces process type.

| Name | Process | Status |
|---|---|---|
| `GS_MANAGER_OPTIONS` | The Manager | Current |
| `GS_GSC_OPTIONS` | Grid Service Container (GSC) | Current |
| `GS_GSA_OPTIONS` | Grid Service Agent (GSA) | Current |
| `GS_RESTV3_OPTIONS` | The V3 REST API process | Current |
| `GS_CLI_OPTIONS` | The `gs.sh`/`gs.bat` CLI process itself | Current |
| `GS_GSM_OPTIONS` | Grid Service Manager (GSM) | **Deprecated** — silently ignored when running `./gs-agent --manager` (the recommended, modern topology). Only still meaningful on a standalone GSM/LUS stack without a Manager, which shouldn't be how a new deployment is set up — see `manager.md`. |
| `GS_LUS_OPTIONS` | Lookup Service (LUS) | **Deprecated** — same as `GS_GSM_OPTIONS`: ignored once a Manager is in play, since the Manager embeds its own LUS. |

## Broad overrides

| Name | Description |
|---|---|
| `GS_OPTIONS_EXT` | Append these options to the default GigaSpaces options. This is the one to reach for — session-wide, applies to every `gs.sh`-launched process. Prefer this over the generic `JAVA_OPTS` convention from other Java tooling — GigaSpaces' own launch scripts don't reference `JAVA_OPTS`, so `GS_OPTIONS_EXT` is the more reliable, GigaSpaces-native way to apply options consistently. |
| `GS_OPTIONS` | Override the default GigaSpaces options entirely. For GigaSpaces-engineering use — use `GS_OPTIONS_EXT` instead for normal customization. |
| `GS_LIBRARY_PATH_EXT` | Override the default library path; sets `java.library.path` to this value. |
| `GS_LIBRARY_PATH` | Append to the default library path; sets `java.library.path` to the result. |

## Pitfalls index

| Symptom | Likely cause | Fix |
|---|---|---|
| A `GS_*` variable doesn't seem to take the value that was set for it | A leftover `XAP_`-prefixed variable (the old naming) is also set, to a different value — but `GS_` silently takes priority over it | Check for the corresponding old `XAP_`-prefixed variable and remove it, rather than assuming the `GS_` setting is being ignored |
| Set `GS_GSM_OPTIONS` or `GS_LUS_OPTIONS` and the JVM option doesn't seem to apply | Both are ignored once the process is running in Manager mode (`./gs-agent --manager`) | Use `GS_MANAGER_OPTIONS` instead — the Manager replaces the standalone GSM/LUS entirely |
| Upgrading GigaSpaces breaks a customization that used to work | The customization was made directly in `setenv` instead of `setenv-overrides` | Move the override into `setenv-overrides`; never edit `setenv` itself |
| Set `JAVA_OPTS` out of habit from other Java tooling and an option doesn't seem to reach a GigaSpaces process | GigaSpaces' own launch scripts don't reference `JAVA_OPTS` | Prefer `GS_OPTIONS_EXT` — it's the mechanism GigaSpaces itself applies consistently across every `gs.sh`-launched process |
| Set `GS_REST_V3_PORT` and a REST client still connects on the wrong port, or vice versa | Two Manager API versions can run side by side: the older V2 API (`com.gs.manager.rest.port`, default `8090`) and the newer V3 API (`GS_REST_V3_PORT`, default `9090`, only active when the Manager is started with `--restv3`/`--webui`) | Confirm which API version the client is actually targeting, and that V3 is even enabled, before changing either port |
| Added an `RMI_OPTIONS`/`java.rmi.server.hostname` override expecting it to be required for `GS_NIC_ADDRESS` to take effect | `GS_NIC_ADDRESS` is sufficient on its own; `RMI_OPTIONS` isn't part of current XAP's environment scripts | Just set `GS_NIC_ADDRESS` — no companion `RMI_OPTIONS` setting is needed |
| Assumed there's no way to set JVM options (e.g. heap size) for the `gs.sh` CLI process itself | `GS_CLI_OPTIONS` exists for exactly this, it's just missing from the official environment-variables page | Set `GS_CLI_OPTIONS` (e.g. `-Xmx`/`-Xms`) the same way as `GS_GSC_OPTIONS`/`GS_GSA_OPTIONS` for other process types |
