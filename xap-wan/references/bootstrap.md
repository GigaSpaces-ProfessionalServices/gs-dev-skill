# WAN Gateway: Bootstrapping a New Site

Bringing a new site online without replaying every historical write through normal replication —
instead, pull the existing site's current data in one explicit operation, then let ordinary
replication take over from that point on. Use when adding a site to an already-running topology, or
recovering a site that needs to be re-seeded from a surviving one.

## Concept

A sink deployed with `requires-bootstrap="true"` **blocks normal replication** until an explicit
bootstrap runs — the gateway link can show `INTACT`/connected while the underlying space still has
zero objects. An `AdminBootstrapInitiator`-style client then explicitly pulls the remote site's data
in; once that completes, replication resumes normally in both directions.

## Topology and deploy order

Both sites otherwise run a full delegator and sink (this is active-active underneath — see
`active-active.md` for that baseline). The only functional difference between the two deployments is
the sink's `requiresBootstrap` value. **Deploy order matters here more than in any other WAN
scenario:**

```
1. Deploy and write to the EXISTING site (EMEA) alone — no gateway link yet.
2. Deploy the NEW site (US) with requires-bootstrap=true.
   → US starts empty even though its gateway is connected to EMEA.
3. Run the bootstrap client against US.
   → US ends up with exactly EMEA's data.
4. Ordinary bidirectional replication continues from here.
```

```bash
# 1. Bring up EMEA and write to it only
gs.sh --server=emea-manager service deploy --zones=EMEA-space   -p localSpaceName=wanSpaceEMEA -p localGatewayName=EMEA -p remoteGatewayName=US   EMEA-space   SpacePU.jar
gs.sh --server=emea-manager service deploy --zones=EMEA-gateway -p localGatewayName=EMEA -p remoteGatewayName=US -p localSpaceUrl=jini://*/*/wanSpaceEMEA -p requiresBootstrap=false ... EMEA-gateway GatewayPU.jar
# write to EMEA here

# 2. Bring up US — requiresBootstrap=true blocks automatic replication
gs.sh --server=us-manager service deploy --zones=US-space   -p localSpaceName=wanSpaceUS -p localGatewayName=US -p remoteGatewayName=EMEA   US-space   SpacePU.jar
gs.sh --server=us-manager service deploy --zones=US-gateway -p localGatewayName=US -p remoteGatewayName=EMEA -p localSpaceUrl=jini://*/*/wanSpaceUS -p requiresBootstrap=true ... US-gateway GatewayPU.jar

# 3. Trigger the bootstrap (see AdminBootstrapInitiator below)
```

## Gateway PU — the `requires-bootstrap` XSD trap

**Do not** try `<os-gateway:sink requires-bootstrap="${requiresBootstrap}" ...>`. It fails at
XML-parse time — `requires-bootstrap` is XSD-typed as `boolean` on the `<os-gateway:sink>` element,
and Xerces validates a raw `${requiresBootstrap}` string against that type before Spring's
`PropertyPlaceholderConfigurer` ever gets a chance to resolve it. This isn't a runtime failure you
can catch — it's a startup-time schema validation rejection.

**Fix**: declare the sink as a plain `GatewaySinkFactoryBean` bean instead of the `<os-gateway:sink>`
element. A plain bean's `<property>` value stays an ordinary `String` until Spring's own type
conversion runs, which happens *after* placeholder resolution — so it accepts the placeholder fine.

```xml
<bean id="remoteGatewaySource" class="org.openspaces.core.gateway.GatewaySource">
    <property name="name" value="${remoteGatewayName}" />
</bean>

<bean id="sink" class="org.openspaces.core.gateway.GatewaySinkFactoryBean">
    <property name="localGatewayName" value="${localGatewayName}" />
    <property name="gatewayLookups" ref="gatewayLookups" />
    <property name="startEmbeddedLus" value="true" />
    <property name="localSpaceUrl" value="${localSpaceUrl}" />
    <property name="requiresBootstrap" value="${requiresBootstrap}" />
    <property name="gatewaySources">
        <list value-type="org.openspaces.core.gateway.GatewaySource">
            <ref bean="remoteGatewaySource" />
        </list>
    </property>
</bean>
```

The delegator and `<os-gateway:lookups>` stay as normal elements — only the sink needs this
treatment, and only because `requiresBootstrap` specifically needs to vary by deploy. If a topology
never needs `requires-bootstrap` to be placeholder-driven (i.e. it's hardcoded per dedicated module,
like `active-passive.md`), the plain `<os-gateway:sink>` element is fine.

## Triggering the bootstrap

A one-shot Admin API client, not a Processing Unit — run it once, it exits when done:

```java
import org.openspaces.admin.Admin;
import org.openspaces.admin.AdminFactory;
import org.openspaces.admin.gateway.BootstrapResult;
import org.openspaces.admin.gateway.Gateway;
import org.openspaces.admin.gateway.GatewaySinkSource;
import java.util.concurrent.TimeUnit;

Admin admin = new AdminFactory().addLocator(locators).useDaemonThreads(true).create();
try {
    Gateway usGateway = admin.getGateways().waitFor("US");
    GatewaySinkSource emeaSinkSource = usGateway.waitForSinkSource("EMEA");
    BootstrapResult result = emeaSinkSource.bootstrapFromGatewayAndWait(3600, TimeUnit.SECONDS);
    System.out.println(result);
} finally {
    admin.close();
}
```

`waitFor("US")` blocks until the US gateway PU is discoverable; `waitForSinkSource("EMEA")` blocks
until US's sink has actually connected to EMEA's delegator — don't call `bootstrapFromGatewayAndWait`
before that connection exists.

## Verify

Before the bootstrap runs, confirm the new site is genuinely empty despite a connected gateway
(proves `requires-bootstrap` is actually blocking, not a no-op flag):

```bash
gs.sh --server=us-manager space info --type-stats wanSpaceUS
```

After the bootstrap, confirm it matches the source site's counts, then write to the new site and
confirm replication continues normally in both directions from that point on.

## Pitfalls specific to this scenario

- **Don't skip the "confirm it's empty first" verification step.** If `requires-bootstrap` was
  accidentally left `false` (or the sink is a plain `<os-gateway:sink>` element that silently failed
  to parse the placeholder — see above), the new site may already have partial data through ordinary
  replication before you ever run the bootstrap client, and you won't notice the flag didn't work.
- **`waitForSinkSource` can hang indefinitely** if the two gateways never actually connect (wrong
  `lookups` host/port, network segmentation — see `SKILL.md`'s cross-cutting pitfalls). If the
  bootstrap client just sits there, check gateway connectivity before assuming the bootstrap itself
  is broken.
