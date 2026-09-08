---
name: remote-proxy-connectivity
description: >
  Field-validated guidance for troubleshooting GigaSpaces XAP 17.3.0 remote SpaceProxy connectivity
  - the client-side connection to a running space over locators/groups discovery. Covers
  SpaceProxyConfigurer vs UrlSpaceConfigurer, how GS_LOOKUP_LOCATORS/GS_LOOKUP_GROUPS env vars and
  -Dcom.gs.jini_lus.locators/-Dcom.gs.jini_lus.groups system properties actually behave, why lookup
  groups are often unnecessary once locators are set, the real CannotFindSpaceException /
  FinderException ("lookup finder" error) and how to read it, the smart-externalizable pitfall on
  17.3.0, and the DIFFERENT default connectivity behavior of a remote proxy created by code running
  inside a Processing Unit deployed to the grid (GSC) versus an independent standalone client. Use
  when the user mentions a proxy that can't find/connect to a space, CannotFindSpaceException,
  FinderException, LookupFinder, "cannot find space", lookup timeout, GS_LOOKUP_LOCATORS,
  GS_LOOKUP_GROUPS, com.gs.jini_lus.locators, com.gs.jini_lus.groups, lookup groups, lookup locators,
  smart-externalizable, UrlSpaceConfigurer, SpaceProxyConfigurer connectivity problems in general, or
  why a deployed PU's space-proxy behaves differently than a standalone client / gs.sh pu run.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# Remote Proxy Connectivity — Troubleshooting Skill

Guidance for diagnosing why a client can't get a working `SpaceProxy` to a remote XAP space —
field-validated against a real standalone XAP 17.3.0 space and a real local grid, not guessed from
docs.

**Default target version: XAP 17.3.0.** The smart-externalizable pitfall is version-specific —
don't carry it forward to a different version pair without re-verifying.

## Triage first — not every "can't connect" is a discovery-layer problem

The reference files below cover the locators/groups/discovery layer in depth, but that's rarely
the first thing to check. Work through this list before assuming the cause is anything covered in
this skill — most of it is cheaper to rule out than a discovery-layer misconfiguration, and several
of these produce a symptom that's indistinguishable from one at the client:

1. **Get the actual exception/stack trace, and its reproducibility** — always or intermittent, one
   client or all of them, one target space or all. "It's slow" or "it won't connect" isn't enough to
   start from.
2. **Read the server/manager's own logs for what it's actually advertising**, before touching the
   client at all. The manager/space startup log states the real locator and group(s) directly (e.g.
   `Started Lookup Service [... groups=[xap-17.3.0] ...]`) — that's ground truth for what the client
   *should* be using, not an assumption or a value copied from a config file that might itself be
   stale. Comparing the client's attempted locators/groups against this is the actual diagnosis;
   guessing at the server side or trusting a doc/config value nobody re-checked is how time gets
   wasted chasing the wrong fix.
3. **Check the GSC log for evidence the space actually deployed successfully, and confirm the exact
   space name while you're there.** A GSC that never finished starting the space (a failed
   initializer, a missing dependency, a schema/partition count that never converged) never registers
   it at all, no matter how correct the client's locators/groups are — and this is also the fastest
   way to catch a client using a space name that's subtly wrong (case, a stray `-mirror`/cluster-
   schema suffix, PU name confused with space name). `Number of Lookup Services` being nonzero in
   the client's `FinderException` but the space still not found is exactly this: a real lookup
   service was reached, it just has no space registered under the name the client asked for.
4. **Ask what changed.** A deploy, a XAP or JDK upgrade, a firewall/security-group change, a
   certificate rotation, a new container/orchestration policy. Sudden connectivity problems
   correlate with a recent change far more often than with a latent misconfiguration.
5. **Confirm plain network reachability** — `nc -zv host port` (or equivalent) from the client host,
   before touching anything XAP-specific.
6. **Check whether multicast is even expected to be enabled here — don't assume it should be.** XAP
   doesn't depend on multicast; explicit locators are fully sufficient on their own and are the
   recommended configuration regardless. Many environments disable multicast permanently and on
   purpose, specifically to stop clients or discovery tooling from picking up spaces in unrelated
   environments that happen to share the same network (see `references/locators-and-groups.md`).
   Treat its absence as the likely intended state, not a network problem to fix, and confirm explicit
   locators are configured instead of chasing multicast connectivity.
7. **Confirm the target space's cluster health**, beyond the "did it deploy" check in step 3 — a
   partition stuck without a primary (split-brain/quorum loss) presents as "client can't connect"
   from the outside just as much as a failed deploy does, and no client-side config change fixes it.
   Check via the Admin UI/API or `gs.sh`.
8. **Check client/server version and classpath parity**, especially after a rolling upgrade where
   not every node landed on the new version yet.
9. **Rule out GC or thread-pool starvation on either side.** A GC pause long enough to blow past the
   lookup timeout looks identical to a genuine discovery failure. Check GC logs/`jstat` before
   assuming it's a config problem.
10. **If there's an auth layer** (Spring Security, LDAP/AD, a custom `LoginModule`, TLS on the
    space), rule it in or out early — an auth failure can surface as a generic connection failure
    rather than an obvious "access denied."
11. **Reproduce with the smallest possible client** — a bare `SpaceProxyConfigurer` call or a
    `gs.sh` CLI operation against the space directly, stripped of the application and any wrapper
    framework. If the minimal repro connects but the app doesn't, the problem is in the app's config
    layer, not XAP or the network.
12. **Get logs from both sides.** The client stack trace shows what the client tried; the
    server/GSC/manager logs (already opened in steps 2-3) show what the server actually saw and
    advertised. Diagnosing from one side alone is guessing with one eye closed.

Only once the basics above are ruled out — or point specifically at locators, groups, or discovery
— does it make sense to dig into the reference files below.

**MANDATORY**: Read the relevant reference file(s) below **before answering any connectivity
question**, using paths relative to this skill's own directory (`references/<file>.md`).

## Reference Files

| Reference Path | Covers |
|---|---|
| `references/locators-and-groups.md` | How locators/groups are configured and resolved, why groups are usually unnecessary once locators are set, why multicast is optional and often deliberately disabled, the silent-failure trap where multicast masks a bad locator (when multicast *is* enabled), reading a real `CannotFindSpaceException`/`FinderException`, the smart-externalizable pitfall, `SpaceProxyConfigurer` vs `UrlSpaceConfigurer` |
| `references/grid-vs-independent-clients.md` | Why a proxy created inside a PU deployed to a grid connects with zero locators/groups configured, and why the identical code fails standalone or via `gs.sh pu run` |

## Quick Decision Guide

```
User's proxy can't find/connect to a space...
  ├── Haven't ruled out network/target-health/version/GC/auth yet     → Triage section above, first
  ├── Works on one machine/network, fails on another (Docker/WAN/cloud), no config change → locators-and-groups.md
  ├── CannotFindSpaceException / FinderException thrown                                    → locators-and-groups.md
  ├── Space confirmed up + registered but client still can't connect                        → locators-and-groups.md (smart-externalizable)
  ├── Legacy code uses a wildcard jini://*/*/name URL                                       → locators-and-groups.md (SpaceProxyConfigurer)
  ├── Works deployed to the grid, fails standalone / under gs.sh pu run                     → grid-vs-independent-clients.md
  └── Conflicting locator/group values across code, -D, and env var                        → locators-and-groups.md (no confirmed precedence)
```

## The three pieces of information a remote proxy needs

To connect to a remote space, a client needs exactly three things: the **space name**, **locators**
(host:port of a lookup service, or of the space itself), and **groups** (a label the lookup service
advertises under). Of the three, groups is the one that's frequently unnecessary — see
`references/locators-and-groups.md`.

| Mechanism | Locators | Groups |
|---|---|---|
| Code | `new SpaceProxyConfigurer(name).lookupLocators("host:port")` | `.lookupGroups("group")` |
| System property | `-Dcom.gs.jini_lus.locators=host:port` | `-Dcom.gs.jini_lus.groups=group` |
| Env var | `GS_LOOKUP_LOCATORS=host:port` | `GS_LOOKUP_GROUPS=group` |

All three can be "live" in a process at once, with no confirmed precedence between them — see the
reference file before assuming which one wins.

## Troubleshooting

Both reference files end with their own **Pitfalls index** (symptom → cause → fix). Check the table
in the relevant file before improvising a diagnosis — most failure modes here (a locator masked by
multicast, multicast bleeding across environments, a wrong space name, a deployed-vs-standalone
discrepancy, smart-externalizable) have a known, non-obvious cause already documented there.

## Imports / Class Cheat Sheet

```java
import org.openspaces.core.space.SpaceProxyConfigurer;
import org.openspaces.core.space.UrlSpaceConfigurer;      // legacy — prefer SpaceProxyConfigurer
import org.openspaces.core.GigaSpaceConfigurer;
import org.openspaces.core.GigaSpace;
import org.openspaces.core.space.CannotFindSpaceException;
import com.j_spaces.core.client.FinderException;
```
