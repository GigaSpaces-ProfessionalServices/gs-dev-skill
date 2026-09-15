# GigaSpaces System Properties

Java system properties (`-D...`) that configure GigaSpaces directly, as an alternative or
complement to the `GS_*` environment variables covered in `environment-variables.md`. This file
covers a deliberately narrow slice of the full system-properties reference — paths/classloading,
discovery, and logging — not the complete list.

## Paths and classloading

| Property | Description | Default |
|---|---|---|
| `com.gs.home` | GigaSpaces home directory. Not required — if not set explicitly, it's resolved automatically. | `$GS_HOME` |
| `com.gs.deploy` | Location of the GSM's deploy directory. | `$GS_HOME/deploy` |
| `com.gs.work` | Location of the GSM's and GSC's work directory. | `$GS_HOME/work` |
| `com.gs.pu-common` | Location of common classes shared across multiple Processing Units. Libraries here load into **each PU instance's own classloader** (not the system classloader). | `$GS_HOME/lib/optional/pu-common` |
| `com.gigaspaces.lib.platform.ext` | PU-shared classloader libraries folder. Jars here load once into the **JVM system classloader**, shared across every PU instance in the GSC. | `$GS_HOME/lib/platform/ext` |

**`com.gs.pu-common` vs. `com.gigaspaces.lib.platform.ext`** — both let multiple PUs share a
library without repackaging it into each PU jar, but at different classloader scopes:
`com.gs.pu-common` loads into each PU instance's own classloader, while
`com.gigaspaces.lib.platform.ext` loads once into the GSC's system classloader and is shared by
every PU instance. For JDBC drivers and other third-party libraries, `com.gigaspaces.lib.platform.ext`
is usually the better choice — it avoids loading duplicate copies of the same driver into every PU
instance, and lets you update the driver in one place without touching any PU's own packaging.

## Discovery: system-property alternatives to GS_LOOKUP_*, and the multicast toggle

`com.gs.jini_lus.locators`/`com.gs.jini_lus.groups` reach the same discovery configuration as
`GS_LOOKUP_LOCATORS`/`GS_LOOKUP_GROUPS` from `environment-variables.md`, just via `-D` instead.
`com.gs.multicast.enabled` has no environment-variable equivalent — it's system-property only:

| Property | Description | Default |
|---|---|---|
| `com.gs.multicast.enabled` | Globally enables or disables multicast discovery. | `true` |
| `com.gs.jini_lus.locators` | Lookup Service locators used by client programs — the system-property alternative to setting `GS_LOOKUP_LOCATORS`. | — |
| `com.gs.jini_lus.groups` | Lookup Service groups — the system-property alternative to setting `GS_LOOKUP_GROUPS`. | `xap-[version]` |

**`com.gs.jini_lus.locators` and `GS_MANAGER_SERVERS` aren't meant to coexist.** `manager.md`
confirms this for the `GS_LOOKUP_LOCATORS` + `GS_MANAGER_SERVERS` combination specifically: a
client's `SpaceProxyConfigurer` fails immediately with `IllegalStateException: Ambiguous locators`
— a fail-fast validation error, not vague "confusing discovery behavior" and not a harmless
redundancy (see `environment-variables.md`'s `GS_MANAGER_SERVERS` row and `manager.md`'s Pitfalls
index). Whether `com.gs.jini_lus.locators` — the system-property form — triggers the identical
fail-fast check wasn't itself tested; treat it as likely the same by analogy, not as independently
confirmed. `com.gs.multicast.enabled` is the system-property form of the
same multicast toggle discussed in the `remote-proxy-connectivity` skill's `locators-and-groups.md`
("Multicast is optional" section) — see that file for the fuller discussion of when disabling
multicast is a deliberate, correct choice rather than a limitation to route around.

## Logging

| Property | Description | Default |
|---|---|---|
| `java.util.logging.config.file` | File path to the Java logging configuration. Use it to enable finer-grained logging for troubleshooting GigaSpaces services. | `$GS_HOME/config/log/xap_logging.properties` |

This is the system-property form of the same setting `environment-variables.md`'s
`GS_LOGS_CONFIG_FILE` controls via environment variable — two methods to the same goal, same as the
discovery properties above. Where both are viable, prefer the environment variable for consistency
with the rest of this skill's `GS_*`-first guidance.

## Pitfalls index

| Symptom | Likely cause | Fix |
|---|---|---|
| A third-party library (e.g. a JDBC driver) is loaded once per PU instance instead of once per GSC, or updating it means repackaging every PU | The library was placed under `com.gs.pu-common` instead of `com.gigaspaces.lib.platform.ext` | Move shared third-party libraries to `com.gigaspaces.lib.platform.ext` instead |
| Client throws `IllegalStateException: Ambiguous locators` before any discovery attempt | `com.gs.jini_lus.locators` (or `GS_LOOKUP_LOCATORS`) is also still set alongside `GS_MANAGER_SERVERS` — confirmed fail-fast for the `GS_LOOKUP_LOCATORS` combination in `manager.md`; the `com.gs.jini_lus.locators` case is assumed identical by analogy, not independently tested | Remove the locator setting once `GS_MANAGER_SERVERS` is configured |
