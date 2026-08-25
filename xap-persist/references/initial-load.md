# Initial Load: Loading Existing DB Data Into a Space

Populating a space from an existing database at deploy time — either the entire mapped dataset, or
a filtered subset via a custom load query. Builds on `mirror-service.md`'s Hibernate setup; read
that first if the `sessionFactory`/`dataSource` wiring isn't already in place.

## Basic initial load: `space-data-source`

```xml
<bean id="hibernateSpaceDataSource"
      class="org.openspaces.persistency.hibernate.DefaultHibernateSpaceDataSourceFactoryBean">
    <property name="sessionFactory" ref="sessionFactory"/>
    <property name="initialLoadChunkSize" value="2000"/>
</bean>

<os-core:embedded-space id="space" space-name="my-space"
                         space-data-source="hibernateSpaceDataSource"/>
```

Initial load is independent of `mirrored="true"` — a space can load from a `SpaceDataSource` at
startup without also write-behind-persisting back through a mirror, and vice versa; don't assume
one implies the other. `initialLoadChunkSize` batches the load instead of pulling the whole dataset
into memory at once.

**`schema="persistent"` is not required for this and is not the default** (the actual default is
`schema="default"`, confirmed against `SpaceURL.DEFAULT_SCHEMA_NAME`) — confirmed live: initial load
via `space-data-source` works identically with `schema="default"` or no `schema` attribute at all.

Most likely explanation: `persistent` is a named schema preset
(`config/schemas/persistent-space-schema.xml`, bundled in `xap-datagrid.jar` alongside
`default`/`mirror`/`cache`/`javaspace`) that pre-wires an **older, XML-embedded**
`<external-data-source>` block — `enabled=true` plus a `data-source-class` — as a holdover from a
persistence mechanism that predates the modern Spring-bean-based `space-data-source` PU attribute
these labs actually use. The template's own comment even names the legacy fallback class
(`com.gigaspaces.datasource.hibernate.HibernateDataSource`, an old `com.gigaspaces.datasource.*`
core package) before overriding it to the newer `org.openspaces.persistency.hibernate.*` one. It
also happens to bundle different cache/connection-pool tuning defaults suited to a DB-backed space
(e.g. `cache_size` 50,000→100,000, `max_sa_connections` 20→500). Either way — legacy convention
carried forward, or deliberate tuning choice — it's not a functional requirement for the modern
`space-data-source` attribute to work.

On deploy, XAP calls this data source to load every row of every mapped entity into the space
before the PU is considered up. Verify with:

```bash
# Note: the deployed PU's app-name (used with `pu deploy`/`pu list`) is not the space's own
# name (used with every `space` subcommand below) — confirm the real space name via `pu list`'s
# SPACE column rather than assuming it matches the app-name. See SKILL.md's cross-cutting notes.
./gs.sh space info --type-stats my-space   # object count per data type
```

## Custom (partial) initial load: `@SpaceInitialLoadQuery`

To load only a subset — e.g. only recent or high-value records — annotate a method on the space
class instead of changing the data source bean:

```java
@SpaceClass
public class Payment implements Serializable {
    // ... other fields/getters ...

    @SpaceInitialLoadQuery
    public String initialLoadQuery() {
        return "paymentAmount > 50"; // a WHERE-clause fragment, not a full query
    }
}
```

Then tell the data source bean where to find annotated classes:

```xml
<bean id="hibernateSpaceDataSource"
      class="org.openspaces.persistency.hibernate.DefaultHibernateSpaceDataSourceFactoryBean">
    <property name="sessionFactory" ref="sessionFactory"/>
    <property name="initialLoadQueryScanningBasePackages">
        <list>
            <value>com.example.model</value>
        </list>
    </property>
</bean>
```

Without `initialLoadQueryScanningBasePackages` naming the package, XAP never scans for
`@SpaceInitialLoadQuery` methods at all and silently falls back to loading everything — this isn't
an error, so a missing/wrong package here reads as "the filter didn't do anything" rather than a
startup failure.

## Verify a custom load actually filtered

```bash
# Every result should satisfy the filter; the inverse filter should return nothing
./gs.sh space query --filter="paymentAmount > 50"  --max-results=10 my-space com.example.model.Payment
./gs.sh space query --filter="paymentAmount <= 50" --max-results=10 my-space com.example.model.Payment  # expect 0 rows

# Confirm both partitions actually got their share (routing still applies during initial load)
./gs.sh space list-instances my-space
./gs.sh space info-instance --type-stats my-space~1_1
./gs.sh space info-instance --type-stats my-space~2_1
```

Both partition primaries should show a non-zero count for the loaded type — initial load doesn't
bypass routing, so a query filtered narrowly enough to only match rows that happen to route to one
partition isn't itself a bug, but seeing literally zero on every partition but one for a filter that
should be broad is worth double-checking against the actual data distribution before assuming
initial load itself is broken.

## The assembly.xml dependency-allowlist trap

If the space/data-source module's `assembly.xml` uses an explicit dependency **allowlist**
(`<includes>...</includes>`) rather than an exclude-based one, renaming or upgrading a dependency's
GroupId/ArtifactId (e.g. swapping a JDBC driver, or Hibernate's own groupId — `org.hibernate` →
`org.hibernate.orm` for Hibernate 6+) silently drops that jar from the bundled PU. `mvn install`
still succeeds — the jar just isn't on the deployed PU's classpath, so the failure only shows up at
PU deploy/runtime, often manifesting as the `LocalSessionFactoryBean` gotcha from
`mirror-service.md` even though the real cause is a missing dependency, not the wrong class name.

**If you add, rename, or upgrade a dependency in a module using an allowlist-style `assembly.xml`,
update the allowlist too** — it will not pick up new dependencies automatically the way an
exclude-based `assembly.xml` does.
