# Mirror Service: Full Hibernate Setup

Wiring a working Hibernate-backed Mirror from scratch — the space-side and Hibernate/Spring
plumbing. For the exception-handling policy question ("what happens when a persistence write
fails"), see `gigaspaces-xap`'s `mirror-persistence.md` instead — this file assumes that decision
is already made.

## Space side: `mirrored="true"`

```xml
<os-core:embedded-space id="space" name="my-space" mirrored="true"/>
```

That's the entire space-side requirement — it marks the space as replicating to a mirror. Nothing
else about the space PU changes for the basic case (contrast with `initial-load.md`, where the
*same* attribute set also carries `space-data-source` for loading data back in).

## Mirror side: `dataSource` → `sessionFactory` → sync endpoint → `<os-core:mirror>`

```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="org.postgresql.Driver"/>
    <property name="url" value="jdbc:postgresql://localhost:5432/mydb"/>
    <property name="username" value="myuser"/>
    <property name="password" value="..."/>
</bean>

<bean id="sessionFactory" class="org.springframework.orm.jpa.hibernate.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <property name="packagesToScan" value="com.example.model" /> <!-- @Entity-annotated POJOs -->
    <property name="hibernateProperties">
        <props>
            <prop key="hibernate.dialect">org.hibernate.dialect.PostgreSQLDialect</prop>
            <prop key="hibernate.globally_quoted_identifiers">true</prop> <!-- see SKILL.md cross-cutting notes -->
            <prop key="hibernate.cache.use_second_level_cache">false</prop>
            <prop key="hibernate.hbm2ddl.auto">create</prop>
        </props>
    </property>
</bean>

<bean id="hibernateSpaceSyncEndpoint"
      class="org.openspaces.persistency.hibernate.DefaultHibernateSpaceSynchronizationEndpointFactoryBean">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>

<!-- Wrap with an exception handler here — see gigaspaces-xap's mirror-persistence.md.
     Skipping the wrapper isn't an error; it's an implicit choice of XAP's default
     retry-forever-and-block policy. -->
<bean id="mirrorExceptionHandler" class="com.example.persistency.MyMirrorExceptionHandler"/>
<bean id="exceptionHandlingSpaceSyncEndpoint"
      class="org.openspaces.persistency.patterns.SpaceSynchronizationEndpointExceptionHandler">
    <constructor-arg ref="hibernateSpaceSyncEndpoint"/>
    <constructor-arg ref="mirrorExceptionHandler"/>
</bean>

<os-core:mirror id="mirror" url="/./mirror-service"
                space-sync-endpoint="exceptionHandlingSpaceSyncEndpoint"
                operation-grouping="group-by-replication-bulk">
    <os-core:source-space name="my-space" partitions="2" backups="1"/>
</os-core:mirror>
```

`packagesToScan` needs to point at the package containing your `@Entity`-annotated model classes —
this is the actual class-to-table mapping mechanism; nothing about `<os-core:mirror>` itself knows
what to persist beyond what Hibernate's `sessionFactory` was told to scan.

## Deploy & verify

```bash
# Grid needs at least (partitions × (1+backups)) + 1 free GSCs — the +1 is the mirror PU itself
./gs.sh host run-agent --auto --gsc=5

./gs.sh pu deploy space-pu  <path>/space-pu/target/space-pu.jar
./gs.sh pu deploy mirror-pu <path>/mirror-pu/target/mirror-pu.jar

./gs.sh pu list   # both should show INTACT
```

Confirm the mirror actually connected — grep the GSC log for a "Channel established" line, once
per primary space instance (not once total):

```
mirror-pu INFO [...replication.channel.in.my-space1...mirror-service] - Channel established [...]
mirror-pu INFO [...replication.channel.in.my-space2...mirror-service] - Channel established [...]
```

Watch replication activity while writing to the space:

```bash
# Note: the PU/app-name (space-pu, used above with pu deploy/pu list) is not the space's own
# name — space subcommands need the latter (my-space here). See SKILL.md's cross-cutting notes.
./gs.sh space list-instances my-space
./gs.sh space info-instance --replication-stats my-space~1_1   # against each PRIMARY instance
./gs.sh space info-instance --replication-stats my-space~2_1
```

Then confirm rows actually landed in the database — remember the reserved-word quoting from
`SKILL.md`'s cross-cutting notes: `SELECT COUNT(*) FROM "User";`, not `FROM User;`.

## Known Gotchas (XAP 17.3.0 / Spring 7.0.8 / Hibernate 7.1.0.Final)

None of these are caught by `mvn install` — only actual PU deployment surfaces them:

1. **`org.springframework.orm.hibernate5.LocalSessionFactoryBean` no longer exists.** Spring
   Framework 7 removed that package entirely. Use
   `org.springframework.orm.jpa.hibernate.LocalSessionFactoryBean` — same property setters, new
   home.
2. **`spring-orm` isn't on the mirror PU's classpath by default.** It's easy to declare it
   `provided`-scope (meant to be supplied by the GSC container) and assume that's enough — but this
   XAP distribution ships it under `lib/optional/`, not the GSC's default classpath. Declare it
   `compile`-scope in the PU's own `pom.xml` so the assembly plugin bundles it into the PU's `lib/`.
3. **A mapped entity's name can collide with your target database's reserved words — worked example:
   `User` in PostgreSQL.** Hibernate's unquoted `CREATE TABLE User (...)` DDL fails with a syntax
   error there. Fixed via `hibernate.globally_quoted_identifiers=true` rather than renaming the
   entity — but that quotes *every* generated identifier, so every query against these tables needs
   quoted, case-preserved names afterward. **This is Postgres's specific reserved-word list and
   quoting behavior, not a general one** — check your actual target database's own reserved-word
   list and identifier-quoting rules (Oracle, for instance, upper-cases unquoted identifiers by
   default and quotes with the same double-quote syntax but different case semantics; SQL Server
   quotes with `[brackets]` instead).
