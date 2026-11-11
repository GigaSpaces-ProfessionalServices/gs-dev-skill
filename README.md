# gs-dev-skill

A collection of [Agent Skills](https://agentskills.io/specification) — self-contained folders of
instructions, references, and examples that an agent loads on demand for a given task. Built and
tested with Claude; because they follow the open Agent Skills format, they should also work with
GitHub Copilot, OpenAI Codex, and other compatible agents. Each skill
folder is independent: install only the ones you need.

## Skills in this repository

| Skill | Use for |
|---|---|
| [`gigaspaces-xap`](gigaspaces-xap/) | Java code generation and guidance for GigaSpaces XAP 17.3.0 |
| [`iidr-cdc-operations`](iidr-cdc-operations/) | Operational runbooks for IBM InfoSphere Data Replication (IIDR) CDC engines |
| [`xap-wan`](xap-wan/) | GigaSpaces XAP WAN Gateway: multi-site replication, bootstrapping, replication filters, conflict resolution |
| [`xap-persist`](xap-persist/) | GigaSpaces XAP external-database persistence: full Mirror service setup, initial load, custom hand-written-JDBC persistence, redolog internals |
| [`xap-upgrade`](xap-upgrade/) | Planning a GigaSpaces XAP version upgrade: a reusable pre-/during-upgrade checklist, Spring/Hibernate compatibility, version-pair notes (17.1→17.2, 17.2→17.3) |
| [`environment-variables-and-manager`](xap-configurations/environment-variables-and-manager/) | GigaSpaces XAP 17.3.0 environment variables (setenv/setenv-overrides, current vs. legacy `GS_*`), system properties, and GigaSpaces Manager setup including HA and ZooKeeper |
| [`firewall-and-static-ports`](xap-configurations/firewall-and-static-ports/) | Running GigaSpaces XAP 17.3.0 across a firewall or NAT: static ports, which ports to open, LRMI network mapping |
| [`remote-proxy-connectivity`](xap-configurations/remote-proxy-connectivity/) | Troubleshooting GigaSpaces XAP 17.3.0 remote SpaceProxy connectivity: locators/groups, `CannotFindSpaceException`, deployed-PU vs. standalone client behavior |

### gigaspaces-xap

Expert guidance for GigaSpaces XAP 17.3.0 Java development: Space POJOs, querying with SQLQuery,
event-driven processing (Polling/Notify containers), colocated task execution
(Task/DistributedTask/DurableTask), space-based remoting, custom aggregators, transactions,
Processing Unit design, Spring Boot integration, Maven setup, and OpenTelemetry distributed tracing.
See [`gigaspaces-xap/SKILL.md`](gigaspaces-xap/SKILL.md) for full details.

### iidr-cdc-operations

Field-validated operational guidance for IIDR CDC engines replicating from Oracle sources:
encrypting IIDR component-to-component traffic with TLS, configuring an IIDR Oracle CDC agent to
decrypt TDE-encrypted redo, and handling source DDL/schema changes safely. See
[`iidr-cdc-operations/SKILL.md`](iidr-cdc-operations/SKILL.md) for full details.

### xap-wan

Expert guidance for GigaSpaces XAP WAN Gateway: active-passive/active-active multi-site
replication, bootstrapping a new site from an existing one's data, selective replication filters,
cross-site conflict resolution, and adding/removing gateway targets on a live space at runtime. See
[`xap-wan/SKILL.md`](xap-wan/SKILL.md) for full details.

Distilled from [`xap-wan-training`](https://github.com/GigaSpaces-ProfessionalServices/xap-wan-training)
(branch `main`, commit `fd2bd7e`, 2026-09-01) — a Docker Compose reactor per scenario. `active-passive.md`
and `conflict-resolution.md` were verified by actually building and deploying those labs, not just
reading them; `runtime-targets.md` was distilled from `lab06-wan_gateway_modify_target` by reading
the lab's source and its own documented verification, not independently rebuilt. **To re-verify this
skill after a XAP upgrade**, rebuild and redeploy the relevant lab(s) against the new release the
same way and check for drift, rather than re-reading the source.

### xap-persist

Expert guidance for GigaSpaces XAP external-database persistence beyond the Mirror service's
exception-handling policy (owned by `gigaspaces-xap`'s `mirror-persistence.md`): full Hibernate-backed
mirror setup, initial load (including custom initial-load queries), and fully custom
hand-written-JDBC persistence. See [`xap-persist/SKILL.md`](xap-persist/SKILL.md) for full details.

Distilled from [`xap-persist-training`](https://github.com/GigaSpaces-ProfessionalServices/xap-persist-training)
(`main`, commit `570cb23`, 2026-08-20). `mirror-service.md` and `redolog.md` were verified by actually
building and deploying those labs against a real local 17.3.0 install, not just reading them —
verification also disproved one of the labs' own claims (`schema="persistent"` is not required for
`space-data-source`) and surfaced a real PU-app-name-vs-space-name bug. **To re-verify this skill
after a XAP upgrade**, rebuild and redeploy the relevant lab(s) against the new release the same way
and check for drift, rather than re-reading the source. Two labs from the same source repo were
deliberately left out — see `SKILL.md`'s intro and the commit history for why.

### xap-upgrade

General methodology for planning a GigaSpaces XAP version upgrade: a reusable checklist of
dependency-compatibility categories (including Spring/Hibernate), config formats or features that
stop working, breaking changes, and planned deprecations. Adopting new features a target version
introduces is out of scope. See [`xap-upgrade/SKILL.md`](xap-upgrade/SKILL.md) for full details.

The specific version numbers cited are confirmed only for the two jumps covered so far,
17.1 → 17.2 and 17.2 → 17.3 (one reference file each); the checklist categories are meant to
generalize to other jumps.

### environment-variables-and-manager

Guidance for configuring GigaSpaces XAP 17.3.0 environment variables and the GigaSpaces Manager:
the `setenv`/`setenv-overrides` mechanism, current vs. legacy `GS_*` variables, a narrow set of
`-D` system properties, and Manager setup from a local single-instance dev setup through a
production 3-Manager HA cluster (ports, embedded ZooKeeper configuration, ZooKeeper TLS). See
[`xap-configurations/environment-variables-and-manager/SKILL.md`](xap-configurations/environment-variables-and-manager/SKILL.md)
for full details.

Claims marked **Confirmed** in the skill were reproduced against a real 3-Manager XAP 17.3.0
cluster on Docker; unmarked claims are documented behavior not independently verified.

### firewall-and-static-ports

Guidance for making a GigaSpaces XAP 17.3.0 cluster reachable across a firewall or NAT boundary:
disabling multicast, statically pinning every listener port, which ports actually need to cross
the boundary, and LRMI network mapping for NAT. The port guidance is scoped to one goal: an
external client discovering and reading/writing an already-deployed space. See
[`xap-configurations/firewall-and-static-ports/SKILL.md`](xap-configurations/firewall-and-static-ports/SKILL.md)
for full details.

Claims marked **Confirmed** in the skill were reproduced against a real two-container (Manager +
GSC) XAP 17.3.0 cluster on Docker with an `iptables` rule enforcing the port boundary.

### remote-proxy-connectivity

Troubleshooting guidance for a client that can't get a working `SpaceProxy` to a remote XAP 17.3.0
space: a triage list to work through first, locators/groups discovery (`GS_LOOKUP_LOCATORS`,
`GS_LOOKUP_GROUPS` and their `-D` equivalents), reading `CannotFindSpaceException` /
`FinderException`, the 17.3.0 smart-externalizable pitfall, and how a proxy created inside a
deployed PU behaves differently from a standalone client. See
[`xap-configurations/remote-proxy-connectivity/SKILL.md`](xap-configurations/remote-proxy-connectivity/SKILL.md)
for full details.

Verified in a lab against a standalone XAP 17.3.0 space and a local grid.

## Installation

Each skill folder (e.g. `gigaspaces-xap/`) is a complete, standalone skill — its `SKILL.md` and
`references/` are all it needs. Skill folders intentionally don't contain their own README; this
file is the entry point for browsing the repo.

### Claude.ai

1. Clone this repo, or download it as a ZIP and extract it.
2. Zip the specific skill folder you want, from inside the repo root so the zip's top level is the
   skill folder itself (not the whole repo):
   ```bash
   zip -r gigaspaces-xap.zip gigaspaces-xap
   ```
3. In Claude.ai: **Settings > Capabilities > Skills > Upload skill**, and select the zip.
4. Test it — ask Claude something the skill should trigger on, e.g. *"Write a GigaSpaces `@SpaceClass`
   POJO with a routing field"* or *"How do I set `mirror_on_ddl_operation` on an IIDR Oracle CDC
   subscription?"*

### Claude Code

1. Clone this repo.
2. Copy (or symlink) the skill folder you want into your Claude Code skills directory (see the
   Claude Code documentation for the current path for your setup — user-level or project-level).
3. Test it the same way as above.

### Other agents (GitHub Copilot, OpenAI Codex, and others)

These skills use the open [Agent Skills](https://agentskills.io/specification) format (`SKILL.md` +
`references/`), so agents other than Claude should be able to use them as they are, with no changes. Copy (or
symlink) each skill folder you want into a folder your agent scans:

| Agent | Repo-level (commit with your project) | User-level (all your projects) |
|---|---|---|
| GitHub Copilot ([docs](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)) | `.github/skills/`, `.claude/skills/`, or `.agents/skills/` | `~/.copilot/skills/` or `~/.agents/skills/` |
| OpenAI Codex ([docs](https://developers.openai.com/codex/skills)) | `.agents/skills/` | `~/.agents/skills/` |

`.agents/skills/` works for both, so use it if your team uses more than one agent. For any other
agent that supports Agent Skills, check its documentation for its skills folder.

```bash
mkdir -p ~/.agents/skills
cp -r gigaspaces-xap ~/.agents/skills/
```

**Notes:**

- **Install each skill as its own top-level folder.** The `xap-configurations/` folder only groups
  related skills and isn't a skill itself. Copy each skill inside it separately:
  ```bash
  cp -r xap-configurations/firewall-and-static-ports ~/.agents/skills/
  ```
- **Some skills refer to another skill by name.** For example, `xap-persist` points to
  `gigaspaces-xap`'s `mirror-persistence.md`, and `xap-upgrade` points to `xap-persist`. Install
  both for those references to work.
- **Test that it triggers.** Each agent decides when to load a skill from its `description`, and
  how reliably that happens depends on the model. Try one of the test prompts above in your agent
  to confirm the skill loads.

## License

MIT — see [LICENSE](LICENSE).
