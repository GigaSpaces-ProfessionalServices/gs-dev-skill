# WAN Gateway: Active-Passive (One-Directional Replication)

One site (active) accepts writes and delegates them out. The other (passive) only ever receives —
it has no delegator and never sends anything back. Use for classic DR: all traffic hits one site,
the other exists purely as a hot/warm standby copy.

## Topology

Two sites, each with its own manager + space GSC(s) + a dedicated gateway GSC:

```
 US (active)                                    EMEA (passive)
 ┌──────────────────────────────┐           ┌───────────────────────────────┐
 │ us-manager                   │           │ emea-manager                  │
 │ us-space-gsc                 │           │ emea-space-gsc                │
 │ us-gateway-gsc  ────delegator──▶ emea-gateway-gsc                        │
 │   (delegator only, no sink)  │           │   (sink only, no delegator)   │
 └──────────────────────────────┘           └───────────────────────────────┘
```

Because the two sides are **structurally asymmetric** (US's space PU has `gateway-targets`, EMEA's
doesn't; US's gateway PU is delegator-only, EMEA's is sink-only), this needs **four separate
modules**, not one shared module deployed twice with different property overrides — there's no
single `pu.xml` that's correct for both sides here. Contrast with `active-active.md`, where the two
sides are symmetric enough that one shared module works.

## Space PU — active side (delegates)

```xml
<os-core:embedded-space id="space" space-name="wanSpaceUS" mirrored="false"
                         gateway-targets="gatewayTargets" />
<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager" />

<os-gateway:targets id="gatewayTargets" local-gateway-name="US">
    <os-gateway:target name="EMEA" />
</os-gateway:targets>
```

`gateway-targets` on the embedded space is what makes writes attempt delegation at all — without
it, this would be a completely ordinary, non-replicating space.

## Space PU — passive side (sink target)

```xml
<os-core:embedded-space id="space" space-name="wanSpaceEMEA" mirrored="false" />
<os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager" />
```

No `gateway-targets` at all — this space never delegates anywhere.

## Gateway PU — active side (delegator only)

```xml
<os-gateway:delegator id="delegator" local-gateway-name="US" gateway-lookups="gatewayLookups"
                      start-embedded-lus="true">
    <os-gateway:delegation target="EMEA"/>
</os-gateway:delegator>

<os-gateway:lookups id="gatewayLookups">
    <os-gateway:lookup gateway-name="US"   host="us-gateway-gsc"   discovery-port="4174" communication-port="8201"/>
    <os-gateway:lookup gateway-name="EMEA" host="emea-gateway-gsc" discovery-port="4174" communication-port="8201"/>
</os-gateway:lookups>
```

No `<os-gateway:sink>` — this gateway never receives.

## Gateway PU — passive side (sink only)

```xml
<os-gateway:sink id="sink" local-gateway-name="EMEA" gateway-lookups="gatewayLookups"
                 start-embedded-lus="true" local-space-url="jini://*/*/wanSpaceEMEA">
    <os-gateway:sources>
        <os-gateway:source name="US"/>
    </os-gateway:sources>
</os-gateway:sink>

<os-gateway:lookups id="gatewayLookups">
    <os-gateway:lookup gateway-name="EMEA" host="emea-gateway-gsc" discovery-port="4174" communication-port="8201"/>
    <os-gateway:lookup gateway-name="US"   host="us-gateway-gsc"   discovery-port="4174" communication-port="8201"/>
</os-gateway:lookups>
```

No `<os-gateway:delegator>` — this gateway never sends.

Every `<os-gateway:lookups>` block, on both sides, lists **both** the local and every remote
gateway it needs to find — a sink still needs to know how to reach the delegator it's sinking from,
even though it never delegates anything itself.

## Deploy

```bash
gs.sh --server=us-manager service deploy --zones=US-space   US-space   SpacePU-US.jar
gs.sh --server=us-manager service deploy --zones=US-gateway US-gateway GatewayPU-US.jar

gs.sh --server=emea-manager service deploy --zones=EMEA-space   EMEA-space   SpacePU-EMEA.jar
gs.sh --server=emea-manager service deploy --zones=EMEA-gateway EMEA-gateway GatewayPU-EMEA.jar
```

No `-p` overrides needed — each of the four modules is dedicated to one side, so its values are
hardcoded rather than deploy-time parameters.

## Verify

Write data at US, then query EMEA and confirm it shows up:

```bash
gs.sh --server=emea-manager space query wanSpaceEMEA \
  com.example.model.Payment --max-results=2000 --columns=paymentId
```

There's no reverse-direction check to make — EMEA has no delegator, so nothing should ever flow
US-ward. If it does, something is misconfigured (check for an accidental `gateway-targets` on the
EMEA space, or a `<os-gateway:delegator>` that shouldn't be there).

## Pitfalls specific to this topology

- **Don't try to "simplify" this into one shared module with `-p` overrides.** It looks tempting
  since `active-active.md` does exactly that, but the bean sets genuinely differ here (one side has
  `gateway-targets`+delegator, the other doesn't have either) — a property override can change a
  *value*, not remove an entire XML element. Keep the four modules separate.
- **The gateway GSC's bind address needs to be reachable by both its own site's manager and the
  remote gateway** — see `SKILL.md`'s cross-cutting pitfalls table for why (the bind address is
  JVM-wide, so it can't serve two disjoint peers).
