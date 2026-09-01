# Skill: Encrypt an Oracle table (TDE) and configure IIDR for CDC against it

## When to use this

Someone needs to (a) put an Oracle table on an encrypted (TDE) tablespace, and/or (b) get an IIDR Oracle
CDC agent to correctly capture changes on tables that are already TDE-encrypted. Validated on
IIDR 11.4.0.5 (build master_5726) against Oracle 19c (non-CDB); should generalize to similar versions.

**Core fact to internalize:** IIDR reads Oracle's redo logs directly from disk, not through SQL. For TDE
tablespaces those redo records are encrypted, so the agent must be given the Oracle master key value(s) to
decrypt them itself (there is no "wallet" config on the agent side — it's master keys, passed as strings).

---

## Part A — Create/verify an encrypted Oracle tablespace + table

1. Confirm (or create) a keystore/wallet and that it's OPEN:
   ```sql
   SELECT status, wallet_type FROM v$encryption_wallet;
   -- if CLOSED:
   ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "<keystore_pw>";
   ```
2. Make it survive restarts (LOCAL auto-login keystore) — do this once per DB:
   ```sql
   ADMINISTER KEY MANAGEMENT CREATE LOCAL AUTO_LOGIN KEYSTORE
     FROM KEYSTORE '<wallet_dir>' IDENTIFIED BY "<keystore_pw>";
   ```
   Verify it took: stop/start the instance (or `SET KEYSTORE CLOSE` then check) — status should come back as
   `LOCAL_AUTOLOGIN`, not `CLOSED`.
3. Create the encrypted tablespace:
   ```sql
   CREATE TABLESPACE my_encrypted_ts
     DATAFILE SIZE 50M AUTOEXTEND ON NEXT 10M MAXSIZE 500M
     ENCRYPTION USING 'AES256' DEFAULT STORAGE(ENCRYPT);
   ```
   Verify: `SELECT tablespace_name, encrypted FROM dba_tablespaces WHERE tablespace_name='MY_ENCRYPTED_TS';`
4. Create the table on it, with **ALL-columns supplemental logging** (IIDR needs full row images to build
   before/after images for CDC):
   ```sql
   CREATE TABLE my_schema.my_table (...) TABLESPACE my_encrypted_ts;
   ALTER TABLE my_schema.my_table ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ```
   Grant tablespace quota to the table owner **before** the first insert, or you'll hit `ORA-01950`:
   ```sql
   ALTER USER my_schema QUOTA UNLIMITED ON my_encrypted_ts;
   ```
5. If master keys will ever be **rotated** on this DB, note it — see Part C, "Ongoing rule."

---

## Part B — Extract the master key value(s) the agent will need

List every key in the keystore (rotations leave old keys behind — you may need more than the active one, see
the note in Part C):

```sql
SELECT key_id, activation_time FROM v$encryption_keys ORDER BY activation_time;
```

For each `key_id`, extract its base64 value:

```bash
echo <keystore_pw> | $ORACLE_HOME/bin/mkstore -wrl <wallet_dir> \
  -viewEntry ORACLE.SECURITY.DB.ENCRYPTION.<key_id>
```

Collect every value you'll need into a comma-separated list — this is what goes into the agent config next.

---

## Part C — Configure the IIDR Oracle agent to decrypt TDE redo

Run as `gsods` (or the IIDR service account) on the Oracle CDC engine host. The engine must be **stopped**
while editing.

1. Stop dependents first, then the engine:
   ```bash
   systemctl stop di-subscription-manager-iidr   # on the AS/subscription-manager host
   systemctl stop iidr_oracle_inst                # on the engine host
   pgrep -f 'dmts64-jav[a]'                        # must return nothing before continuing
   ```
2. Back up the instance config (cheap insurance):
   ```bash
   cd <engine-install>/instance/<INST>
   cp -a conf conf.bak-$(date +%Y%m%d)
   ```
3. Run `bin/dmconfigurets` (interactive TTY tool — script it with `expect` if automating):
   - Main menu → `3` (Edit an Instance) → select the ORACLE instance
   - `6` (Specify Database Parameters)
   - Enter through ORACLE_HOME / TNSNAMES / TNS name / advanced params (n) / kerberos (n) / external
     secret store (n)
   - Username: your DB user. **Password must be typed** — leaving it blank makes the tool assume Kerberos
     and error with a misleading "Kerberos Authentication is not supported" message.
   - `Select y to replicate encrypted Columns/Tables` → **y**
   - `Enter oracle database Master Key:` → paste the **comma-separated list of ALL key values** from Part B
   - Back at the edit menu → `9` (Save) → expect "Changes saved successfully."
4. Start the engine, then di-subscription-manager, and wait for their ports to come back
   (Oracle engine typically :11001; subscription-manager typically :6082).

**⚠️ The config only takes effect when the engine (re)starts.** If you change the master-key list without
restarting the engine, you're still running the old config and will misdiagnose the failure as something
else.

**Ongoing rule:** every time the master key is rotated on the DB, you must add the new key value to this
comma-separated list and restart the engine — old tablespaces stay wrapped under old keys, so removing an
old key from the list, or forgetting to add a new one, breaks decryption for whichever tablespaces depend on
the missing key.

---

## Verification

Update a row in the TDE table, force a fast redo read, and check the target (space/Kafka/etc.) picks it up:

```sql
UPDATE my_schema.my_table SET some_col = 'CDC_TEST_1' WHERE id = 1;
COMMIT;
ALTER SYSTEM SWITCH LOGFILE;   -- forces the CDC engine to read this redo promptly
```

Expect the change to arrive at the target within ~10 seconds. Check the engine's own trace log for errors:

```bash
grep -c -E 'ohrsparse|2919|Internal error' instance/ORACLE/log/trace_dmts_*.log   # expect 0
```

If you have a working non-TDE (plaintext) pipeline as a control, update it too — if the plaintext row
arrives but the TDE row doesn't, and there's no error anywhere, that's the signature of missing/incomplete
decryption keys.

---

## Troubleshooting

| Symptom | Meaning | Fix |
|---|---|---|
| `ohrsparse.c:668` error (event 2919), change silently dropped | Missing/wrong master key for that tablespace | Give the agent ALL keys, not just the active one (Part B/C) |
| `ORA-28365: wallet is not open` | DB keystore closed | `SET KEYSTORE OPEN`; add LOCAL auto-login (Part A step 2) |
| `ORA-01950` on first insert | Table owner has no tablespace quota | `ALTER USER <owner> QUOTA UNLIMITED ON <tablespace>` |
| "Kerberos Authentication is not supported" | Password field left blank in dmconfigurets | Re-enter and actually type the password |
| Config change has no effect | Engine wasn't restarted after saving | Stop/start `iidr_oracle_inst` |

---

## Re-syncing rows that were dropped while decryption was broken

Changes dropped during an outage are gone from the delivery pipe forever — fixing the config only affects
*future* changes. To backfill the target with current DB values, force a table refresh (re-snapshot):

```bash
# must stop the subscription first, or you get HTTP 500 / ERR2316
curl -X POST '<subscription-manager-url>/api/v1/<source>/subscriptions/<sub>/stop' \
     -H 'Content-Type: application/json' -d '{}'

# wait for state == INACTIVE, then:
curl -X POST '<subscription-manager-url>/api/v1/<source>/subscriptions/<sub>/refresh' \
     -H 'Content-Type: application/json' \
     -d '{"schema":"MY_SCHEMA","table":"MY_TABLE","forceRefresh":true}'

curl -X POST '<subscription-manager-url>/api/v1/<source>/subscriptions/<sub>/start' \
     -H 'Content-Type: application/json' -d '{}'
```

On start, IIDR snapshots the whole table (SELECT-based, so it doesn't need redo decryption) and then resumes
live CDC. Note: a plain `UPDATE t SET col = col` does **not** work as a re-sync trick — IIDR suppresses
no-op updates (no before/after change ⇒ no message sent).
