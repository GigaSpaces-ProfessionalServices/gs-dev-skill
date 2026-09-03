---
name: xap-upgrade
description: >
  General methodology for planning a GigaSpaces XAP version upgrade — what to check before and
  during the jump, framed as a reusable checklist rather than a single hardcoded version-pair diff.
  Use when the user mentions upgrading XAP, migrating to a new XAP version, GigaSpaces version
  compatibility, Spring/Hibernate compatibility during an XAP upgrade, or asks what might break
  moving between XAP versions. Default XAP target version is 17.3.0 unless the user specifies
  otherwise.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 0.1.0
---

# GigaSpaces XAP Upgrade Skill

> Anchored to two verified version jumps so far
> (**17.2 → 17.3** and **17.1 → 17.2**). Deliberately **not** extending further back to
> older/EOL-track versions for now — see "Version Scope" below.

This skill answers the *planning* question — "what do I need to check before/while upgrading XAP
versions" — as a checklist of dependency-compatibility categories, not a single hardcoded diff
between two specific releases. The checklist categories are meant to generalize across jumps; the
specific version numbers cited under each one are only confirmed for the jump named above.

**Scope**: this skill covers what's needed to make an upgrade succeed — dependency compatibility,
config formats or features that stop working, breaking changes. It does **not** cover adopting new
features a target version happens to introduce (e.g. MCP Server, GraphQL, Semantic Layer, Vector
Search) — those are optional additions, not upgrade requirements, and don't belong here. The one
exception: if something reads as a **planned deprecation** of a feature or format currently in use,
that's in scope — it's a compatibility risk on the way out, not a new capability on the way in.
REST API V3 is exactly this case, not the "new feature" case: it was introduced in 17.2, and per
that release's own notes the V2 manager API (and the V2-based ops-ui) is deprecated alongside it,
slated for removal in a future version — a team still depending on V2/ops-ui has a real
migration to plan for.

**Confirmed**: XAP does not support a rolling, GSC-by-GSC upgrade of a live cluster across versions.
Don't suggest that as an upgrade strategy.

**Not currently covered**: what the actual supported upgrade procedure looks like given that
constraint (e.g. full cluster stop/redeploy, blue-green via a second cluster). Not yet verified
against GigaSpaces' own docs — don't assert a specific procedure here until that's confirmed,
rather than inferring one.

**Default target version: XAP 17.3.0**

---

## Identifying the Team's Version

Before consulting any reference file, establish which version(s) are actually in play — everything
below is keyed to specific version-pair files, and picking the wrong one produces confidently wrong
answers. Ways to confirm this, most reliable first:

1. **Ask directly** if it's ambiguous or the team already knows — cheaper and more reliable than
   inferring from artifacts.
2. **The project's own `pom.xml`** — the `<gigaspaces.version>` property (or equivalent
   `org.gigaspaces`/`com.gigaspaces` Maven coordinate) states the *target* version a project is
   built against. See `gigaspaces-xap`'s `maven-pom.md` for where this normally lives.
3. **The installed distribution itself**, for the *current* running version: the distribution root
   directory name typically encodes the version (e.g. `gigaspaces-smart-cache-enterprise-17.2.2`),
   and `$GS_HOME/lib`'s version-pinned jar filenames (e.g. `spring-web-6.2.15.jar`) can corroborate
   it — this is exactly how every version-pair file in this skill was itself verified.

Don't assume the current running version and the project's declared target version match. Also
don't assume a team's own dependency versions (Spring, Hibernate) will move just because XAP's
version does — see the checklist's "pinning to align is far less friction" guidance. Confirming
both XAP's version and the team's own dependency versions explicitly, rather than assuming
either, avoids diagnosing the wrong problem.

Once XAP's current/target versions are known, `references/17.1-to-17.2.md` and
`references/17.2-to-17.3.md` cover 17.1→17.2 and 17.2→17.3; see "Version Scope" below for jumps
spanning both, and what's not covered yet.

---

## Version Scope

This skill intentionally does not attempt to document every pairwise version jump — that's out of
scope. Reference files are named and organized one minor-version step at a time (17.1 → 17.2, not
17.1.5 → 17.2.2) purely to match how GigaSpaces' own documentation is organized — that's a
documentation/organizational choice, not a claim about what upgrade paths are actually possible or
supported. A team jumping straight from 17.1 to 17.3, skipping 17.2 entirely, is a real,
plausible scenario; when that comes up, read both `references/17.1-to-17.2.md` and
`references/17.2-to-17.3.md` and combine them — don't treat the per-step files as a requirement to
upgrade one minor version at a time. Each jump is actually verified against specific patch installs
(noted inside each reference file):

- **17.2 → 17.3** — done, see `references/17.2-to-17.3.md`.
- **17.1 → 17.2** — done, see `references/17.1-to-17.2.md`.
- Older/EOL-track versions (17.0 and earlier) — deliberately out of scope for now, even though a
  team may reasonably choose to do a major version upgrade only every other year and so in
  practice be jumping from one of these. Revisit if/when there's a concrete need.

**For a jump with no matching reference file** (e.g. 17.0 → 17.3, or anything involving 16.4 or
earlier): the checklist's categories still apply, but don't invent version numbers or findings by
pattern-matching from a covered jump. If a real installed distribution for the relevant version is
available, verify directly against it the same way `17.1-to-17.2.md` and `17.2-to-17.3.md` were
built — check jar filenames for version numbers (e.g. `spring-web-6.2.15.jar`), manifest files
(`Implementation-Version` in `spring-boot-loader.jar`), and whether a given config file is present
or absent between the two installs (e.g. `metrics.xml` vs. `metrics.properties`) — rather than
inferring. Otherwise, use GigaSpaces' own release/lifecycle notes.

---

## Reference Files

**MANDATORY**: Read the relevant reference file below **before citing specific version numbers or
confirmed changes** — this file's checklist stays version-agnostic on purpose; the reference files
hold the version-specific detail.

| Reference Path | Covers |
|------|--------|
| `references/dependency-compatibility-checklist.md` | The version-agnostic checklist itself — Java, Spring Framework/Boot, Hibernate, client-server version alignment, and why each matters |
| `references/17.2-to-17.3.md` | Confirmed version numbers for the 17.2 → 17.3 jump (Java, Spring Framework, Spring Security, Spring Boot, Hibernate), plus any deprecations noted for that window |
| `references/17.1-to-17.2.md` | Confirmed version numbers for the 17.1 → 17.2 jump (same categories), plus any deprecations noted for that window |

---

## Scope boundary with `gigaspaces-xap` / `xap-persist`

The Spring 7 / Hibernate 7 breaking-change details for the Mirror service specifically — the
`LocalSessionFactoryBean` package move, `spring-orm` classpath scope — are
already fully documented in `xap-persist`'s `mirror-service.md` (its "Known Gotchas" section and
cross-cutting notes #2/#3). `dependency-compatibility-checklist.md`'s items 2 and 3 both point
there rather than repeating it. Likewise, the Spring Boot vs. non-Spring-Boot POM split (client apps vs.
Processing Units) is already documented in `gigaspaces-xap`'s `maven-pom.md` — checklist item 2
points there too. Don't duplicate either file's content here if this skill grows.

---

## Quick Decision Guide

```
Planning a XAP version upgrade...
(rule of thumb throughout: your own Spring/Hibernate/client versions can diverge from XAP's bundled
 or server versions, but pinning to match is far less friction than deliberately diverging —
 checklist #2/#3/#4)

  ├── What JDK does the target version need?                        → checklist #1
  ├── Does the app use Spring independently of GigaSpaces (direct
  │   org.springframework.* imports, or Spring Boot)?                 → checklist #2 (its own JDK
  │                                                                       floor is checklist #1)
  │     ├── Direct org.springframework.* imports                     → supported anywhere, incl. PU code
  │     └── Spring Boot specifically, to package/launch...
  │           ├── ...a client app (feeder/REST/remoting)              → supported, see maven-pom.md
  │           └── ...the space-holding PU itself                      → not supported, see maven-pom.md
  ├── Does the project have a Hibernate-backed Mirror / initial load? → checklist #3, then
  │                                                                       xap-persist's mirror-service.md
  ├── Does the app use Hibernate independently of the Mirror?         → checklist #3; check its own
  │                                                                       compatibility too
  ├── Does the target version deprecate anything currently in use?    → check that jump's reference
  │                                                                       file's "Deprecation" section
  ├── Will GigaSpaces client and server versions be mismatched at
  │   any point during the upgrade?                                    → checklist #4
  │     ├── Client older than server (lagging behind)                 → ask GigaSpaces support first
  │     └── Client newer than server (ahead of it)                    → never — not supported
  └── "It compiled, will it deploy?"                                  → no signal either way; these
                                                                          breaks only surface at deploy

(checklist items → references/dependency-compatibility-checklist.md;
 confirmed version numbers for the current anchor jump → references/17.2-to-17.3.md)
```

---

## Cross-References

| Question | See |
|---|---|
| The general checklist (why each category matters) | `references/dependency-compatibility-checklist.md` |
| Confirmed version numbers for the 17.2 → 17.3 jump | `references/17.2-to-17.3.md` |
| Confirmed version numbers for the 17.1 → 17.2 jump | `references/17.1-to-17.2.md` |
| Full Spring 7 / Hibernate 7 Mirror-service gotchas (package moves, classpath scope) | `xap-persist/references/mirror-service.md` — "Known Gotchas" + Cross-Cutting Notes #2/#3 |
| Spring Boot Client POM vs. Processing Unit (non-Spring-Boot) POM shapes | `gigaspaces-xap/references/maven-pom.md` |
| Mirror's exception-handling policy (separate from the dependency-upgrade question above) | `gigaspaces-xap/references/mirror-persistence.md` |
