# gs-dev-skill

A collection of Agent Skills for Claude — self-contained folders of
instructions, references, and examples that Claude loads on demand for a given task. Each skill
folder is independent: install only the ones you need.

## Skills in this repository

| Skill | Use for |
|---|---|
| [`gigaspaces-xap`](gigaspaces-xap/) | Java code generation and guidance for GigaSpaces XAP 17.3.0 |
| [`iidr-cdc-operations`](iidr-cdc-operations/) | Operational runbooks for IBM InfoSphere Data Replication (IIDR) CDC engines |

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
