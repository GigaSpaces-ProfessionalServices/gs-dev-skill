---
name: xap-persist
description: >
  Expert guidance for GigaSpaces XAP external-database persistence beyond the Mirror service basics:
  Hibernate-backed mirror setup end-to-end, initial load (including custom initial-load queries),
  and fully custom (hand-written JDBC) persistence. Use when the user mentions initial load,
  SpaceInitialLoadQuery, SpaceDataSource, SpaceSynchronizationEndpoint, ManagedEntriesSpaceDataSource,
  hibernateSpaceDataSource, redolog, redo log, flushRedoLogToStorage, SqliteRedoLogFileStorage, or
  persisting/loading space data to-or-from an external database. For the mirror's exception-handling
  policy specifically (PersistencyExceptionHandler, retry/dead-letter), see gigaspaces-xap's
  mirror-persistence.md first — this skill assumes that decision is already made and covers what
  surrounds it. Default XAP target version is 17.3.0 unless the user specifies otherwise.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# GigaSpaces XAP Persistence Skill

Covers getting data into and out of an external store around a XAP space: the Mirror service's full
Hibernate/Spring setup (not just its exception handling), initial load, fully custom
hand-written-JDBC persistence, and redolog internals.

**Not currently covered**: NoSQL `SpaceDocument` persistence via MongoDB. The training lab it would
be distilled from pins a stale MongoDB Java driver (`mongo-java-driver` 3.11.2, ~2019) against a
current `mongo:7` server, and it's unverified whether GigaSpaces' `xap-mongodb` artifact itself
requires that exact driver version — held back until that's actually researched, not just
version-bumped and assumed to work.

**Default target version: XAP 17.3.0**

---

## Scope boundary with `gigaspaces-xap`

`gigaspaces-xap`'s `mirror-persistence.md` already owns the mirror's **exception-handling policy**
question — `PersistencyExceptionHandler`, the bounded-retry/dead-letter boilerplate, and the
alternatives table for picking a policy. Read that file first if the question is "what happens when
a persistence write fails." This skill's `mirror-service.md` assumes that decision is already made
and covers everything around it instead: the actual Hibernate `SessionFactory`/`dataSource` wiring,
`mirrored="true"`, deploy/verify, and the dependency-upgrade gotchas that surfaced migrating this
lab to XAP 17.3.0 / Spring 7 / Hibernate 7. Don't duplicate the exception-handler content here if
extending this skill later — point back to `gigaspaces-xap` instead.

---

## Reference Files

**MANDATORY**: Read the relevant reference file(s) below **before generating any code or answering
any question** in this domain. Select by scenario, then read using paths relative to this skill's
own directory (`references/<file>.md`).

| Reference Path | Covers |
|------|--------|
| `references/mirror-service.md` | Full Mirror service setup: Hibernate `SessionFactory`, `dataSource`, `mirrored="true"`, `<os-core:mirror>`, deploy/verify, Spring 7/Hibernate 7 migration gotchas |
| `references/initial-load.md` | Loading existing DB data into a space at deploy time: `hibernateSpaceDataSource`, `@SpaceInitialLoadQuery` for partial/custom loads, the assembly.xml dependency-allowlist trap |
| `references/custom-persistence.md` | Fully hand-written persistence (no Hibernate): custom `SpaceSynchronizationEndpoint`/`ManagedEntriesSpaceDataSource`, raw JDBC DAOs, cross-database JDBC porting gotchas (worked example: Postgres) |
| `references/redolog.md` | Redolog internals: SQLite-backed storage, flushing to disk, replaying — **relies on unsupported internal GigaSpaces classes, not public API; read the warning at the top before using this in anything beyond a one-off diagnostic** |

## Quick Decision Guide

```
User wants to...
  ├── Wire up a working Hibernate-backed mirror from scratch    → mirror-service.md
  │   (for "what happens when a write fails" specifically       → gigaspaces-xap's mirror-persistence.md)
  ├── Load existing DB data into a space at deploy time          → initial-load.md
  ├── Persist to a DB without Hibernate (hand-written JDBC)      → custom-persistence.md
  └── Inspect, flush, or replay the pending-replication redolog  → redolog.md
```

---

## Cross-Cutting Notes

These recur across multiple reference files in this skill — apply them regardless of which one
you're following:

1. **Porting a Hibernate mapping or hand-written SQL to a different target database reliably surfaces
   the same three categories of problem, whatever the actual database is**: reserved-word collisions
   with your own identifiers, type mappings that don't carry over 1:1, and JDBC driver strictness
   differences (what one driver tolerates, another enforces per spec). Every lab this skill is
   distilled from happened to target PostgreSQL, so `mirror-service.md` and `custom-persistence.md`
   show the concrete Postgres-specific shape these took (e.g. `User` being reserved, `int(11)` having
   no direct equivalent) — treat those as **a worked example of the pattern, not a checklist that
   transfers directly to Oracle, SQL Server, or any other target.** Expect to do the equivalent
   research (reserved-word list, type mapping table, driver quirks) for whatever database you're
   actually porting to.
2. **`spring-orm` is not on a PU's classpath by default** in this XAP distribution — it ships under
   `lib/optional/`, not the GSC's default classpath. Any PU using Hibernate directly needs to
   redeclare it `compile`-scope (not `provided`) so the assembly plugin bundles it into the PU's own
   `lib/`.
3. **`org.springframework.orm.hibernate5.LocalSessionFactoryBean` no longer exists** — Spring
   Framework 7 removed that package. Use `org.springframework.orm.jpa.hibernate.LocalSessionFactoryBean`
   instead; same property setters (`dataSource`, `packagesToScan`, `hibernateProperties`).
4. **A PU with `mirrored="true"` blocks on undeploy if its mirror is already gone.** Undeploy drains
   pending replication to the mirror first; if the mirror was undeployed already, that drain can
   never complete and blocks for the full timeout. Undeploy the space *before* the mirror, or pass
   `--drain-mode=NONE` to skip the drain.
5. **A space with initial load doesn't finish loading the instant `pu deploy` returns.** Initial load
   runs as part of space startup and can take several seconds; querying immediately after deploy can
   see 0 objects, or "space type descriptor not found" for a type with no data yet (types register
   lazily on first object of that type).
6. **The PU/app-name (what you pass to `pu deploy`/`pu list`/`pu undeploy`) is not always the same
   string as the space's own name (what every `space` subcommand needs)** — confirmed live, and this
   skill's own examples deliberately use visibly distinct names (`space-pu` vs. `my-space`,
   `my-app-space-pu` vs. `redolog` in `redolog.md`) specifically so the two are never confusable at a
   glance. Watch for this in your own project too even when the two names merely differ in casing
   (e.g. `MyApp-Space` the app-name vs. `MyApp-space` the space name) — that's easy to misread as the
   same string and was a real, confirmed source of `404 Not Found` errors while verifying this skill.
   Confirm the actual space name via `pu list`'s `SPACE` column rather than assuming it matches the
   app-name you deployed under.

---

## Imports Cheat Sheet

```java
// Mirror / Hibernate persistence
import org.springframework.orm.jpa.hibernate.LocalSessionFactoryBean; // NOT orm.hibernate5 (removed in Spring 7)
import org.openspaces.persistency.hibernate.DefaultHibernateSpaceSynchronizationEndpointFactoryBean;
import org.openspaces.persistency.patterns.SpaceSynchronizationEndpointExceptionHandler; // see gigaspaces-xap's mirror-persistence.md
import org.openspaces.persistency.patterns.PersistencyExceptionHandler;                 // see gigaspaces-xap's mirror-persistence.md

// Initial load
import com.gigaspaces.datasource.SpaceDataSource;
import com.gigaspaces.datasource.DataIterator;
import com.gigaspaces.annotation.pojo.SpaceInitialLoadQuery;

// Custom (hand-written) persistence
import com.gigaspaces.sync.SpaceSynchronizationEndpoint;
import com.gigaspaces.sync.DataSyncOperation;
import com.gigaspaces.sync.OperationsBatchData;
import org.openspaces.persistency.patterns.ManagedEntriesSpaceSynchronizationEndpoint;
import org.openspaces.persistency.patterns.ManagedEntriesSpaceDataSource;

// Redolog (see redolog.md's warning about internal-API usage before reaching for these)
import com.gigaspaces.internal.server.space.redolog.storage.SqliteRedoLogFileStorage;
import com.gigaspaces.internal.cluster.node.impl.packets.IReplicationOrderedPacket;
import com.j_spaces.core.admin.IInternalRemoteJSpaceAdmin;
```
