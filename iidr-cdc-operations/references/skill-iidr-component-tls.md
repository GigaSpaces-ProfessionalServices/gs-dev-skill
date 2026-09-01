# Skill: Encrypt communication between IIDR components (TLS)

> **Scope note — read this first:** this covers encrypting the **IIDR-internal** legs only:
> Access Server ↔ clients, and CDC engine ↔ CDC engine (the replication data channel). It does
> **NOT** cover encrypting the Kafka message bus itself (broker listeners on :9092, `security.protocol` in
> `kafkaproducer.properties`), nor Oracle SQL*Net (:1521 → TCPS). Those are separate, independent efforts —
> don't assume this skill secures the whole pipeline end to end.

Validated on IIDR 11.4.0.5, build master_5726 (AS + CDC Engine for Kafka + CDC Engine for Oracle).
Certificate handling and `tls.properties` steps follow the vendor install guide, section
"Configuring TLS for IIDR (Access Server, agents and client)"; the engine-to-engine profile
activation steps (Part D) reflect what was actually needed to retrofit TLS onto **already-running**
instances, which goes beyond what that install-time walkthrough covers.

## What gets encrypted, and how

There are two independent layers. Securing one says **nothing** about the other — both are needed.

### Layer A — Access Server protocol traffic (the control plane)

Anything that talks **to the Access Server** to manage or query the setup: the Management Console (GUI),
`chcclp` (CLI client), di-subscription-manager (REST). This traffic is commands and metadata — "create a
subscription," "start mirroring," "what's the status" — **never actual replicated table data**.

**How it's secured:** a `tls.properties` file, present on the AS and on every client that talks to it. Each
side has `trustStorePath` (who it trusts), `enableTLS=true`, and — on the AS only — `privateKeyStorePath`
(its own identity; clients don't need to prove who they are in this setup).

### Layer B — engine-to-engine replication channel (the data plane)

The actual pipe between the **Oracle CDC engine** and the **Kafka CDC engine** — a completely separate TCP
connection from Layer A, on the engines' own ports (11001 / 11701 in this environment), carrying the
captured row changes themselves. If Layer A is encrypted but Layer B isn't, anyone who can see the wire
between the two engines still sees every replicated row in the clear.

**How it's secured:** an **encryption profile** — a named bundle (server keystore + trust store + an
enablement level) attached to the engine's instance config. Unlike Layer A this isn't a simple file edit:
creating a profile and **activating** it are two separate steps (a profile can exist and simply not be in
effect), and the enablement level matters a lot — see Part D.

### Which layer the vendor install guide actually covers

| | Layer A (AS-protocol) | Layer B (engine-to-engine) |
|---|---|---|
| Covered in the vendor install guide? | Yes — "Configuring TLS for IIDR", with real certs | Only the **unencrypted** case (the install transcript answers `Disabled`) |
| Encrypted version documented there? | Yes | **No** — not shown anywhere in that guide |

Following the guide end to end leaves Layer A encrypted and Layer B still plaintext — it never revisits
the encryption-profile menu with real certs and `Enabled` selected. Part D of this skill fills that gap with
what was actually worked out and validated on already-running instances.

**Key fact:** IIDR negotiates TLS **in-band**, after a short proprietary preamble — not from byte 0. So
`openssl s_client` against an IIDR port always "fails" and is **not a valid test**. Verify instead via packet
capture (look for TLS record headers `16 03 03` / `17 03 03`) or the engine's own event log (event IDs
1579/1581, see Verification below).

## Part A — Get certificate material into Java keystores

**IIDR doesn't generate its own certs.** You (or your PKI/security team) provide four files — a private key
and three certificates — and this step packages them into the two Java keystores IIDR actually reads. This
does not cover *issuing* the certs — substitute your own CA/process, self-signed or corporate, as long as
you end up with these four files.

**You need, before starting:**

| File | What it is |
|---|---|
| `server.key` | the server's private key |
| `server.cer` | the certificate for that key, signed by your CA |
| `client.cer` | a client certificate, signed by the same CA |
| `ca.cer` | the CA's own public certificate (the trust anchor) |

Do the rest **once**, anywhere convenient (doesn't have to be an IIDR host — but if it is, run as the IIDR
service account, not root).

1. **Bundle the private key + server cert into PKCS12** (OpenSSL, then convert to JKS with `keytool` — you
   can't build a JKS with a private key directly from a raw `.key`+`.cer` pair):
   ```bash
   openssl pkcs12 -export -inkey server.key -in server.cer -out server.p12 -name myserver
   # prompts for an export password — this becomes the keystore password

   keytool -importkeystore -srckeystore server.p12 -srcstoretype PKCS12 \
     -destkeystore server.jks -deststoretype JKS
   ```
2. **Validate the identity keystore:**
   ```bash
   keytool -v -list -keystore server.jks
   # expect: 1 entry, alias "myserver", Entry type: PrivateKeyEntry
   ```
3. **Build the trust store** — import the client cert, the CA cert, and the server cert as separate trusted
   entries (a flat "trust these three" store, not a certificate-chain validation setup):
   ```bash
   keytool -importcert -file client.cer -keystore trust.jks -alias myclient -noprompt
   keytool -importcert -file ca.cer     -keystore trust.jks -alias myca     -noprompt
   keytool -importcert -file server.cer -keystore trust.jks -alias myserver -noprompt
   ```
4. **Validate the trust store:**
   ```bash
   keytool -v -list -keystore trust.jks | grep -A3 Alias
   # expect 3 entries, all Entry type: trustedCertEntry
   ```
5. **Copy both keystores to every host that needs them** — same path on each (e.g. `/giga/iidr/tls/`):
   ```bash
   scp server.jks trust.jks <other-host>:/giga/iidr/tls/
   ssh <other-host> "chown gsods:gsods /giga/iidr/tls/{server,trust}.jks && chmod 644 /giga/iidr/tls/{server,trust}.jks"
   ```

**⚠️ Which `keytool` you build the trust store with matters.** Each IIDR engine bundles its **own** JRE, and
an older bundled JRE (e.g. IBM JRE 8 SR7) cannot read a trust store built by a newer JRE's default algorithms
(PBES2/AES) — the file is still named `.jks` but is internally PKCS12, and fails at runtime with "trust
store is invalid," or breaks TLS silently with no clear error. **Rebuild the trust store with the *oldest*
bundled JRE among every component that will read it** — e.g. `<oldest-engine-install>/jre64/jre/bin/keytool`,
never the system `java`/`keytool`. Verify what you built is real JKS: magic bytes `FE ED FE ED`
(`od -A x -t x1 -N 4 trust.jks`); `30 82` means it's actually PKCS12 — rebuild it.

### Where these files go

| File | Contains | Needs to be present on |
|---|---|---|
| `server.jks` | server private key + its signed cert | AS host + every engine host, identical file (used as `privateKeyStorePath`) |
| `trust.jks` | CA + server + client certs, no private keys | AS host + every engine host + any AS client host (used as `trustStorePath`) |
| `server.key`, `server.p12`, loose `.cer` files | working files | keep off the IIDR hosts once the `.jks` files are built; restrict permissions |

## Part B — Access Server TLS

1. Restrict the file before putting secrets in it: `chmod 600 tls.properties` (as the doc recommends —
   it holds keystore passwords).
2. Edit `<AS-install>/tls.properties`:
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
   `ENABLED` vs `REQUIRED` note in Part D.) Optionally add `encodedStorePassword=true` (11.4.0.4-11107+) and
   store the two passwords base64-encoded instead of plaintext.
3. Point every AS client (e.g. di-subscription-manager's own `tls.properties`) at the same trust store —
   this first section of the file (trust store only) is shared by both the AS and its clients; the
   `privateKeyStore*` lines are AS-only.
4. `systemctl restart iidr_as_inst`.
5. Verify: capture a real client session and look for a TLS handshake in the payload (see Verification).

## Part C — Engine `tls.properties` (Layer A, per engine)

Same file format as Part B, in each engine's install dir (e.g. `<kafka-engine>/tls.properties`,
`<oracle-engine>/tls.properties`). Use the JKS trust store from Part A on **every** engine, especially
any with an older bundled JRE.

## Part D — Engine-to-engine encryption profile (Layer B)

**Two ways to get here, pick based on whether the instance already exists:**

- **Brand-new instance:** `dmconfigurets` asks for an encryption profile as one of the normal instance
  parameters *during creation* ("Add an Instance" → "Select an encryption profile" → "Manage encryption
  profiles" → "Add encryption profile" → name it → pick enablement → point it at your `server.jks`/
  `trust.jks` from Part A → save). Selecting the profile right there **activates** it for the new instance —
  no extra step needed. This is the path in the install guide, and it's how instances created with the
  wizard's default answers end up with a deliberately disabled placeholder profile (e.g. `kafkaDummy`).
- **Existing, already-running instance** (the case handled here): re-running the profile wizard through
  "Edit an Instance" creates the profile fine, but activating a *changed* profile on a live instance needs
  the extra export→edit→import step below — creating alone was not enough in practice. Use steps 1–3 below.

**Engine-to-engine encryption enablement has four levels**, not two — pick the setting when prompted (either
path):

| Level | Effect |
|---|---|
| Disabled | plaintext (this is what the `*Dummy` placeholder profiles use) |
| **Enabled** | opportunistic TLS — negotiates encryption between engines that support it, but still accepts IIDR's own internal plaintext loopback clients. **Use this one.** |
| Required | refuses any unencrypted connection, **including IIDR's own internal agent-reader** — silently starves the redo scraper (see the warning below) |
| Always | not exercised in this work — treat as untested here, confirm behavior before relying on it |

For an **existing** instance, do the following **per engine**, engine **stopped**, running as the IIDR
service account (not root).

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
4. Do Part D (create + activate profile) on both engines.
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
