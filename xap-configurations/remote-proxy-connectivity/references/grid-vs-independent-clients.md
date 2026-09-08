# Independent Clients vs. Clients Running Inside a Deployed PU

`locators-and-groups.md` describes an **independent client** — any JVM that is not itself part of
the grid: a standalone `java` process, a test, an external app server, `gs.sh pu run` (runs a PU's
container *without* deploying it to a grid). A remote space-proxy created by code running **inside
a Processing Unit that has been deployed to a GSC** (`gs.sh service deploy` / `gs.sh pu deploy`)
behaves differently by default, and conflating the two is a real source of confusing bug reports
("it just works when deployed, but the identical code fails/hangs standalone").

## The mechanism

A proxy with **zero** explicit locators/groups — the classic `<os-core:space-proxy
space-name="X"/>`, and equally the modern `SpaceProxyBeansConfig`/`gs-service-config.yaml`
auto-config used by `gs blueprint stateless` templates — connects successfully to another
PU-deployed space in the same grid even with multicast fully disabled. The identical "no locators,
no groups" configuration, run as a genuinely independent standalone process against the same
target with multicast also disabled, fails with the ordinary "lookup finder" error.

Pointing a deployed PU's proxy at a space name that isn't deployed anywhere fails with the exact
same `CannotFindSpaceException`/`FinderException` an independent client gets — it's the same
`LookupFinder` code path, not a different, grid-registry-aware bypass. The exception dump reveals
the real difference:

```
Jini Lookup Groups: [xap-17.3.0]
Jini Lookup Locators: [jini://localhost:4174/]
Number of Lookup Services: 1
```

`Number of Lookup Services: 1` — a lookup service was genuinely found; it just correctly has no
space registered under that name. **A proxy created inside a deployed PU defaults its locator to
the grid's own manager lookup service** (the same one its GSC joined at startup) instead of leaving
locators empty and relying on multicast. The default *group* is unaffected — still the ordinary
version-derived default (`xap-17.3.0`). This is why a `gs blueprint stateless` template can ship
with no locators/groups anywhere in `gs-service-config.yaml` and "just work" once deployed: the grid
supplies a working locator for free, not because groups/locators stopped mattering.

`gs.sh pu run` (a PU's container run standalone, **not** deployed to any grid) gets none of this —
it's an ordinary independent client and falls back to the same multicast + version-derived-default-
group mechanism as any raw standalone client.

## Practical implications

- This only helps when the **target is itself deployed in the same grid**. A PU-hosted proxy
  reaching for an external space (a standalone `gs.sh space run` process, or a space in an unrelated
  grid) gets no special treatment — it needs the same explicit `lookup-locators`/`lookup-groups` (or
  working multicast) as any independent client.
- If a PU's proxy "just works" with no locators/groups when deployed but the same code fails or
  hangs when run standalone/in a unit test/via `gs.sh pu run`, this is why — it's not a regression.
  Give the non-grid context explicit locators/groups rather than assuming something broke.
- Don't assume a deployed PU's unconfigured proxy will always connect — it fails exactly like an
  independent client (same exception, same diagnostic value in the dump) whenever the target isn't
  actually part of the same grid.

## Pitfalls specific to this topic

| Symptom | Cause | Fix |
|---|---|---|
| PU deploys fine, its proxy connects to another space in the same grid with no locators/groups configured anywhere | Expected — a deployed PU's proxy defaults its locator to the grid's own manager LUS | No action needed; don't add locators/groups defensively here — the default only works because the proxy is genuinely running inside the grid |
| Identical "no locators, no groups" code works deployed but fails standalone / under `gs.sh pu run` / in a unit test | Only a proxy created inside a grid-deployed PU gets the free manager-LUS default; a standalone process or `pu run` is an ordinary independent client | Give the non-grid context explicit `lookup-locators`/`lookup-groups`, or run the test against a grid-deployed target instead |
| A deployed PU's proxy still throws `CannotFindSpaceException` for a target that's confirmed running | Target isn't deployed into the *same* grid — the free locator default only resolves targets registered on that grid's manager LUS | Configure explicit locators/groups for that external target, same as an independent client would need |
| A JVM option/system property set via a `gs.sh` env-var override doesn't seem to take effect on the process you meant it for | Assumed a per-role override variable exists (e.g. one scoped to just the GSC) when it doesn't — `gs.sh` silently ignores an unrecognized override variable rather than erroring, so the typo-looking mistake produces no feedback at all | Use `GS_OPTIONS_EXT`; it's session-wide, applying to every `gs.sh`-launched process (agent, manager, and all GSCs alike) rather than one role. Confirm the flag actually landed by checking the target process's real command line, not by assuming the env var took effect |
