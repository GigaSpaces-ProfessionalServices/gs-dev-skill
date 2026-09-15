---
name: environment-variables-and-manager
description: >
  Guidance for configuring GigaSpaces XAP 17.3.0 environment variables and the GigaSpaces Manager.
  Covers the setenv/setenv-overrides mechanism, which GS_* environment variables are current vs.
  legacy (GS_GSM_OPTIONS/GS_LUS_OPTIONS are ignored once a Manager is in play), and how to
  properly set up the GigaSpaces Manager: local single-Manager development, a production 3-Manager
  HA cluster via GS_MANAGER_SERVERS, ports, the embedded ZooKeeper's configuration (zoo.cfg,
  GS_ZOOKEEPER_SERVER_CONFIG_FILE, leader-election tuning), and securing
  ZooKeeper with TLS. Use when the user mentions setenv, setenv-overrides, GS_HOME,
  GS_LOOKUP_GROUPS, GS_LOOKUP_LOCATORS, GS_NIC_ADDRESS, GS_MANAGER_SERVERS, GS_MANAGER_OPTIONS,
  GS_GSC_OPTIONS, GS_GSM_OPTIONS, GS_GSA_OPTIONS, GS_LUS_OPTIONS, GS_RESTV3_OPTIONS, GS_REST_V3_PORT,
  GS_CLI_OPTIONS,
  GS_OPTIONS_EXT,
  GS_OPTIONS, GS_LIBRARY_PATH, GS_ZOOKEEPER_SERVER_CONFIG_FILE, XAP_LOOKUP_LOCATORS, XAP_NIC_ADDRESS,
  XAP_HOME, legacy XAP_ environment variable prefix, DurableTask management via REST,
  GigaSpaces Manager, gs-agent --manager, high availability / HA cluster setup, ZooKeeper / zoo.cfg /
  zookeeper-server.cfg / zoo-client.cfg, GSM leader election, com.gs.home, com.gs.deploy, com.gs.work,
  com.gs.pu-common, com.gigaspaces.lib.platform.ext, com.gs.multicast.enabled,
  java.util.logging.config.file, com.gs.jini_lus.locators, com.gs.jini_lus.groups, GigaSpaces system
  properties, or GigaSpaces environment variable configuration in general.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# Environment Variables & Manager — Configuration Skill

Guidance for configuring the GigaSpaces environment correctly: which environment variables are
current vs. legacy, which are niche enough that they shouldn't be reached for casually, and how to
set up the GigaSpaces Manager properly — from a local single-instance dev setup through a
production 3-node HA cluster. Where a claim is marked **Confirmed**, it's been reproduced against a
real 3-Manager XAP 17.3.0 cluster on Docker, not just read off the docs — where it isn't marked that
way, it's documented behavior that hasn't been independently verified by this skill yet.

**Default target version: XAP 17.3.0.**

**This skill covers server/grid-side setup — for diagnosing why a *client* can't connect to a
space**, use the sibling `remote-proxy-connectivity` skill instead. The two overlap at exactly one
point: `GS_MANAGER_SERVERS` (this skill) and `GS_LOOKUP_LOCATORS`/`GS_LOOKUP_GROUPS` (that skill)
aren't meant to both be set — see `references/manager.md`'s Pitfalls index and that skill's
`locators-and-groups.md` for each side of that conflict.

**MANDATORY**: Read the relevant reference file(s) below **before answering any environment or
Manager configuration question**, using paths relative to this skill's own directory
(`references/<file>.md`).

## Reference Files

| Reference Path | Covers |
|---|---|
| `references/environment-variables.md` | The `setenv`/`setenv-overrides` mechanism, the legacy `XAP_` variable-name prefix vs. today's `GS_` prefix, the core `GS_*` variables, `GS_NIC_ADDRESS` syntax on multi-NIC hosts, which process-options variables (`GS_GSM_OPTIONS`, `GS_LUS_OPTIONS`) are deprecated now that the Manager exists, and the V2/V3 Manager API port split (`GS_REST_V3_PORT`) |
| `references/manager.md` | Setting up the GigaSpaces Manager correctly: local single-Manager dev use (`--auto`, local-only), a shared non-HA single instance reachable from other machines (`--manager --restv3 --webui` + `GS_MANAGER_SERVERS`/`GS_NIC_ADDRESS`), a production 3-Manager HA cluster via `GS_MANAGER_SERVERS`, ports, ZooKeeper configuration (including the `GS_ZOOKEEPER_SERVER_CONFIG_FILE` override and its `dataDir`/`dataLogDir` pitfall) and leader-election tuning, securing ZooKeeper with TLS |
| `references/system-properties.md` | A narrow set of `-D` system properties: paths/classloading (`com.gs.home`, `com.gs.deploy`, `com.gs.work`, `com.gs.pu-common` vs. `com.gigaspaces.lib.platform.ext`), discovery system-property alternatives to `GS_LOOKUP_LOCATORS`/`GS_LOOKUP_GROUPS`/multicast (`com.gs.jini_lus.locators`, `com.gs.jini_lus.groups`, `com.gs.multicast.enabled`), and logging (`java.util.logging.config.file`) |

## Quick Decision Guide

```
User's question is about...
  ├── A specific GS_* variable's purpose/default, or whether it's still relevant → environment-variables.md
  ├── Customizing the environment safely (setenv vs. setenv-overrides)           → environment-variables.md
  ├── Setting up the Manager (dev or production HA)                              → manager.md
  ├── GS_MANAGER_SERVERS syntax, or a conflict with GS_LOOKUP_LOCATORS           → manager.md
  ├── Manager/ZooKeeper ports, or ZooKeeper leader-election tuning               → manager.md
  ├── Whether GS_GSM_OPTIONS/GS_LUS_OPTIONS still work                           → environment-variables.md (they don't, under a Manager)
  ├── A -D system property (paths/classloading, discovery, logging)              → system-properties.md
  └── A client can't find/connect to a space at all                              → remote-proxy-connectivity skill instead (check manager.md's Pitfalls index first if GS_MANAGER_SERVERS is also set)
```

## Never edit setenv directly

Environment configuration lives in `setenv` (`$GS_HOME/bin`), invoked by every GigaSpaces script.
**Always put customizations in `setenv-overrides`** (same directory) instead — `setenv` calls it
automatically, and editing `setenv` itself complicates future upgrades. See
`references/environment-variables.md` for the full variable reference.

## The Manager, not the legacy GSM/LUS stack

New deployments should use the GigaSpaces Manager (`./gs.sh host run-agent --auto`), not a
standalone GSM/LUS pair. The Manager already bundles its own GSM and LUS, so it replaces the
standalone pair rather than running alongside it — a given cluster is one topology or the other, not
both. `GS_GSM_OPTIONS`/`GS_LUS_OPTIONS` are silently ignored once a process runs in Manager mode — see
`references/manager.md` for correct setup (a 3-Manager cluster for production HA, exactly one
Manager per host) and `references/environment-variables.md` for the variable-level detail.

## Troubleshooting

All three reference files end with their own **Pitfalls index** (symptom → cause → fix). Check the
table in the relevant file before diagnosing.
