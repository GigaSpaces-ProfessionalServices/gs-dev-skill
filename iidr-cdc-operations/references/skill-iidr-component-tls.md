# Skill: Encrypt communication between IIDR components (TLS)

> **Scope note — read this first:** this covers encrypting the **IIDR-internal** legs only:
> Access Server ↔ clients, and CDC engine ↔ CDC engine (the replication data channel). It does
> **NOT** cover encrypting the Kafka message bus itself (broker listeners on :9092, `security.protocol` in
> `kafkaproducer.properties`), nor Oracle SQL*Net (:1521 → TCPS). Those are separate, independent efforts —
> don't assume this skill secures the whole pipeline end to end.

Validated on IIDR 11.4.0.4-5672 (AS + CDC Engine for Kafka + CDC Engine for Oracle).

## What gets encrypted, and how

There are two independent layers, both needed:

- **Layer A — AS-protocol comms** (`tls.properties` in each component's install dir): encrypts
  client↔Access-Server traffic (Management Console, chcclp, di-subscription-manager) and each engine's
  control-plane link to the AS. Does **not** encrypt the replication data channel.
- **Layer B — engine-to-engine replication channel**: an **encryption profile**
  (`dmcreateencryptionprofile`) that must then be **activated** in the instance config. This is what
  encrypts the actual CDC data as it flows between the two engines.

**Key fact:** IIDR negotiates TLS **in-band**, after a short proprietary preamble — not from byte 0. So
`openssl s_client` against an IIDR port always "fails" and is **not a valid test**. Verify instead via packet
capture (look for TLS record headers `16 03 03` / `17 03 03`) or the engine's own event log (event IDs
1579/1581, see Verification below).

## Prerequisites

- A CA + server cert + client cert, in two forms:
  - a genuine **JKS** keystore holding the server private key (identity for AS + both engines)
  - a genuine **JKS** trust store (import CA/client/server certs with `keytool`)
- **Watch out for PKCS12-mislabeled-as-`.jks`:** if a trust store was built with modern OpenSSL (PBES2/AES),
  older bundled JREs on some engines can't read it and fail with "trust store is invalid" or break TLS
  silently. Check magic bytes: `FE ED FE ED` = real JKS, `30 82` = PKCS12. If in doubt, rebuild the trust
  store as classic JKS with `keytool -importcert` using the **oldest** engine's bundled JRE, so it's readable
  everywhere. Engines always use their own bundled JRE, not the system `java`.

## Part A — Access Server TLS

1. Edit `<AS-install>/tls.properties`:
   ```properties
   trustStorePath=/path/to/trust.jks
   trustStorePassword=<pw>
   trustStoreType=JKS
   privateKeyStorePath=/path/to/server.jks
   privateKeyStorePassword=<pw>
   privateKeyStoreType=JKS
   enableTLS=true
   datastoresAlwaysTLS=false
   ```
   (`datastoresAlwaysTLS=false` = opportunistic AS→datastore TLS rather than forced — deliberate, see the
   `ENABLED` vs `REQUIRED` note in Part C.)
2. Point every AS client (e.g. di-subscription-manager's own `tls.properties`) at the same trust store.
3. `systemctl restart iidr_as_inst`.
4. Verify: capture a real client session and look for a TLS handshake in the payload (see Verification).

## Part B — Engine `tls.properties` (Layer A, per engine)

Same file format as Part A, in each engine's install dir (e.g. `<kafka-engine>/tls.properties`,
`<oracle-engine>/tls.properties`). Use the JKS trust store from Prerequisites on **every** engine, especially
any with an older bundled JRE.

## Part C — Engine-to-engine encryption profile (Layer B)

Do this **per engine**, engine **stopped**, running as the IIDR service account (not root).

1. Back up the instance config first (cheap, and the profile change lives in the instance metadata store):
   ```bash
   cp -a instance/<INST>/conf instance/<INST>/conf.bak-$(date +%Y%m%d)
   cp -p tls.properties tls.properties.bak-$(date +%Y%m%d)
   ```
2. Create the profile (from the engine's install dir):
   ```bash
   bin/dmcreateencryptionprofile -n MYTLS -e ENABLED \
     -pf /path/to/server.jks  -pp <pw> -pt JKS \
     -tf /path/to/trust.jks  -tp <pw> -tt JKS
   ```
   **⚠️ Use `-e ENABLED`, not `REQUIRED`.** `REQUIRED` also blocks IIDR's own internal plaintext
   loopback client (the agent-reader connecting to its own engine), which **silently starves the redo
   scraper at 0% CPU while the subscription still shows "Mirror Continuous"** — no error surfaces
   anywhere except event 9604 in the engine log. `ENABLED` is opportunistic: engine↔engine still
   negotiates TLS, internal components keep working in plaintext.
   Judge success by the **absence** of error text — the tool can exit 0 while printing an error, and vice
   versa.
3. **Activate** the profile — creating it is not enough; each instance has an `encryptionConfiguration`
   field that defaults to a do-nothing placeholder. There's no CLI setter; use export → edit → import:
   ```bash
   bin/dmexportconfiguration /tmp/config.xml        # prompts for a password on a real TTY
   sed -i 's|<encryptionConfiguration>.*</encryptionConfiguration>|<encryptionConfiguration>MYTLS</encryptionConfiguration>|' /tmp/config.xml
   bin/dmimportconfiguration /tmp/config.xml         # same password prompt
   ```
   These tools require a real TTY; for automation, drive them with `expect`:
   ```tcl
   #!/usr/bin/expect -f
   set timeout 90
   spawn {*}$argv
   expect { -re "password" { send "<password>\r"; exp_continue } eof }
   ```
   Verify by re-exporting and grepping `<encryptionConfiguration>` — **don't** trust
   `instance/<INST>/conf/tsprop`, it's a stale creation-time snapshot.

## Maintenance-window order of operations

Expect ~10–15 min of replication pause; mirroring resumes from bookmark, no data loss.

1. Stop di-subscription-manager (it auto-restarts idle subscriptions every ~15 s and will fight you).
2. Gracefully end replication for every subscription (via chcclp: `end replication;`) — never stop an
   engine with active subscriptions, or the next start fails with "target still has active subscriptions."
3. Stop both engines. **Verify the JVM actually died** — `pgrep -f 'dmts64-jav[a]'` — `systemctl stop` has
   been observed to leave the process running.
4. Do Part C (create + activate profile) on both engines.
5. Start both engines; confirm their ports are listening and no keystore/TLS errors appear in the fresh
   trace log.
6. Restart mirroring per subscription (chcclp: `start mirroring;`).
7. Restart di-subscription-manager.

## Verification (authoritative)

1. Engine's own event log after mirroring restarts, per subscription:
   - event **1579**: "The Control channel is encrypted with the TLSv1.3 protocol and the ... cipher suite."
   - event **1581**: same, for the **Data channel**.
2. On the wire: capture traffic between the two engines during active mirroring — every payload should start
   with a TLS record header (`17 03 03` = application data), not a plaintext proprietary frame.
3. No recurring event **9604** (TLS refusal). If you see it, an internal/legacy client is being blocked —
   the profile is probably `REQUIRED`; switch it to `ENABLED`.

## Rollback

Per engine: stop it → restore the `conf.bak-*` directory over `instance/<INST>/conf` → restore
`tls.properties.bak-*` → start it. For the AS: comment out the `enableTLS`/keystore lines in `tls.properties`
and restart. To soften just the channel policy without a full rollback, rerun the profile-create command
with `-e DISABLED` and restart the engine.

## Pitfalls index

| Pitfall | Consequence | Rule |
|---|---|---|
| Testing IIDR ports with `openssl s_client` | False "no TLS" verdict | Verify via packet capture or events 1579/1581 |
| `-e REQUIRED` | Redo scraper starves silently, status still shows "Mirror Continuous" | Use `-e ENABLED`; watch for event 9604 |
| Creating a profile without activating it | Placeholder profile stays active, still plaintext | Always export → edit `encryptionConfiguration` → import, then re-verify |
| Grepping `tsprop` to check config | Stale creation-time snapshot | Use `dmexportconfiguration` |
| Trust store is actually PKCS12 | Old bundled JRE fails with "trust store is invalid" or breaks TLS silently | Check magic bytes; rebuild as classic JKS |
| Trusting `systemctl stop` to kill the engine | Stale JVM causes "already running" errors later | `pgrep -f 'dmts64-jav[a]'` after every stop |
| dm* tool exit codes | rc=0 with error text, or vice versa | Judge success by output text, not exit code |
| Running dm* tools as root | Tools refuse | Run as the IIDR service account |
| Leaving di-subscription-manager running during the window | Auto-restarts subscriptions mid-change | Stop it first, start it last |
