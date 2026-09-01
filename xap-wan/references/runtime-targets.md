# WAN Gateway: Adding and Removing Targets at Runtime

Pausing or resuming replication toward a specific site — e.g. because it's going down for planned
maintenance — without redeploying the space or the gateway. A site's outbound gateway-target list is
mutable at runtime through the Admin API; the gateway link (delegator/sink) itself never needs to
change.

## Concept

A **connected gateway is necessary but not sufficient for replication.** `<os-gateway:targets>` can
be deployed with **zero** `<os-gateway:target>` children — this is schema-valid (`<os-gateway:target>`
is `minOccurs="0"` in `openspaces-gateway.xsd`), not a workaround. A space deployed this way is
gateway-enabled (its `ReplicationManager` accepts `addGatewayTarget`/`removeGatewayTarget` calls) but
starts with nothing to replicate to, even while its gateway PU's delegator and sink show `CONNECTED`
to the remote site. Don't mistake a connected gateway for evidence that writes are actually leaving
the site — check the space's live target list (see Pitfalls in `SKILL.md`).

Targets only affect writes made **after** they're added — adding a target has no retroactive effect
on data already in the space. This is the opposite of `bootstrap.md`'s `AdminBootstrapInitiator`,
which exists specifically to pull a site's *existing* data in one explicit operation; ordinary target
add/remove never does that.

## Topology

Both sites deploy a fully connected gateway (delegator + sink), same as `active-active.md`. What's
different is the space PU: its `<os-gateway:targets>` list is empty on deploy, and stays that way
until something calls the Admin API against it.

```
 wanSpaceUS   ── gateway: CONNECTED ──   wanSpaceEMEA
  targets: []                             targets: []
```

## Space PU — empty target list

```xml
<os-core:embedded-space id="space" space-name="${localSpaceName}" mirrored="false"
                         gateway-targets="gatewayTargets" />
<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager" />

<os-gateway:targets id="gatewayTargets" local-gateway-name="${localGatewayName}" />
```

No `<os-gateway:target>` children, on either site — unlike `active-active.md`, this doesn't even need
a `-p remoteGatewayName` override for the target list, since there's nothing in it to parameterize.

## Gateway PU

Identical delegator/sink/lookups wiring to `active-active.md` — see that file for the
`<os-gateway:*>` XML form. As an alternative to XML, the gateway PU can also be wired as plain
`@Configuration`/`@Bean` classes instead — useful since there's no XSD attribute to trip over, and it
keeps all three gateway beans in one place:

```java
@Configuration
public class GatewayBeansConfig {

    @Value("${localGatewayName:US}") private String localGatewayName;
    @Value("${remoteGatewayName:EMEA}") private String remoteGatewayName;
    @Value("${localSpaceUrl:jini://*/*/wanSpaceUS}") private String localSpaceUrl;
    @Value("${requiresBootstrap:false}") private boolean requiresBootstrap;
    // ...lookup host/port fields...

    @Bean
    public GatewayLookupsFactoryBean gatewayLookups() {
        GatewayLookup local = new GatewayLookup();
        local.setGatewayName(localGatewayName);
        // ...host/discoveryPort/communicationPort...
        GatewayLookup remote = new GatewayLookup();
        // ...same, for the remote side...
        GatewayLookupsFactoryBean lookups = new GatewayLookupsFactoryBean();
        lookups.setGatewayLookups(List.of(local, remote));
        return lookups;
    }

    // Calling gatewayLookups() here (not taking it as a parameter) relies on @Configuration's
    // CGLIB proxying to return the same singleton — the annotation equivalent of both
    // os-gateway:delegator and os-gateway:sink referencing one shared gateway-lookups bean by id.
    @Bean
    public GatewayDelegatorFactoryBean delegator() {
        GatewayDelegatorFactoryBean delegator = new GatewayDelegatorFactoryBean();
        delegator.setLocalGatewayName(localGatewayName);
        delegator.setGatewayLookups(gatewayLookups());
        delegator.setStartEmbeddedLus(true);
        delegator.setGatewayDelegations(Collections.singletonList(new GatewayDelegation(remoteGatewayName, null)));
        return delegator;
    }

    @Bean
    public GatewaySinkFactoryBean sink() {
        GatewaySinkFactoryBean sink = new GatewaySinkFactoryBean();
        sink.setLocalGatewayName(localGatewayName);
        sink.setGatewayLookups(gatewayLookups());
        sink.setStartEmbeddedLus(true);
        sink.setLocalSpaceUrl(localSpaceUrl);
        sink.setRequiresBootstrap(requiresBootstrap);
        sink.setGatewaySources(Collections.singletonList(new GatewaySource(remoteGatewayName)));
        return sink;
    }
}
```

```java
@Configuration
@Import({DefaultServiceConfig.class, GatewayBeansConfig.class})
public class ServiceConfig {
}
```

This needs a compile-time dependency on `com.gigaspaces:xap-admin`, scoped `provided` (the
delegator/sink classes live there; the GSC already supplies them at runtime, so don't bundle a
duplicate copy). `bootstrap.md`'s XSD trap only forces the *sink* into a plain bean, for one
placeholder — going fully annotation-based moves the delegator and lookups too, whether or not any of
them are placeholder-driven.

## Adding and removing a target — Admin API

A one-shot Admin API client, not a Processing Unit — same shape as `bootstrap.md`'s
`AdminBootstrapInitiator`:

```java
import org.openspaces.admin.Admin;
import org.openspaces.admin.AdminFactory;
import org.openspaces.admin.space.Space;
import org.openspaces.core.gateway.GatewayTarget;
import java.util.concurrent.TimeUnit;

Admin admin = new AdminFactory().addLocator(locators).useDaemonThreads(true).create();
try {
    Space space = admin.getSpaces().waitFor(spaceName, waitTimeoutSeconds, TimeUnit.SECONDS);

    // Add: the `true` is the resetTarget flag (GS-14480, 15.8.1+) — it relaxes the
    // sequential-packet-numbering check normally enforced when a target is (re-)added, so a
    // target that was previously removed can be added back cleanly instead of being rejected
    // for a gap in packet numbers.
    space.getReplicationManager().addGatewayTarget(new GatewayTarget(gatewayName), true);

    // Remove:
    space.getReplicationManager().removeGatewayTarget(gatewayName);
} finally {
    admin.close();
}
```

`admin.getSpaces().waitFor(spaceName, timeout, unit)` blocks until the space PU is discoverable —
same pattern as `bootstrap.md`'s `waitFor`/`waitForSinkSource` on gateways, just against a space
instead. Always pass `true` for `resetTarget` unless you specifically need the packet-numbering check
enforced — without it, re-adding a target you previously removed is rejected.

## Verify

Check the target site's per-type counts before and after each add/remove step:

```bash
gs.sh --server=emea-manager space info --type-stats wanSpaceEMEA
```

- Before any target exists: `wanSpaceEMEA` has 0 objects even though the gateway shows `CONNECTED` —
  confirms connectivity alone doesn't replicate anything.
- After adding the target and writing fresh data: those new writes show up on `wanSpaceEMEA`, but
  anything written to `wanSpaceUS` *before* the target was added is still missing — targets aren't
  retroactive.
- After removing the target and writing again: `wanSpaceEMEA`'s counts are unchanged — writes made
  with no active target stay local, exactly like before the target was ever added.

## Pitfalls specific to this scenario

- **A `CONNECTED` gateway proves the link is up, not that anything is replicating.** Always check the
  space's own target list (or just try a write and confirm it lands) before concluding a topology or
  config problem — the two failure modes (no target vs. `requires-bootstrap` pending, see
  `bootstrap.md`) look identical from the gateway's connection state alone.
- **Adding a target back after removing it needs `resetTarget=true`.** Omitting it (or calling the
  single-argument `addGatewayTarget(GatewayTarget)` overload, if present) can get rejected over a gap
  in replication packet numbering left by the earlier removal.
- **Don't expect a newly-added target to backfill data written before it existed.** If a site needs to
  catch up on a target's *entire* history, that's `bootstrap.md`'s job, not this one — the two aren't
  interchangeable.
