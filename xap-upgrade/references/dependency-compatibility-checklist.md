# Dependency-Compatibility Checklist

Walk through these before committing to a XAP version bump. Each is a place where "the space
server upgraded fine but the surrounding project didn't" bites teams — usually silently, surfaced
only at PU deploy time, not at `mvn install`. Version-agnostic on purpose — for the confirmed
version numbers behind each item for a specific jump, see the relevant `references/<from>-to-<to>.md`
file.

## 1. Java version supported by the target XAP version

Confirm the target release's supported JDK range against GigaSpaces' own release/lifecycle page
before touching anything else — it gates every other choice below, since a newer bundled
Spring/Spring Boot baseline usually requires a newer JDK too.

Don't assume the server's minimum JDK is automatically the right target for client applications
too. If it's unclear whether clients should track ahead of the server's own JDK floor, check the
target release's notes or ask GigaSpaces support — this has varied by release.

## 2. Spring Framework / Spring Boot — GigaSpaces has a strong dependency on Spring

XAP bundles Spring directly (`lib/required/spring-*.jar`, `lib/optional/spring/`) and several of
its own modules — Mirror/persistence, security, interop — are built on it. A Spring version bump
inside XAP is not optional or swappable; it travels with the XAP version.

- A Spring Framework major bump can remove or relocate packages that your own code — or a
  lab/starter kit you're extending — imports directly. Worked, already-verified example:
  `org.springframework.orm.hibernate5.LocalSessionFactoryBean` was removed outright in Spring 7;
  GigaSpaces' own Mirror service setup had to move to
  `org.springframework.orm.jpa.hibernate.LocalSessionFactoryBean` (same property setters, new
  package). Full details: `xap-persist`'s `mirror-service.md`, "Known Gotchas" section.
- **Direct `org.springframework.*` imports are supported anywhere, including PU code — but in
  practice, pinning the project's own Spring version to match whatever XAP bundles is
  significantly less friction than deliberately running a different one.** A mismatched version can
  still work, but it's the harder, less-trodden path; aligning versions avoids a category of
  problems before they start rather than debugging them after.
- **Practical check**: when the target XAP version bumps its bundled Spring major version, audit
  every place the project's own code imports `org.springframework.*` directly — not just through
  GigaSpaces' own wrapping API. Those direct imports are what a Spring major bump can silently
  break. `mvn install` won't catch it if the project's own compile-time Spring dependency still
  resolves; the break tends to surface only once the PU actually deploys and the classloader
  resolves against XAP's bundled version.
- **Spring Boot is not a supported deployment model for the GigaSpaces server itself** — the
  Processing Units that actually hold spaces are packaged as plain JARs deployed through the
  GigaSpaces Service Grid, not launched via a Spring Boot parent POM (`gigaspaces-xap`'s
  `maven-pom.md`, "Processing Unit (Colocated / Server-Side) POM" section spells out the
  non-Spring-Boot POM shape). GigaSpaces' own distribution reflects this too — it bundles only
  `spring-boot-loader.jar` (the fat-jar launcher machinery), not the full Spring Boot framework.
- Spring Boot **is** a supported and common choice on the **client side** — feeders, REST
  gateways, remoting clients connecting to a remote space. Some teams specifically prefer
  Spring Boot for these. See the same `maven-pom.md`'s "Spring Boot Client / REST Application POM"
  section for that shape. Don't conflate the two: "we use Spring Boot for our client app" says
  nothing about whether the space-holding PU itself can or should use it (it can't/shouldn't). Same
  practical caveat as above, though: a client app's Spring Boot (and the Spring Framework version
  it pulls in) is free to run independently of XAP's own bundled version, but in practice pinning
  it to align with what XAP bundles is significantly less friction than deliberately diverging.

## 3. Hibernate

If the project has no `<os-core:mirror>` wired to a Hibernate `SessionFactory` and doesn't use
`hibernateSpaceDataSource` for initial load, the Mirror-specific coupling below doesn't apply — but
see the "independently of the Mirror" note at the end of this section before skipping entirely.

**Hibernate-backed Mirror or initial load**: GigaSpaces' `xap-hibernate-spring.jar` is only the
integration glue between XAP and Hibernate; the actual Hibernate library version is supplied by the
project's own POM, not bundled by XAP. That makes "which Hibernate version" really a question of
"does the Hibernate version this project has pinned still work with the Spring version XAP now
bundles" — the two are coupled through `LocalSessionFactoryBean` and `spring-orm`, not independent
choices. See `xap-persist`'s `mirror-service.md` for the full worked Spring 7 / Hibernate
7.1.0.Final gotcha list (package moves, `spring-orm` classpath scope).

**Hibernate used independently of the Mirror** (a separate persistence layer elsewhere in the app,
not going through `xap-hibernate-spring.jar`): not forced into the same coupling as above, but the
same practical caveat as Spring (item 2) applies — pinning that Hibernate version, and
whatever Spring integration it uses, to align with what XAP bundles is far less friction than
deliberately diverging. That's especially true if the code shares a classloader with a PU, where it
becomes subject to the same Spring-version breakage either way.

## The pattern underlying all three above

The recurring risk isn't "does XAP itself upgrade cleanly" — it generally does. It's that **the
surrounding project's own code, or a team's, may not be upgradable in lockstep with XAP's
bundled dependencies**, and most of these breaks are invisible until actual deployment:

- Inventory every direct `org.springframework.*` / `org.hibernate.*` import in the project's own
  code (not just GigaSpaces' own API surface).
- Check whether any of them touch a package that moved or was removed in the new bundled version.
- Check whether the project's own pinned versions of these libraries are even compatible with —
  or are being silently shadowed by — XAP's bundled ones.
- Treat "compiles fine" as no signal here — every gotcha `mirror-service.md` documents is
  explicitly one that `mvn install` does not catch; only actual PU deployment surfaces it.

## 4. GigaSpaces client-server version alignment

Keeping client and server versions aligned is significantly less friction than deliberately letting them diverge. If there's a deliberate plan to
run older client versions, confirm it with GigaSpaces support first.

**Never run a newer client version against an older server — this direction is not supported.**