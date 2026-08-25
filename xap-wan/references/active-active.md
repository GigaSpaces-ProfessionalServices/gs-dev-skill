# WAN Gateway: Active-Active (Bidirectional Replication)

Both sites are full peers: either can accept writes, and each replicates out to the other. Use for
multi-region applications where both sites need to serve local writes with eventual cross-site
consistency (not for global uniqueness — see `conflict-resolution.md` for what happens when both
sides write to the same identity independently).

## Topology

```
 US                                    EMEA
 ┌──────────────────────────────┐    ┌─────────────────────────────────┐
 │ us-manager                   │    │ emea-manager                    │
 │ us-space-gsc                 │    │ emea-space-gsc                  │
 │ us-gateway-gsc  ─────────────┼────┼──  emea-gateway-gsc             │
 │  (delegator + sink)          │    │  (delegator + sink)             │
 └──────────────────────────────┘    └─────────────────────────────────┘
```

Because both sides run the same shape — delegator, sink, and a space with `gateway-targets` — this
is symmetric enough for **one shared module per role**, deployed twice (once per site) with
different `-p` overrides, instead of `active-passive.md`'s four dedicated modules. This is the
default pattern to reach for whenever both sides genuinely have the same bean set; only fall back
to per-site modules when they don't (see `replication-filter.md`, where one side has an extra bean
the other doesn't).

## Space PU (shared module, deployed once per site)

```xml
<os-core:embedded-space id="space" space-name="${localSpaceName}" mirrored="false"
                         gateway-targets="gatewayTargets" />
<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager" />

<os-gateway:targets id="gatewayTargets" local-gateway-name="${localGatewayName}">
    <os-gateway:target name="${remoteGatewayName}" />
</os-gateway:targets>
```

Placeholders: `localSpaceName`, `localGatewayName`, `remoteGatewayName` — all ordinary, safe to
resolve via `PropertyPlaceholderConfigurer` since none of them sit on an XSD-typed attribute (unlike
`bootstrap.md`'s `requires-bootstrap`).

## Gateway PU (shared module, deployed once per site — both delegator and sink)

```xml
<os-gateway:delegator id="delegator" local-gateway-name="${localGatewayName}" gateway-lookups="gatewayLookups"
                      start-embedded-lus="true">
    <os-gateway:delegation target="${remoteGatewayName}"/>
</os-gateway:delegator>

<os-gateway:sink id="sink" local-gateway-name="${localGatewayName}" gateway-lookups="gatewayLookups"
                 start-embedded-lus="true" local-space-url="${localSpaceUrl}">
    <os-gateway:sources>
        <os-gateway:source name="${remoteGatewayName}"/>
    </os-gateway:sources>
</os-gateway:sink>

<os-gateway:lookups id="gatewayLookups">
    <os-gateway:lookup gateway-name="${localGatewayName}"  host="${localLookupHost}"  discovery-port="${localLookupPort}"  communication-port="${localCommunicationPort}"/>
    <os-gateway:lookup gateway-name="${remoteGatewayName}" host="${remoteLookupHost}" discovery-port="${remoteLookupPort}" communication-port="${remoteCommunicationPort}"/>
</os-gateway:lookups>
```

Both `<os-gateway:delegator>` and `<os-gateway:sink>` are present — that's the entire difference
from `active-passive.md`'s two gateway modules.

## Deploy (one shared jar, `-p` overrides per site)

```bash
gs.sh --server=us-manager service deploy --zones=US-space \
  -p localSpaceName=wanSpaceUS -p localGatewayName=US -p remoteGatewayName=EMEA \
  US-space SpacePU.jar

gs.sh --server=us-manager service deploy --zones=US-gateway \
  -p localGatewayName=US -p remoteGatewayName=EMEA \
  -p localSpaceUrl=jini://*/*/wanSpaceUS \
  -p localLookupHost=us-gateway-gsc -p localLookupPort=4174 -p localCommunicationPort=8201 \
  -p remoteLookupHost=emea-gateway-gsc -p remoteLookupPort=4174 -p remoteCommunicationPort=8201 \
  US-gateway GatewayPU.jar
```

Mirror this with `EMEA`/`US` swapped and `--server=emea-manager` for the other side — same two
jars, different `-p` values.

## Verify

Write to either site and confirm the other converges to the same record count — unlike
`active-passive.md`, both directions need to be exercised, not just one:

```bash
gs.sh --server=us-manager space query wanSpaceUS \
  com.example.model.Payment --max-results=2000 --columns=paymentId
```

Both `wanSpaceUS` and `wanSpaceEMEA` should show matching counts regardless of which site
originally received the write.

## Packaging note

If the space module's `pu.xml` references domain model classes (for deserializing the entries being
replicated) but the gateway module's `pu.xml` never does, the space module needs a
`maven-assembly-plugin` bundling those model classes into its deployable jar — the gateway module
can usually get away with Maven's default jar packaging, which already bundles
`META-INF/spring/pu.xml` from `src/main/resources`.

## Pitfalls specific to this topology

- **`localSpaceUrl` in the gateway PU must point at the space PU's actual name** (`${localSpaceName}`
  from the space module) — if these two placeholders' resolved values ever drift apart across the
  two deploy commands, the gateway silently ends up not writing to (or reading from) the space you
  think it is.
- Bidirectional replication means **both** sides need writes exercised to fully prove the topology
  out — confirming replication with only one side ever written to doesn't prove the reverse
  direction actually works.
- If both sites can genuinely write the *same* logical record independently, see
  `conflict-resolution.md` — active-active alone doesn't define what happens when that occurs, it
  just means it's possible.
