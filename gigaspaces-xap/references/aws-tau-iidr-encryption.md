# AWS-TAU: Enable TLS encryption across the IIDR replication stack (AS ↔ Kafka engine ↔ Oracle engine)

| Field | Value |
|---|---|
| **Type** | Task / Epic report |
| **Environment** | AWS TAU staging (`tfjosh` env1) |
| **Components** | IIDR Access Server, IIDR CDC Engine for Kafka, IIDR CDC Engine for Oracle |
| **Hosts** | `10.0.4.143` (iidr1: AS + Kafka engine), `10.0.4.215` (dba1: Oracle engine), `10.0.1.129` (Oracle DB12), `10.0.1.201/.90/.18` (dih1-3: Kafka brokers, DI-manager, Flink) |
| **Dates** | 2026-07-15 → 2026-07-16 |
| **Status** | ✅ Done (TLS on all three IIDR components + E2E verified). Follow-up open: TDE-table CDC (see §8). |

---

## 1. Background & problem statement

An audit of the IIDR replication stack found that **every network leg was unencrypted**:

| Leg | Before | Evidence |
|---|---|---|
| Access Server (10101) ↔ clients (MC / chcclp / di-subscription-manager) | ❌ plaintext | `tls.properties` fully commented out; TLS probe got no handshake |
| Oracle engine (11001) ↔ Kafka engine (11701) — **the CDC data channel** | ❌ plaintext | Packet capture showed proprietary `0x80`-framed payloads carrying table data in the clear |
| Kafka engine → Kafka brokers (9092) | ❌ plaintext | No `security.protocol` in `kafkaproducer.properties` (out of scope for this ticket — cluster-wide change) |
| Oracle engine → Oracle DB (1521) | ❌ plaintext TCP (not TCPS) | `tnsnames.ora` uses `PROTOCOL=TCP`; no client `sqlnet.ora` (out of scope for this ticket) |

**Goal of this ticket:** encrypt the three IIDR components' communications — the AS protocol and the engine-to-engine replication channel — and prove it end-to-end (Oracle change → space) over the encrypted path.

**Key architectural fact discovered during the work:** IIDR negotiates TLS **in-band** (after a short proprietary preamble), not from byte 0. Therefore `openssl s_client` always "fails" against IIDR ports and is **not a valid test**. Valid verification methods are packet capture (look for TLS record headers `16 03 03` / `17 03 03` in payloads) and the engine's own event log (event IDs 1579/1581).

### PKI used

A self-signed CA and certs were pre-created in `/giga/iidr/tls/` on 10.0.4.143 (`gstestaws-ca`, `gstestaws-server`, `gstestaws-client`; password `gigaspaces123`). Two facts about these files mattered a lot:

- `gstestaws-server.jks` — genuine JKS, holds the server private key. Used as the TLS identity of AS and both engines.
- `gstestaws-trust.jks` — **despite the name, this is a PKCS12 file**, built with modern OpenSSL-3 algorithms (PBES2/PBKDF2/AES-256, SHA-256 MAC). See Task 3.1 for the consequence.

---

## 2. Component 1 — Access Server TLS (10.0.4.143)

### Task 1.1 — Configure `tls.properties` on the AS

**What/why:** the AS reads `<AS-install>/tls.properties` at startup; `enableTLS=true` plus a private-key store and trust store make it accept TLS from clients (MC, chcclp, di-subscription-manager).

**How:** edit `/giga/iidr/as/tls.properties`:

```properties
trustStorePath=/giga/iidr/tls/gstestaws-trust.jks
trustStorePassword=gigaspaces123
trustStoreType=PKCS12
privateKeyStorePath=/giga/iidr/tls/gstestaws-server.jks
privateKeyStorePassword=gigaspaces123
privateKeyStoreType=JKS
enableTLS=true
datastoresAlwaysTLS=false
```

Note: `datastoresAlwaysTLS=false` keeps AS→datastore connections on negotiated (opportunistic) TLS rather than forcing it — deliberate, see Task 4.2.

### Task 1.2 — Configure the client side (di-subscription-manager)

**What/why:** AS clients read the *first* section of their own `tls.properties` to know which CA to trust.

**How:** in `/giga/di-subscription-manager/di-subscription-manager-1.13.2.2/tls.properties` set `trustStorePath/-Password/-Type` to the same trust store.

### Task 1.3 — Restart the AS

```bash
systemctl restart iidr_as_inst      # on 10.0.4.143
```

### Verification (component 1)

1. Config actually loaded: restart happened **after** the file edit (`stat -c %y tls.properties` vs `ps -o lstart` of `dmaccessserver-java`).
2. Live proof — capture a real client session (`openssl s_client` is NOT valid here):
   ```bash
   tcpdump -i any -w /tmp/as.pcap port 10101 &
   su - gsods -c "cd /giga/iidr/as && bin/dmlistdatastores -accessserver 10.0.4.143 10101 admin <as-admin-password>"
   # then: tcpdump -r /tmp/as.pcap -X | less
   ```
   Expected: short plaintext preamble, then `16 03 03 … 01` (ClientHello), `… 02` (ServerHello), `14 03 03` (ChangeCipherSpec), then encrypted records. Observed: TLS 1.2 handshake completed. ✅
3. di-subscription-manager keeps polling with no connection errors in `logs/di-subscription-manager-iidr.log` (one transient `ERR2225` right at restart is normal — stale session).

---

## 3. Component 2 — CDC Engine for Kafka (10.0.4.143) & Component 3 — CDC Engine for Oracle (10.0.4.215)

Both engines need the same two layers:

- **Layer A — AS-protocol comms:** `tls.properties` in the engine install dir (same format as the AS one). This does **not** encrypt the replication data channel.
- **Layer B — engine-to-engine replication channel:** an **encryption profile** created with `dmcreateencryptionprofile` *and activated* in the instance configuration. This is what encrypts the actual CDC data on port 11701/11001.

### Task 3.1 — Rebuild the trust store as a true JKS (`gstestaws-trust2.jks`)

**What/why:** the two engines bundle *different* IBM JRE 8 service releases (Kafka engine: 1.8.0_361/SR8; Oracle engine: 1.8.0_351/SR7). The old SR7 JRE **cannot parse modern PKCS12** (PBES2/AES) — `dmcreateencryptionprofile` failed on 10.0.4.215 with `The trust store … is invalid`, and would also have silently broken that engine's TLS at runtime. A classic-format JKS is readable by every JRE. (The engines always run on their **bundled** JRE — the system `java`/`alternatives` setting is irrelevant.)

**How (on 10.0.4.143, as gsods):**

```bash
KT=/giga/iidr/kafka/jre64/jre/bin/keytool
for c in ca client server; do
  $KT -importcert -noprompt -alias gstestaws-$c \
      -file /giga/iidr/tls/gstestaws-$c.cer \
      -keystore /giga/iidr/tls/gstestaws-trust2.jks \
      -storepass gigaspaces123 -storetype JKS
done
# copy to 10.0.4.215:/giga/iidr/tls/ (chown gsods:gsods, chmod 644)
```

**Verify:** magic bytes are `fe ed fe ed` (`od -A x -t x1 -N 4 gstestaws-trust2.jks`), and the **old** JRE reads it:
`/giga/iidr/oracle/jre64/jre/bin/keytool -list -keystore /giga/iidr/tls/gstestaws-trust2.jks -storepass gigaspaces123` → lists 3 entries.

### Task 3.2 — Engine `tls.properties` (Layer A)

**How:** create/verify `/giga/iidr/kafka/tls.properties` (10.0.4.143) and `/giga/iidr/oracle/tls.properties` (10.0.4.215) with the same content as the AS file (Task 1.1), but pointing the trust store at **`gstestaws-trust2.jks` with `trustStoreType=JKS`** — mandatory on 10.0.4.215 (old JRE), recommended on both for uniformity.

**Verify:** `grep -vE '^\s*#|^\s*$' <install>/tls.properties` shows `enableTLS=true` + both stores; engine starts with no keystore errors in `instance/<INST>/log/trace_dmts_*.log`.

### Task 3.3 — Backup before touching instance metadata

**What/why:** the encryption profile is written into the instance metadata DB (`md.dbn` + WAL inside `instance/<INST>/conf/`). With the engines stopped, a cold copy is a complete, trivially restorable backup.

**How (as gsods, engines stopped):**

```bash
# 10.0.4.143
cp -a /giga/iidr/kafka/instance/KAFKA/conf /giga/iidr/kafka/instance/KAFKA/conf.bak-20260715
cp -p /giga/iidr/kafka/tls.properties /giga/iidr/kafka/tls.properties.bak-20260715
# 10.0.4.215
cp -a /giga/iidr/oracle/instance/ORACLE/conf /giga/iidr/oracle/instance/ORACLE/conf.bak-20260715
cp -p /giga/iidr/oracle/tls.properties /giga/iidr/oracle/tls.properties.bak-20260715
```

**Restore path:** stop engine → `rm -rf conf && mv conf.bak-20260715 conf` → start engine.

### Task 3.4 — Create the encryption profile on each engine (Layer B, part 1)

**What/why:** `dmcreateencryptionprofile` defines the TLS identity/trust and the enablement policy for **engine-to-engine** communications. Run per engine, **engine stopped**, as gsods. There is **no `-I` flag** — with a single instance per install it auto-selects (multi-instance installs use the `TSINSTANCE` env var).

**How (identical on both hosts, from the engine install dir):**

```bash
# 10.0.4.143: cd /giga/iidr/kafka   |   10.0.4.215: cd /giga/iidr/oracle
bin/dmcreateencryptionprofile -n GSTLS -e ENABLED \
  -pf /giga/iidr/tls/gstestaws-server.jks  -pp gigaspaces123 -pt JKS \
  -tf /giga/iidr/tls/gstestaws-trust2.jks -tp gigaspaces123 -tt JKS
```

**⚠️ Policy choice — `ENABLED`, not `REQUIRED`:** we first used `-e REQUIRED` and it **broke CDC silently**: IIDR's own internal agent-reader connects to its engine in plaintext over loopback, `REQUIRED` refused it (`AgentExecutive.ensureTlsEncrypted` exceptions, event 9604 "The datastore requires TLS encryption but the client does not support it"), and the redo scraper starved at 0% CPU **while the subscription still displayed "Mirror Continuous"**. `ENABLED` = opportunistic: engine↔engine still negotiates TLSv1.3 (both sides capable), internal plaintext components keep working. Rerunning the command with the same `-n` name updates the profile idempotently.

**Gotchas:** the tool can print an error yet exit 0 — judge success by the **absence of error text**. `-tt` must match the *real* file format (see Task 3.1).

### Task 3.5 — Activate the profile (Layer B, part 2)

**What/why:** **creating a profile does not activate it.** Each instance has an `encryptionConfiguration` field naming the active profile; out of the box it pointed to do-nothing placeholders (`kafkaDummy` on the Kafka engine, `prod-dummy` on the Oracle engine) — the reason the channel was plaintext in the first place. There is no CLI setter; use the export → edit → import cycle (engine stopped).

**How (as gsods, from the engine install dir):**

```bash
bin/dmexportconfiguration /tmp/config.xml          # prompts for a password on a TTY:
                                                   #   KAFKA instance: tsuser's password
                                                   #   ORACLE instance: DB user gsiidr's password
sed -i 's|<encryptionConfiguration>.*</encryptionConfiguration>|<encryptionConfiguration>GSTLS</encryptionConfiguration>|' /tmp/config.xml
bin/dmimportconfiguration /tmp/config.xml          # same password prompt
```

The tools insist on a real TTY (getpass-style); for automation drive them with `expect`:

```tcl
#!/usr/bin/expect -f
set timeout 90
spawn {*}$argv
expect { -re "password" { send "<password>\r"; exp_continue } eof }
```

**Verify:** re-export and check — `grep -o '<encryptionConfiguration>[^<]*' /tmp/verify.xml` → `GSTLS`. (Do **not** grep `instance/<INST>/conf/tsprop` — it's a creation-time snapshot, not live config.)

---

## 4. Maintenance-window runbook (order of operations)

The engine work (Tasks 3.3–3.5) must happen inside this sequence. Total replication pause ≈ 10–15 min; mirroring resumes from bookmark, no data loss.

| # | Step | Command / detail | Check |
|---|---|---|---|
| 1 | **Freeze automation** | `systemctl stop di-subscription-manager-iidr` (10.0.4.143). It auto-restarts idle subscriptions every 15 s and will fight the window. Also confirm `di-iidr-watchdog.timer` is inactive on both hosts. | `systemctl is-active` → inactive |
| 2 | **End replication gracefully** | chcclp on the AS: `connect server username admin password <pw>;` → `connect datastore name ORACLE context source;` → `connect datastore name KAFKA context target;` → per subscription: `select subscription name GS_xxxx; end replication;` | `monitor replication;` shows **Inactive** for all target subscriptions. Never stop an engine with active subscriptions ("target still has active subscriptions" error on next start). |
| 3 | **Stop both engines** | `systemctl stop iidr_kafka_inst` (143), `systemctl stop iidr_oracle_inst` (215). **Verify the JVM actually died** — `pgrep -f dmts64-jav[a]` (bracket avoids self-match); `pkill` leftovers. systemd stop did *not* kill the process at least once during this work. | no `dmts64-java` process |
| 4 | Tasks 3.3 → 3.4 → 3.5 on both engines | see above | export XML shows `GSTLS` |
| 5 | **Start engines** | `systemctl start iidr_oracle_inst`, `systemctl start iidr_kafka_inst` | listeners back (`ss -tlnp | grep -E '11001|11701'`); no keystore/TLS errors in fresh `trace_dmts_*.log` |
| 6 | **Restart mirroring** | chcclp: `select subscription name GS_xxxx; start mirroring;` per subscription | `monitor replication;` → **Mirror Continuous** |
| 7 | **Unfreeze automation** | `systemctl start di-subscription-manager-iidr` | its log clean; pipelines report RUNNING |

### Final verification of the replication channel

1. **Engine's own log (authoritative):** `bin/dmshowevents -I ORACLE -a -c 20` after mirroring start shows, per subscription:
   - event **1579** `The Control channel is encrypted with the TLSv1.3 protocol and the TLS_AES_256_GCM_SHA384 cipher suite.`
   - event **1581** — same for the **Data channel**. ✅
2. **On the wire:** `tcpdump -i any -c 60 'port 11701 and host 10.0.4.215'` during active mirroring → every payload starts `17 03 03` (TLS application data). Before the change the same capture showed plaintext `80 00 …` frames. ✅
3. No recurring event **9604** (TLS refusals) — if you see them, an internal/legacy client is being blocked: the profile is probably `REQUIRED`; switch to `ENABLED` (Task 3.4).

---

## 5. End-to-end validation (Oracle → space over the encrypted channel)

Performed with a dedicated pipeline to prove the full path: **Oracle redo → Oracle engine (.215) → TLSv1.3 :11701 → Kafka engine (.143) → Kafka topic → Flink CDC job → `dih-tau-space`**.

| # | Step | How | Check |
|---|---|---|---|
| 1 | Pick a source table | Chose `RETAIL_TEST.PRODUCTS` (4 rows): has a PK (`PRODUCT_ID`) and lives in the **unencrypted** `DEMO_PLAIN` tablespace. DB-level supplemental logging is `ALL=YES` so any table qualifies on that front. | `dba_tables`, `dba_constraints`, `dba_log_groups` |
| 2 | Attach table to pipeline | `POST http://10.0.1.201:6080/api/v1/pipeline/{id}/add_tables` body `{"sourceSchema":"RETAIL_TEST","sourceTables":["PRODUCTS"]}` | response `ALL_TABLES_ADDED_SUCCESSFULLY`; `GET …/tablepipeline` lists it |
| 3 | Start pipeline | `POST …/start` body `{"reconciliationPolicy":"NONE","kafkaRunParameters":{"CDC":{"kafkaOffsetStrategy":"EARLIEST"}}}` | `GET …/status` → Flink job RUNNING, subscription Mirror Continuous |
| 4 | Initial load | Automatic on first start (new tables are refresh-flagged); engine event: "refresh … is complete. 4 rows were sent" | topic `GS_359` end-offset = 4; space type `PRODUCTS` entries = 4 (`GET /v2/spaces/dih-tau-space/statistics/types`, auth gs-admin) |
| 5 | Query a record | `GET /v2/spaces/dih-tau-space/query?typeName=PRODUCTS&filter=PRODUCT_ID='1'` | row returned |
| 6 | **CDC test** | `UPDATE retail_test.products SET price=1234.56 WHERE product_id=1; COMMIT;` | space query shows `PRICE=1234.56` — **arrived in ≤ 10 seconds** ✅ |

Manager REST reference: `http://10.0.1.212:8090/v2` (Basic auth `gs-admin`). DI-manager REST: `http://10.0.1.201:6080/api/v1/pipeline/` (note the trailing slash; OpenAPI at `/api-docs`).

If the topic backlog was purged by Kafka retention (check: `kafka-get-offsets.sh --topic GS_xxxx --time -2` earliest == `--time -1` latest), a from-earliest restart replays nothing — reload with an IIDR refresh instead: end replication for the subscription, then on the source engine `bin/dmrefresh -I ORACLE -a -s GS_xxxx`, then start mirroring (`dmrefresh` refuses while the subscription is replicating).

---

## 6. Latency fixes applied along the way (Oracle side)

CDC latency after restarts was minutes-to-hours instead of seconds. Two causes, two fixes:

1. **Giant online redo logs:** 3×200 MB groups on a near-idle DB, `archive_lag_target=0` → one log spanned 12 days; every mirroring restart re-parsed the whole open log (slowly — it is read over a **read-only NFS mount**, `/mnt/oracle` → `10.0.1.129:/data/install/OracleDB19/oradata/DB12/onlinelog`; closed/archived logs read fast, tailing the open log is slow).
   **Fix:** `ALTER SYSTEM SET ARCHIVE_LAG_TARGET=1800 SCOPE=BOTH;` (forced switch ≥ every 30 min) — caps restart catch-up at ≤ 30 min of redo. An immediate `ALTER SYSTEM SWITCH LOGFILE;` unblocks a stuck catch-up instantly (the scraper "completes" the closed file at full speed).
   **Check:** `select group#, sequence#, status, first_time from v$log;` — the CURRENT log's `first_time` should never be more than ~30 min old.
2. **Steady-state expectation:** with logs rotating and no restarts, CDC latency is seconds (verified: ≤ 10 s in §5). The multi-minute delays only occur during post-restart catch-up.

---

## 7. Lessons learned / pitfalls index

| # | Pitfall | Consequence | Rule |
|---|---|---|---|
| 1 | Testing IIDR ports with `openssl s_client` | False "no TLS" verdict | IIDR TLS is in-band; verify via tcpdump payloads or events 1579/1581 |
| 2 | `-e REQUIRED` profile | Internal plaintext clients refused → redo scraper starves at 0 % CPU while status shows Mirror Continuous; no data errors anywhere | Use `-e ENABLED`; watch for event 9604 |
| 3 | `dmcreateencryptionprofile` alone | Profile exists but placeholder (`kafkaDummy`/`prod-dummy`) stays active → still plaintext | Always activate via export→edit `encryptionConfiguration`→import, and re-export to verify |
| 4 | Grepping `tsprop` to verify config | It's a creation-time snapshot — always stale | Verify with `dmexportconfiguration` |
| 5 | `.jks` file that is actually PKCS12 (modern algorithms) | Old bundled JRE (SR7) can't read it: "trust store is invalid" / silent TLS breakage | Check magic bytes (`FEEDFEED` = JKS, `3082` = PKCS12); build trust stores as classic JKS |
| 6 | Trusting `systemctl stop` to kill the engine | `dmts64-java` survived a stop; later "instance already running" / "still active on this target" errors | Always `pgrep -f 'dmts64-jav[a]'` after stop |
| 7 | dm* tools' exit codes | Error text with `rc=0` and vice versa | Judge by output text |
| 8 | Running dm* tools as root | "cannot be run as root" | `su - gsods` |
| 9 | di-subscription-manager left running during maintenance | Auto-restarts subscriptions mid-window every 15 s | Stop it first, start it last |
| 10 | Kafka topic offsets ≠ topic content | end-offset 999 but retention had deleted everything (earliest==latest) | Compare `--time -2` vs `--time -1` before planning replays |

---

## 8. TDE / wallet follow-up (RETAIL_DEMO tables)

CDC updates for tables in TDE-encrypted tablespaces (`RETAIL_DEMO.*` in `USERS_ENC`) are **silently not replicated** in the current remote-log-reading setup, while refresh/initial load works (JDBC path decrypts server-side). Root cause, options considered, and the chosen fix: **see §11** — TDE decryption is enabled by feeding the Oracle master key(s) to the agent via `dmconfigurets`. Proven end-to-end 2026-07-19 for **both** a brand-new pipeline and the pre-existing `GS_5073` — the agent needs the **complete master-key list** (not just the active key) and an **engine restart** for the config to load; no table re-establishment was needed (§11.5).

### 8.1 Tablespace encryption map (DB12, captured 2026-07-19)

Source: `dba_tablespaces.encrypted` + `dba_tables` for non-Oracle-maintained owners.

| Tablespace                             | TDE-encrypted | Application tables                                                     |
| -------------------------------------- | ------------- | ---------------------------------------------------------------------- |
| `USERS_ENC`                            | ✅ yes        | `RETAIL_DEMO.{CUSTOMERS, ORDERS, PRODUCTS, TEST1, TEST2, TL_SEM}`      |
| `USERS`                                | ✅ yes        | `GSIIDR.TEST`, `STUD.{TL_KURS, TL_SEM, TL_SEM_BACKUP_20260105_010001}` |
| `SHMULIK`                              | ✅ yes        | (none)                                                                 |
| `DEMO_PLAIN`                           | ❌ no         | `RETAIL_TEST.{CUSTOMERS, ORDERS, PRODUCTS}`                            |
| `CDCTEST`                              | ❌ no         | `GSIIDR.CDCTEST`                                                       |
| `SYSTEM`, `SYSAUX`, `TEMP`, `UNDOTBS1` | ❌ no         | (Oracle-internal only)                                                 |

Notes:

- **`USERS` is TDE-encrypted too**, not just `USERS_ENC` — the silent-CDC problem in this section also applies to `STUD.*` and `GSIIDR.TEST`. The only CDC-safe (plaintext-redo) application tables are those in `DEMO_PLAIN` and `CDCTEST`; pick from these for E2E replication tests (§5 used `RETAIL_TEST.PRODUCTS` for exactly this reason).
- **TDE keystore:** `WALLET_ROOT=/data/install/ORACLEDB/admin/DB12/wallet`, keystore in `<root>/tde/` — a **password keystore with no auto-login** (`ewallet.p12` only, no `cwallet.sso`; several timestamped `ewallet_*.p12` backups from 2023-2026 sit alongside, plus a `tde2/` sibling dir). Consequence: after every DB restart `V$ENCRYPTION_WALLET.STATUS = CLOSED` until someone runs `ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY <pw>;` — and while it is closed, even server-side access to TDE tables fails (`ORA-28365: wallet is not open`), independent of IIDR. Check status with `select status, wallet_type from v$encryption_wallet;`. Note this interacts with the auto-start unit from §10: a reboot now brings the DB up automatically, but the keystore stays closed. **Superseded 2026-07-19:** a LOCAL auto-login keystore (`cwallet.sso`) was created (§11.1) — the wallet now opens by itself after restart (`WALLET_TYPE=LOCAL_AUTOLOGIN`), closing this gap.

### 8.2 Wallet verification on the Oracle agent (10.0.4.215) — checked 2026-07-19

Question: is the IIDR Oracle agent configured with a wallet so it can decrypt TDE redo? **Answer: no — and this build has no place to configure one.** Every layer checked:

| # | Check | How | Result |
|---|-------|-----|--------|
| 1 | Engine system parameters | `bin/dmset -I ORACLE` (lists set params) | 5 params, none wallet/TDE-related |
| 2 | Wallet files on agent host | `find /giga/iidr /mnt /cdc -name "cwallet.sso*" -o -name "ewallet*"` | none |
| 3 | Instance configuration | `dmexportconfiguration` XML, grep wallet/tde | no such fields |
| 4 | Engine code capability | `unzip -p ts.jar\|replication-core-oracle.jar \| strings \| grep -i wallet` | zero matches |
| 5 | Native redo reader capability | `strings liboraclenativeapi.so \| grep -ic wallet` | zero matches |
| 6 | JDBC driver | same on `CIoracle-6.0.0.000977.jar` | has SSO-wallet classes — but that is TCPS/SSL support (§9), not TDE |
| 7 | DB-side keystore | `select status from v$encryption_wallet` on DB12 | `CLOSED` (see 8.1 note) |

**Implication:** with CDC 11.4.0.4-5672 reading redo remotely over NFS, TDE redo is undecryptable by design of this install — there is no wallet parameter, no wallet code in the Java engine, and none in the native parser. This is the mechanical root cause behind the silent non-replication of `USERS`/`USERS_ENC` tables. The JDBC refresh path works only because the DB server decrypts (and only while the DB keystore is OPEN).

**Superseded 2026-07-19:** the conclusion "no place to configure decryption" was wrong — the agent takes the Oracle **master key value(s)** directly (not a wallet) via `dmconfigurets` → Edit instance → option 6 (*Specify Database Parameters*) → `replicate encrypted Columns/Tables = y` → `Enter oracle database Master Key:`. See §11 for the full setup and proof.

**Definitive functional test** (run whenever the setup is changed): with the plaintext control pipeline running (pl_josh / `GS_359` / `RETAIL_TEST.PRODUCTS`) and the TDE pipeline running (`GS_5073` / `RETAIL_DEMO.ORDERS`), issue one committed UPDATE against each table. Control row must reach the space in ≤10 s (§5); if the TDE row does not arrive while the control does, wallet/TDE decryption is broken — expect **no error anywhere** (engine events and trace stay clean; the change is simply skipped).

---

## 9. Current state & rollback

**State after this ticket:**

- AS ↔ clients: **TLS 1.2** (verified by capture).
- Oracle engine ↔ Kafka engine control + data channels: **TLSv1.3 / TLS_AES_256_GCM_SHA384** (engine events 1579/1581 + capture), profile `GSTLS`, policy `ENABLED`, on both engines.
- E2E over the encrypted channel: **≤ 10 s** Oracle→space (non-TDE table).
- Still plaintext (tracked separately): Kafka engine → brokers :9092 (needs broker SSL listeners + `security.protocol=SSL` in `kafkaproducer.properties`); Oracle SQL*Net :1521 (DB already exposes an unused TCPS :2484 endpoint — one `tnsnames.ora` change once client wallets are sorted).

**Rollback (per engine):** stop engine → restore `conf.bak-20260715` over `instance/<INST>/conf` → restore `tls.properties.bak-20260715` → start engine. For the AS: comment out `enableTLS`/keystore lines in `tls.properties` and restart. To soften only the channel policy without rollback: rerun Task 3.4 with `-e DISABLED` (or `ENABLED`) and bounce the engine.

**AS backup (added 2026-07-20):** the AS itself had no file backup until now (unlike the two engines). On `10.0.4.143`, `/giga/iidr/as/` — a plain-file cold copy, no service stop needed (nothing here is a live WAL-backed DB like the engines' `instance/<INST>/conf/`):
```bash
cd /giga/iidr/as
cp -p tls.properties tls.properties.bak-20260720
cp -a conf conf.bak-20260720   # JVM launch args (*.vmargs), static since install
cp -a data data.bak-20260720   # real state: destination.ini, registered datastores (agents/KAFKA, agents/ORACLE), users/admin
```
Restore: stop `iidr_as_inst` → `rm -rf conf data && cp -a conf.bak-20260720 conf && cp -a data.bak-20260720 data`, `rm -f tls.properties && cp -p tls.properties.bak-20260720 tls.properties` → start.

## 10. Oracle DB auto-start (added 2026-07-19)

A reboot of the DB host `10.0.1.129` used to take down the whole stack: Oracle has no native auto-start (the `:Y` flag in `/etc/oratab` does nothing by itself), the IIDR Oracle engine crash-loops at startup on `Connection refused 10.0.1.129:1521` and never binds :11001, di-subscription-manager spams `ERR2225`, and every pipeline shows `SUBSCRIPTION_NOT_FOUND` — misleading, since nothing is wrong with the pipelines or TLS.

Fixed with `/etc/systemd/system/oracle-db.service` on 10.0.1.129 (enabled): oneshot + `RemainAfterExit`, runs `dbstart $ORACLE_HOME` / `dbshut $ORACLE_HOME` as `gsods` with `ORACLE_HOME=/data/install/ORACLEDB`, `ORACLE_SID=DB12` (passing `$ORACLE_HOME` as arg makes dbstart manage the listener too). After DB recovery the engine self-heals via its systemd restart loop within ~1 min; subscriptions reappear on the next di-subscription-manager poll.

## 11. TDE redo decryption via master key — enabled and proven (2026-07-19)

The fix for §8's silent non-replication. The agent does not use a wallet; it takes the **master key value(s)** extracted from the DB keystore and decrypts TDE redo itself. Configuration alone is *not* sufficient — the capture context matters (§11.4/11.5).

### 11.1 DB side (10.0.1.129, DB12 — non-CDB)

1. **Open the keystore** (was `CLOSED` after every restart): `ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "<pw>";`
2. **Create LOCAL auto-login** so it survives restarts: `ADMINISTER KEY MANAGEMENT CREATE LOCAL AUTO_LOGIN KEYSTORE FROM KEYSTORE '/data/install/ORACLEDB/admin/DB12/wallet/tde' IDENTIFIED BY "<pw>";` → verified: after `SET KEYSTORE CLOSE` it reopens as `LOCAL_AUTOLOGIN`. Together with §10, a reboot now needs no manual steps.
3. **Master key rotation** (done as part of the clean-slate test): `ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY "<pw>" WITH BACKUP;` (`FORCE KEYSTORE` needed while auto-login is active). `V$ENCRYPTION_KEYS` went 5 → 6 keys; **on this 19c install the `cwallet.sso` is rewritten in the same operation** (mtime = rotation time), so no auto-login regeneration is needed — an explicit `CREATE LOCAL AUTO_LOGIN` afterwards fails with `ORA-46630` (already exists), harmless.
4. **Extract key values** for the agent (one entry per key in `V$ENCRYPTION_KEYS`):

```bash
echo <pw> | $ORACLE_HOME/bin/mkstore -wrl /data/install/ORACLEDB/admin/DB12/wallet/tde \
  -viewEntry ORACLE.SECURITY.DB.ENCRYPTION.<KEY_ID>
```

### 11.2 Agent side (10.0.4.215) — `dmconfigurets` procedure

Order of operations (engine must be stopped; config lives in the instance metadata store):

1. Stop `di-subscription-manager-iidr` on 10.0.4.143 (`systemctl stop` reports state `failed` from a non-zero ExecStop — process is stopped, ignore).
2. Stop `iidr_oracle_inst` on 10.0.4.215; verify `pgrep -f dmts64-jav[a]` is empty.
3. Back up `instance/ORACLE/conf` (`cp -a conf conf.bak-<date>`).
4. `bin/dmconfigurets` (interactive TTY only — drive with expect if scripting): main menu `3` (Edit) → instance `1` (ORACLE) → **option `6` Specify Database Parameters** → ENTER through ORACLE_HOME / TNSNAMES / TNS `DB12` / advanced `n` / kerberos `n` / external-secret `n` → username `gsiidr`, password **must be typed** (blank ⇒ misleading "Kerberos Authentication is not supported" error) → `replicate encrypted Columns/Tables (y/n)` = **`y`** → `Enter oracle database Master Key:` = **comma-separated list of mkstore values** (all keys; whichever wraps a given tablespace must be present) → back at edit menu → `9` Save → "Changes saved successfully".
5. Start the engine (wait for :11001), start di-subscription-manager (wait for :6082).

Expect-scripting gotchas: the bundled expect rejects inline `(?i)` regex flags (use `-nocase`); the post-save main-menu exit is option `6`, not `0`.

### 11.3 Result on the PRE-EXISTING subscription — still fails

With decryption configured (tried with the single active key, then with **all 4** historical keys), every committed change to `RETAIL_DEMO.ORDERS` (`USERS_ENC`, subscription `GS_5073`) triggers redo-scraper **Event ID 2919, `ohrsparse.c` line 668 "Internal error, internal code 0"** (SHAREDSCRAPE LOG READER) and the change is dropped — nothing reaches Kafka; the plaintext control (`RETAIL_TEST.PRODUCTS`, `GS_359`) replicates in ~10 s throughout. Behavior *changed* from silent-skip (`encrypted=n`) to attempt-and-error (`encrypted=y`), proving the config took effect.

**Caveat (added after §11.5's discriminating test):** the "identical error with all 4 keys" observation is suspect — the engine only loads `dmconfigurets` changes at startup, and it is not certain the engine was restarted between saving the 4-key config and that retest. The single-key failure is solid; the 4-key failure may have actually been the 1-key config still running.

### 11.4 Clean-slate proof — TDE CDC works when everything is created after decryption is on

| Step | What | Result |
|------|------|--------|
| 1 | Rotate master key (§11.1.3) → new active key #6 | `V$ENCRYPTION_KEYS` 5→6 |
| 2 | `CREATE TABLESPACE TDE_PROBE_TS ... ENCRYPTION USING 'AES256' DEFAULT STORAGE(ENCRYPT)` (OMF) | `ENCRYPTED=YES, AES256` |
| 3 | `GSIIDR.TDE_PROBE` (ID PK, VAL, UPDATED) on it + `SUPPLEMENTAL LOG DATA (ALL) COLUMNS` + 2 seed rows (needed `ALTER USER GSIIDR QUOTA UNLIMITED ON TDE_PROBE_TS` — first insert hit `ORA-01950`) | rows 1,2 |
| 4 | Reconfigure agent (§11.2) with **new key + 4 old keys** | saved OK |
| 5 | Subscription `TDEPROBE` via di-subscription-manager (`POST /api/v1/ORACLE/subscriptions/` body `{"name":"TDEPROBE"}`; add table `{"schema":"GSIIDR","table":"TDE_PROBE"}`; start with body `{}`) | `MIRROR_CONTINUOUS`, refresh delivered both rows to topic |
| 6 | **Mirror test**: UPDATE + INSERT + `ALTER SYSTEM SWITCH LOGFILE` | **both in Kafka in ~8 s** (topic offset 2→4), UPDATE carries full before-image (`B_*` fields) — decrypted from redo |
| 7 | Pipeline `pl_tdeprobe` via DI-manager REST (`POST /api/v1/pipeline/` `{"name","sorName":"ORACLE","spaceName":"dih-tau-space","dataFormat":"IIDR"}` → `add_tables` `{"sourceSchema":"GSIIDR","sourceTables":["TDE_PROBE"]}` → `start` with the §5 body). DI-manager auto-created its own fresh subscription `GS_3527` (ignores the hand-made one); `TDEPROBE` sub stopped as redundant | CDC job RUNNING + Mirror Continuous |
| 8 | **E2E**: UPDATE ID=2 → `e2e-final-2`, INSERT ID=4, log switch | **in `dih-tau-space` (type `TDE_PROBE`) in ~9 s** |
| 9 | Engine trace during all of the above | **0** ohrsparse/2919 events |

### 11.5 Root cause & remediation — settled by the discriminating test (2026-07-19)

After §11.4, a discriminating update was run against the **old, untouched** pipeline: `UPDATE RETAIL_DEMO.ORDERS SET ORDER_STATUS='DISCRIM_A', ORDER_TOTAL=111222 WHERE ORDER_ID=7` + log switch. **It replicated in ~10 s** — Kafka `GS_5073` offset 20→21, message carried the decrypted **before-image** (`B_ORDER_STATUS: TDE_CDC_OK` — a value that only ever existed inside the encrypted table, from the earlier dropped update), space applied it, zero trace errors. **No table re-add, no subscription recreation.**

**Settled root cause:** the agent needs the **complete master-key list** — every key that wraps any replicated tablespace's encryption key, not just the currently-active one (`USERS_ENC` predates the last rotations, so its tablespace key is wrapped by an *older* master key; the single-active-key config could not decrypt it). Partial key list ⇒ `ohrsparse.c:668 "Internal error"` — IIDR's opaque way of saying *wrong/missing decryption key*. Two pitfalls that muddied earlier diagnosis: (1) **the config only loads at engine start** — always restart `iidr_oracle_inst` after `dmconfigurets` changes; (2) the earlier "stale capture context" hypothesis is **refuted** — old capture contexts work fine once the key list is complete.

**Reconciling rows that were dropped during the broken period** (space had stale values for ORDERS 7/8/9):

- A "touch update" (`SET col = col`) does **not** work — Oracle writes the redo, the agent decrypts it, but IIDR **suppresses no-op updates** (before-image = after-image ⇒ no Kafka message, no error).
- The working method — **table refresh via di-subscription-manager** (`POST /api/v1/ORACLE/subscriptions/GS_5073/refresh`, body `{"schema":"RETAIL_DEMO","table":"ORDERS","forceRefresh":true}`). Under the hood it runs chcclp `readd replication table` + regenerates the mapping, so it **requires the subscription to be stopped first** — on a running subscription it fails HTTP 500 with `ERR2316: Replication must be stopped before the subscription's configuration can be changed` (visible only in the service log at `/giga/di-subscription-manager/latest-di-subscription-manager/logs/di-subscription-manager-iidr.log`). Sequence: `POST .../stop {}` → wait `INACTIVE` → `POST .../refresh` → `POST .../start {}`. On start it snapshots the whole table (`RR` records; offset 21→32 = 11 rows) then resumes mirror. Space matched the DB on the first poll (~8 s after mirror resumed). DB untouched.

(Side benefit: since the refresh endpoint internally *re-adds* the table, it doubles as the capture-context re-establishment tool if that is ever actually needed.)

### 11.6 End state (cleanup decision: keep as TDE canary, 2026-07-19)

**Kept — the permanent TDE-CDC canary** (parallel to pl_josh's plaintext canary, §8.2's functional test now covers both):

- Pipeline `pl_tdeprobe` (id `c6034a57-2671-41b4-936a-6c93d786eca9`) + auto-created subscription `GS_3527` + Kafka topics `GS_3527`, `KAFKA-GS_3527-commitstream` — running, Mirror Continuous.
- DB12: tablespace `TDE_PROBE_TS` (OMF datafile under `/data/install/OracleDB19/oradata`), table `GSIIDR.TDE_PROBE` (4 rows, ALL-columns supplemental logging, `GSIIDR` quota). Canary test: `UPDATE GSIIDR.TDE_PROBE SET VAL='<marker>', UPDATED=SYSTIMESTAMP WHERE ID=1;` + commit + `ALTER SYSTEM SWITCH LOGFILE;` → expect the marker in `dih-tau-space` type `TDE_PROBE` within ~10 s.
- Master key #6 stays active (rotation is legitimate — old keys remain usable in the keystore and in the agent's key list; **never** attempt to "revert" a rotation).
- Agent conf backups on 10.0.4.215: `conf.bak-20260715`, `conf.bak-20260719-tde`, `conf.bak-20260719-tde-1key`, `conf.bak-20260719-rotate` (pre-5-key-list; the *live* conf is the 5-key config — restoring any backup loses TDE keys).

**Deleted** (redundant hand-made proof artifacts): subscription `TDEPROBE` (`DELETE /api/v1/ORACLE/subscriptions/TDEPROBE` — also removed `/giga/iidr/kafka/instance/KAFKA/conf/TDEPROBE.properties`), Kafka topics `TDEPROBE` and the orphaned `KAFKA-TDEPROBE-commitstream`.
