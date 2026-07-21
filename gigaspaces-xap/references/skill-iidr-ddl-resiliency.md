# Skill: Handle source-DDL changes in IIDR CDC (`mirror_on_ddl_operation`)

## When to use this

A replicated source table will get a schema change (ADD / DROP / ALTER COLUMN) and you need to decide what
the IIDR CDC subscription does when it hits that DDL in the redo — fail, ignore, or ride through it. Also
covers recovering a subscription that already failed on a DDL.

Validated on IIDR 11.4.0.5 (build master_5726), Oracle 19c source, remote redo reading.

## The core setting

`mirror_on_ddl_operation` is an **instance-level** parameter on the CDC engine, set with `dmset`:

```bash
/giga/iidr/oracle/bin/dmset -I <INSTANCE> mirror_on_ddl_operation=<value>
```

**Two hard requirements:**
1. **Engine restart** — the change does not take effect until the instance is restarted
   (`sudo systemctl restart iidr_oracle_inst`, or `dmshutdown -I <INST>` + `dmts64 -I <INST>`).
2. **Hierarchy rule** — it must be set at the **instance level** first; you can't set a single
   subscription/table to a mode the instance-level parameter doesn't also allow.

### The three modes

| Value | Behavior on source DDL |
|---|---|
| `STOP` (default) | Subscription **stops immediately** when any DDL on a replicated table is seen. Manual recovery required. |
| `IGNORE` | DDL is ignored. ADD column tolerated; DROP / type-change are not. Per the reference doc it can still fail *later* on DML parsing with "latest table definition ... is newer than ... current operation" — **not tested here.** |
| `RESILIENT` | Subscription **keeps running**. New columns are silently **not** replicated until you manually map them; existing-column data of new rows keeps flowing. **Confirmed working on Oracle source (11.4.0.5).** |

> ⚠️ The reference doc `Add coulmn TAU.odt` summary table claims "RESILIENT — not applicable for Oracle
> source." That is **wrong** on this build — both the doc's own narrative (Oracle `STUD.TA_IDS`) and a direct
> env1 test (`GSIIDR.TDE_PROBE`, ADD column + insert) show RESILIENT holding the subscription up on an Oracle
> source. Trust the tested behavior, not that table cell.

## Procedure — switch an instance to RESILIENT

1. Set it: `dmset -I ORACLE mirror_on_ddl_operation=RESILIENT` (takes effect immediately in the file, but
   not in the running engine).
2. Restart the engine cleanly (see the IIDR TLS skill's maintenance-window notes — stop di-subscription-
   manager first, stop the engine, confirm `pgrep -f 'dmts64-jav[a]'` is empty, start engine, start
   di-subscription-manager). Subscriptions auto-resume ~15 s later
   (`mirror_auto_restart_interval_seconds`).
3. Verify it persisted: `dmset -I ORACLE` → the output list must show `mirror_on_ddl_operation=RESILIENT`.

## Verifying the behavior

With RESILIENT active, on the source:

```sql
ALTER TABLE myschema.mytable ADD (newcol VARCHAR2(200));
INSERT INTO myschema.mytable (id, ...) VALUES (<new id>, ...);
COMMIT;                       -- CDC only reads COMMITTED redo — no commit, nothing replicates
ALTER SYSTEM SWITCH LOGFILE;  -- makes the engine read this redo promptly
```

Expected: subscription stays `Mirror Continuous`, pipeline stays `RUNNING`; the new row lands in the target
with its **mapped** columns only — `newcol` is absent until you add it to the pipeline mapping. (SpaceDeck /
OPSUI total-count rises; the new column just isn't there yet.)

## Recovering a subscription that already failed on DDL (STOP mode)

When STOP is in effect, a DDL produces engine **event 9505** ("critical data definition (DDL) change ...
will shutdown. Please re-add the table definition with `dmreaddtable` and the `-a` option ...") and the
subscription goes to `FAILED`. Recover by re-reading the table definition, which the
di-subscription-manager `refresh` endpoint does for you:

```bash
# a FAILED subscription can be refreshed directly (it isn't "actively replicating", so no ERR2316);
# a RUNNING one must be stopped first.
curl -X POST 'http://<subman>:6082/api/v1/ORACLE/subscriptions/<sub>/refresh' \
     -H 'Content-Type: application/json' \
     -d '{"schema":"MYSCHEMA","table":"MYTABLE","forceRefresh":true}'

curl -X POST 'http://<subman>:6082/api/v1/ORACLE/subscriptions/<sub>/start' \
     -H 'Content-Type: application/json' -d '{}'
```

On start it re-snapshots the table (so rows inserted during the outage are backfilled) and resumes mirror.
The low-level equivalent is IBM's `dmreaddtable -a` from event 9505; the refresh endpoint wraps chcclp's
`readd replication table`.

## Pitfalls index

| Pitfall | Consequence | Rule |
|---|---|---|
| Setting `dmset` but not restarting the engine | Old mode still active; test "fails" confusingly | Always restart the instance after `dmset` |
| Trusting the doc's "not applicable for Oracle source" for RESILIENT | You'd needlessly avoid a working mode | It works on 11.4.0.5 Oracle source — verify, don't assume |
| Forgetting `COMMIT` on the source | Nothing replicates, looks like a failure | CDC reads committed redo only |
| Expecting the new column to appear in the target | It won't under RESILIENT | New columns need manual pipeline mapping; only the stream survival is automatic |
| `refresh` on a running subscription | HTTP 500 / `ERR2316` (real error only in the subman log) | Stop first; a FAILED sub can be refreshed directly |
| `pl_tdeprobe`/`GS_3527` = env1's TDE canary | — | It's the permanent test pipeline; safe to run these DDL tests against |
