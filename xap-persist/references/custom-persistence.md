# Custom Persistence (Hand-Written JDBC, No Hibernate)

Full control over persistence by implementing the sync endpoint and data source interfaces
directly against raw JDBC, instead of Hibernate. Use this as a reference for any persistence
target Hibernate doesn't fit well (a store with no JPA provider, or where you need precise control
over the SQL). More setup than `mirror-service.md`'s Hibernate path — nothing is generated for you.

## Write-behind: `SpaceSynchronizationEndpoint`

```java
package com.example.persistence;

import com.gigaspaces.sync.DataSyncOperation;
import com.gigaspaces.sync.OperationsBatchData;
import com.gigaspaces.sync.SpaceSynchronizationEndpoint;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.Resource;

public class MySpaceSynchronizationEndpoint extends SpaceSynchronizationEndpoint {

    @Resource
    private DataSource datasource; // e.g. org.apache.commons.dbcp.BasicDataSource

    private UserDAO userDAO;

    @PostConstruct
    public void init() {
        userDAO = new UserDAO();
        userDAO.setDatasource(datasource);
        userDAO.initWithCreateIfMissing(); // creates the table on first connect if missing
    }

    @Override
    public void onOperationsBatchSynchronization(OperationsBatchData batchData) {
        super.onOperationsBatchSynchronization(batchData);
        for (DataSyncOperation operation : batchData.getBatchDataItems()) {
            if (operation.supportsDataAsObject()) {
                Object obj = operation.getDataAsObject();
                if (obj instanceof User) {
                    userDAO.writeObjectToDB(obj);
                }
                // type-dispatch to the rest of your DAOs similarly
            }
        }
    }
}
```

`onOperationsBatchSynchronization` is the one method that matters for basic write-behind — it's
called with a batch of mixed-type operations, same `OperationsBatchData` shape
`gigaspaces-xap`'s `mirror-persistence.md` documents for the exception-handling case. This class
doesn't rethrow to signal failure the way a `PersistencyExceptionHandler` does — wrap it with
`SpaceSynchronizationEndpointExceptionHandler` (see that file) for the same retry/dead-letter
control if needed.

## Initial load: `ManagedEntriesSpaceDataSource`

```java
package com.example.persistence;

import com.gigaspaces.datasource.DataIterator;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.Resource;
import org.openspaces.persistency.patterns.ManagedEntriesSpaceDataSource;

import java.util.ArrayList;
import java.util.HashSet;

public class MySpaceDataSource extends ManagedEntriesSpaceDataSource {

    @Resource
    private HashSet<String> mEntries; // the fully-qualified class names this data source loads

    @Resource
    private DataSource datasource;

    private UserDAO userDAO;

    @Override
    public Iterable<String> getManagedEntries() {
        return mEntries; // XAP asks this what types it's responsible for
    }

    @PostConstruct
    public void init() {
        userDAO = new UserDAO();
        userDAO.setDatasource(datasource);
        userDAO.init();
    }

    @Override
    public DataIterator<Object> initialDataLoad() {
        ArrayList<Object> results = userDAO.readFromDB();
        // combine every DAO's results the same way
        return new CustomDataIterator(results); // a plain DataIterator<Object> wrapper
    }
}
```

`getManagedEntries()` and `initialDataLoad()` are the two methods that matter — the first tells
XAP what types this source is responsible for (used to validate/route), the second does the actual
load, run once at space startup.

## Wiring both into `pu.xml`

```xml
<!-- Declared identically in both the space PU and the sync-endpoint PU -->
<bean id="supportedManageSpaceClasses" class="java.util.HashSet">
    <constructor-arg>
        <set>
            <value>com.example.model.User</value>
            <value>com.example.model.Merchant</value>
        </set>
    </constructor-arg>
</bean>

<!-- Space PU: wires the custom data source via a FactoryBean -->
<bean id="myDataSourceFactory" class="com.example.persistence.MyCustomFactoryBean">
    <property name="managedEntries" ref="supportedManageSpaceClasses" />
    <property name="datasource" ref="dataSource" />
</bean>
<os-core:embedded-space id="space" name="my-space" mirrored="true" schema="persistent"
                         space-data-source="myDataSourceFactory"/>
<!-- schema="persistent" is likely a holdover from an older, XML-embedded external-data-source
     mechanism that predates this Spring-bean-based space-data-source attribute — see
     initial-load.md. Not a functional requirement: confirmed live that space-data-source
     loads correctly under schema="default" too. -->

<!-- Sync-endpoint PU: wired directly on <os-core:mirror>, no FactoryBean needed -->
<bean id="mySyncEndpoint" class="com.example.persistence.MySpaceSynchronizationEndpoint">
    <property name="datasource" ref="dataSource" />
</bean>
<os-core:mirror id="mirror" url="/./mirror-service" space-sync-endpoint="mySyncEndpoint">
    <os-core:source-space name="my-space" partitions="2" backups="1" />
</os-core:mirror>
```

The `FactoryBean` (`getObject()`/`getObjectType()`) indirection on the space side exists because
`ManagedEntriesSpaceDataSource` needs both `managedEntries` and `datasource` injected before use —
a plain `<bean class="...MySpaceDataSource">` declaration works too if those are just regular
`<property>` setters on the class itself; the factory pattern is only needed if construction has to
happen in a specific order or with extra logic.

## Known Gotchas (cross-database JDBC porting)

Hand-written SQL means porting to a different target database is a real job, not a config flip —
there's no ORM layer absorbing the difference for you. Three categories of problem show up in
basically every such port: JDBC driver strictness differences, reserved-word collisions, and type
mappings that don't carry over 1:1. The specifics below happened to surface migrating from MySQL to
PostgreSQL, none caught by `mvn install`, only at runtime — they're kept as a concrete worked example
of those three categories, **not a Postgres-specific checklist**. Expect an equivalent but different
set of surprises porting to Oracle, SQL Server, or anywhere else — same three categories, different
specifics, and you'll need to research your own target database's version of each rather than reuse
the ones below.

1. **JDBC driver strictness differs even for spec-defined behavior.** `ResultSet.first()` requires a
   scrollable result set per the JDBC spec; MySQL's driver tolerated calling it on a plain
   forward-only `ResultSet` anyway, PostgreSQL's driver enforced the spec strictly and threw
   `PSQLException: Operation requires a scrollable ResultSet, but this ResultSet is FORWARD_ONLY`.
   Use `resultSet.next()` for a single-row existence check instead — equivalent, and portable
   regardless of which driver you're on. Avoid `.first()`/`.last()`/`.absolute()` entirely unless the
   statement was explicitly created with `TYPE_SCROLL_INSENSITIVE`. Whatever database you're
   actually targeting may or may not enforce this same spec point as strictly — don't assume either
   way without checking.
2. **A catch block that assumes every `SQLException` means "table doesn't exist" masks real
   failures.** A common `initWithCreateIfMissing()`-style pattern retries with `CREATE TABLE` on
   *any* exception without checking what it actually said — so a genuine connection failure (e.g.
   the database still finishing its own startup) gets misdiagnosed the same way, and can then NPE
   on a null connection in the fallback path instead of failing with a clear error. Log or inspect
   the actual `SQLException` before assuming what it means. (This one's fully database-agnostic —
   it's an application bug, not a portability issue.)
3. **Reserved words and type mappings don't carry over between databases.** Here: `User` collided
   with a PostgreSQL reserved word, and MySQL types needed translating — `int(11)` → `integer`,
   `double` → `double precision`, `datetime` → `timestamp`. With hand-written SQL there's no
   `hibernate.globally_quoted_identifiers`-style escape hatch (see `mirror-service.md`) — either
   quote every reference or rename the table (simpler than quoting everywhere it's referenced, e.g.
   `user` → `app_user`). **The specific word, quoting syntax, and type list above are Postgres's —
   look up your actual target database's own reserved-word list and type-mapping table** rather than
   assuming this one transfers.
4. **Undeploying the space PU after its mirror/sync-endpoint PU is already gone hangs on drain.** A
   `mirrored="true"` space drains pending replication before undeploying; if the mirror is already
   undeployed, that drain can never succeed and blocks for the full timeout before failing. Undeploy
   the space PU *before* the mirror PU, or pass `--drain-mode=NONE` to skip the drain.
5. **Querying right after deploy can see 0 objects or "type descriptor not found," even when the
   load is working.** Custom initial load runs as part of space startup and can take several
   seconds; types also register lazily on first object of that type, so a type with genuinely no
   data yet won't resolve as a known type at all until something of that type exists. Wait a beat
   before concluding initial load didn't run.
