# Locators, Groups, and the Lookup Finder Error

How a remote `SpaceProxy` actually resolves a space over Jini lookup discovery, why lookup groups
are usually unnecessary once locators are set, why multicast is optional and often deliberately
disabled rather than something to depend on, the trap where a bad locator can hide behind multicast
when it *is* enabled, how to read a real `CannotFindSpaceException`/`FinderException`, and the
smart-externalizable pitfall on XAP 17.3.0.

## The three pieces of information a remote proxy needs

A client needs exactly three things to connect: the **space name**, **locators** (host:port of a
lookup service, or of the space itself), and **groups** (a label the lookup service advertises
under). Of the three, groups is the one that's frequently unnecessary.

## Multicast is optional — disabling it is often the right call, not a workaround

XAP does not depend on multicast. Explicit locators (unicast) are fully sufficient on their own and
are the more predictable configuration — multicast is a convenience for zero-config discovery, not
a requirement, and nothing below should be read as XAP needing it to function.

Many environments disable multicast permanently and on purpose
(`-Dcom.gs.multicast.enabled=false` set cluster-wide), and that's usually the *correct* choice, not
a limitation to work around. With multicast on and no explicit locator, a client's default discovery
matches *any* reachable lookup service advertising the same default group — including one in a
completely unrelated environment that happens to share the same network segment (a dev client
silently attaching to a staging or prod space, or admin tooling listing spaces from another team's
grid, are both real versions of this). Turning multicast off forces every client to be explicit
about which lookup service it means, which is usually the actual goal.

**Practical implication:** don't treat "multicast isn't reaching this space" as a problem to fix —
check whether it was disabled on purpose before assuming a network issue. And regardless of whether
multicast happens to be enabled here, prefer explicit locators as the default configuration either
way; the sections below describe what happens *when* multicast is enabled and something else is
misconfigured, not an endorsement of relying on it.

## How locators/groups get configured

Three independent mechanisms exist, and more than one can be "live" in a process at once:

| Mechanism | Locators | Groups |
|---|---|---|
| Code | `new SpaceProxyConfigurer(name).lookupLocators("host:port")` | `.lookupGroups("group")` |
| System property | `-Dcom.gs.jini_lus.locators=host:port` | `-Dcom.gs.jini_lus.groups=group` |
| Env var | `GS_LOOKUP_LOCATORS=host:port` | `GS_LOOKUP_GROUPS=group` |

`GS_LOOKUP_LOCATORS`/`GS_LOOKUP_GROUPS` are read directly by the client library itself, not just
translated to `-D` by launch scripts — don't assume the env var only matters because some IDE run
config or lab script happens to forward it into a system property.

There's no confirmed precedence when code, system property, and env var are all set to *different*
values at once. If conflicting values turn up across these three sources, clear all but the one you
intend to use rather than assume which one wins — a stale env var or sysprop left over from a
previous run is a common way these conflicts happen in the first place.

`GS_MANAGER_SERVERS` is set only on the cluster's own machines (Managers, GSCs, GSAs), never on a
remote/independent client, per the sibling `environment-variables-and-manager` skill's `manager.md`.
It's worth knowing about anyway because a misconfigured fleet can leak it onto a client by accident (a
copied `setenv-overrides`, an env var inherited from the wrong host template): that variable already
carries LUS locator information for the Manager's embedded LUS. If a client this skill covers has
`GS_MANAGER_SERVERS` set at all, that's the misconfiguration to remove.

**Confirmed** against a real Manager (XAP 17.3.0, `gs.sh host run-agent --manager` with
`GS_MANAGER_SERVERS` set explicitly — the shared-server setup, not the `--auto` local-dev shortcut
that never needs `GS_MANAGER_SERVERS` set at all): setting
`GS_MANAGER_SERVERS` and `GS_LOOKUP_LOCATORS` together doesn't produce a vague "confusing discovery
behavior" or a silent preference of one over the other — `SpaceProxyConfigurer` fails immediately,
before any discovery attempt is made, with `java.lang.IllegalStateException: Ambiguous locators:
Manager locators: [...], explicit locators: [...]`. `GS_MANAGER_SERVERS` alone (no
`GS_LOOKUP_LOCATORS`) does correctly derive the Manager's embedded-LUS locator on its own, matching
`manager.md`'s finding from the GSC-log side, now independently reconfirmed from the client-side
`FinderException`.

## Groups are optional — explicit locators bypass group matching entirely

A `SpaceProxyConfigurer` with only `lookupLocators()` set connects successfully with no
`.lookupGroups()` call anywhere. It goes further than "optional": pointing `lookupLocators()` at a
real lookup service **with a deliberately wrong group** still connects. Unicast locator discovery
(`LookupLocatorDiscovery`) talks directly to the LUS at that host:port and asks it for the named
space — it doesn't filter by group at all. Group matching is purely a *multicast*-discovery
concept, used only when there's no explicit locator and the client has to find some LUS advertising
a given group over the network.

```java
// Connects even though "totally-bogus-group" matches nothing real —
// the explicit locator bypasses group filtering entirely.
GigaSpace gigaSpace = new GigaSpaceConfigurer(
    new SpaceProxyConfigurer("MySpace")
        .lookupLocators("myhost:4174")
        .lookupGroups("totally-bogus-group")
).gigaSpace();
```

**Practical implication — and a real risk.** If you're using explicit locators, a group mismatch
isn't the cause of a connectivity problem, so don't spend time debugging it. But the same fact cuts
the other way: group is not a safety net once locators are set. If a locator ever points at the
wrong environment's lookup service (a copy-pasted staging locator in a prod config, say), a
"safe-looking" mismatched group will **not** block that connection — the client connects to the
wrong space silently. Use distinct space names or genuinely separate locator values per environment
instead of relying on group for that.

## The silent-failure trap: a bad locator can hide behind multicast

This only applies where multicast is actually enabled (see above — plenty of environments turn it
off deliberately, in which case a bad locator fails immediately instead of being masked, which is
one more reason those environments prefer it that way). Where it *is* enabled: a *bad* (unreachable)
locator, with no group set, still connects successfully as long as multicast reaches the real lookup
service on the same network — `LookupLocatorDiscovery` logs a `Connection refused` warning for the
bad locator and moves on, but multicast discovery finds the space anyway in parallel.

This isn't the client falling back to matching *any* group, either — "no group set" resolves to a
real, version-derived default group (`xap-17.3.0`, for example), not a wildcard. Groups "feel"
optional mainly because that default happens to match an unconfigured space's own default group.

This means a genuinely wrong locator value can sit in a config doing nothing useful for a long time
on a network where multicast reaches the real space — masked, not fixed — and only surface as a
real failure once that same code runs somewhere multicast doesn't reach: Docker (especially without
`network_mode: host`), WAN-separated sites, most cloud VPCs. If a client works locally/on one
network and fails to find the same space elsewhere with no config change, **suspect a locator that
was never actually correct**, not a difference between the environments.

`-Dcom.gs.multicast.enabled=false` is the real switch for multicast discovery. If it isn't already
set as a standing part of this environment's configuration (see above), it's also the clean way to
test a locator honestly: rerun the client once with that flag added to its JVM args. If it still
connects, the locator is fine and something else changed; if it now fails, the locator was the
problem all along.

**A client/server XAP version mismatch does not break default discovery** — a client and server on
different XAP versions still find each other via ordinary default (no locator, no group) discovery.
An explicit locator bypasses group matching regardless of version anyway, so it's the reliable
choice either way.

## The "lookup finder" error — what it actually looks like

With a locator pointing at a port nothing is listening on *and* a group nothing advertises
(defeating both unicast and multicast discovery at once), connecting throws:

```
org.openspaces.core.space.CannotFindSpaceException: Failed to find space RemoteProxyDemoSpace
Caused by: com.j_spaces.core.client.FinderException: LookupFinder failed to find service using the following service attributes:

	 Service attributes: [net.jini.lookup.entry.Name(name=RemoteProxyDemoSpace)]
	 Service attributes: [com.j_spaces.lookup.entry.State(state=started,electable=null,replicable=null)]
	 Lookup timeout: [8000]
	 Class: com.j_spaces.core.service.Service
	 Jini Lookup Groups: [totally-bogus-group]
	 Jini Lookup Locators: [jini://localhost:4199/]
	 Number of Lookup Services: 0
```

`org.openspaces.core.space.CannotFindSpaceException` is thrown from `InternalSpaceFactory`/
`SpaceProxyFactoryBean` when building the proxy; its cause is
`com.j_spaces.core.client.FinderException`, thrown from `com.j_spaces.core.client.LookupFinder.find(...)`.
Read the exception's own dump before guessing — it states exactly what was tried:

- **`Jini Lookup Locators`** — the actual locator(s) the client tried, resolved from whichever of
  code/sysprop/env-var was in effect. Confirm this matches a host:port something is actually
  listening on (`nc -zv host port`, or the target space's own startup log for its `locator=jini://...`
  line).
- **`Jini Lookup Groups`** — the actual group(s) the client tried. Compare against the target
  space's startup log line (`Started Lookup Service [... groups=[xap-17.3.0] ...]`) — but this only
  matters for multicast discovery; if locators are also wrong, fixing the group alone won't help.
- **`Number of Lookup Services: 0`** — confirms no LUS was found via either path in the given
  timeout. If this number is nonzero, a LUS was found but didn't have the named space registered —
  a different problem (wrong space name, or the space failed to start/register).
- **`Lookup timeout`** — if discovery is generally slow on the target network, a real space might
  just need a longer `.lookupTimeout(...)` before this fires as a false negative.

## smart-externalizable — must match on both sides

`com.gs.smart-externalizable.enabled` is a serialization-optimization flag. **The rule is that it
must be set to the same value (`true` or `false`) on both the client and the server**, not that it
should be avoided entirely. A mismatch between the two sides is what causes the failure below.

The failure mode when the two sides disagree (e.g. a leftover from an older-version migration that
never got cleaned out of a launch script, JVM-args template, or env-var override on just one side)
is specifically misleading: the space is genuinely up and correctly registered — discovery
succeeds, the space shows up as running — but the client still can't get a working proxy to it. It
presents as a generic "cannot find space" symptom even though the actual cause is a
proxy-deserialization mismatch, not a discovery problem at all.

If you find `com.gs.smart-externalizable.enabled` set on one side (client launch args, server
`GS_OPTIONS_EXT`/`setenv-overrides.sh`, an old IDE run config carried forward from a prior setup),
check what the other side is actually running with — set both sides to the same value explicitly,
or remove it from both.

## Prefer SpaceProxyConfigurer over UrlSpaceConfigurer

Legacy client code sometimes builds a proxy from a wildcard Jini URL:

```java
// Legacy pattern — depends on multicast for both host and group resolution
String lookupURL = "jini://*/*/" + spaceName;
UrlSpaceConfigurer spaceConfigurer = new UrlSpaceConfigurer(lookupURL);
spaceConfigurer.lookupGroups(groups);
IJSpace space = spaceConfigurer.space();
```

Prefer `SpaceProxyConfigurer` with explicit locators instead:

```java
GigaSpace gigaSpace = new GigaSpaceConfigurer(
    new SpaceProxyConfigurer(spaceName)
        .lookupLocators(locators)   // e.g. from -Dcom.gs.jini_lus.locators or GS_LOOKUP_LOCATORS
).gigaSpace();
```

The wildcard `jini://*/*/name` form leans on multicast to resolve both "any host" and "any group" —
it inherits the silent-failure trap above, with no explicit locator to fall back on once multicast
isn't available. `SpaceProxyConfigurer` with an explicit locator gives predictable unicast discovery
that behaves the same on a network with or without multicast.

## Pitfalls index

| Symptom | Likely cause | Fix |
|---|---|---|
| Works on one machine/network, `CannotFindSpaceException` on another (Docker, WAN, cloud), no config changed | A locator was never actually correct — multicast was masking it on the working network | Pair the locator with a group nothing real advertises, or rerun with `-Dcom.gs.multicast.enabled=false`, to force a real unicast-only test; fix the actual locator value |
| `CannotFindSpaceException` / `FinderException`, `Number of Lookup Services: 0` | Neither the given locator(s) are reachable nor does any reachable LUS advertise the given group(s) | Check locator host:port reachability directly; check the target space's startup log for its real `groups=[...]`; or drop `lookupGroups()` entirely and rely on locators alone |
| `Number of Lookup Services` is nonzero but the space still isn't found | A LUS was found, but no space is registered under the exact name the client asked for | See the next row for how to confirm the real registered name |
| Client's space name looks right at a glance but still isn't found, even with locators/groups confirmed correct | A subtly wrong space name — case mismatch, a stray cluster-schema suffix (e.g. `-mirror`), or the PU name used where the space name was meant | Check the GSC log for the space's actual registered name rather than trusting the client config or a doc value; fix the client to match exactly |
| Client connects to an unexpected space despite a "wrong" group being set | Explicit locators bypass group filtering entirely — group isn't a safety net here | Use distinct space names or genuinely separate locator values per environment instead of relying on group |
| A client or discovery tool (Admin UI, `gs.sh`, an IDE plugin) unexpectedly finds/connects to a space in an unrelated environment (e.g. dev reaching staging or prod) | Multicast enabled with no explicit locator matches *any* reachable lookup service advertising the same default group, regardless of which environment it belongs to | Disable multicast for that environment (`-Dcom.gs.multicast.enabled=false`) and require explicit locators instead — this is standard practice in many shops specifically to prevent this, not just a network hardening step |
| "Cannot find space" even though the space is confirmed up and registered | `com.gs.smart-externalizable.enabled` set to different values (or set on only one side) between client and server | Set the same value (`true` or `false`) on both sides, or remove it from both |
| Legacy client fails to connect only on a network without multicast | `UrlSpaceConfigurer` with a `jini://*/*/name` wildcard URL depends on multicast | Switch to `SpaceProxyConfigurer` with explicit `lookupLocators()` |
| Client throws `IllegalStateException: Ambiguous locators: Manager locators: [...], explicit locators: [...]` before any discovery attempt | `GS_MANAGER_SERVERS` and `GS_LOOKUP_LOCATORS` are both set on the same client — a fail-fast check, not a silent preference | Remove `GS_LOOKUP_LOCATORS` — a remote/independent client should never have `GS_MANAGER_SERVERS` set in the first place, see `environment-variables-and-manager`'s `manager.md` |
| Conflicting locator/group values appear to be set in more than one place (code, `-D`, env var) | No confirmed precedence between the three sources | Clear all but the one you intend to use rather than assume which wins |
| Need to confirm whether a locator is actually correct, without multicast quietly masking a bad one | Multicast discovery runs in parallel with unicast locator discovery by default | Rerun the client once with `-Dcom.gs.multicast.enabled=false` — a genuinely bad locator will now fail instead of silently succeeding |
