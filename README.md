# gs-dev-skill

A collection of Agent Skills for Claude — self-contained folders of
instructions, references, and examples that Claude loads on demand for a given task. Each skill
folder is independent: install only the ones you need.

## Skills in this repository

| Skill | Use for |
|---|---|
| [`gigaspaces-xap`](gigaspaces-xap/) | Java code generation and guidance for GigaSpaces XAP 17.3.0 |
| [`iidr-cdc-operations`](iidr-cdc-operations/) | Operational runbooks for IBM InfoSphere Data Replication (IIDR) CDC engines |
| [`xap-wan`](xap-wan/) | GigaSpaces XAP WAN Gateway: multi-site replication, bootstrapping, replication filters, conflict resolution |
| [`xap-persist`](xap-persist/) | GigaSpaces XAP external-database persistence: full Mirror service setup, initial load, custom hand-written-JDBC persistence, redolog internals |

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

## License

MIT — see [LICENSE](LICENSE).
