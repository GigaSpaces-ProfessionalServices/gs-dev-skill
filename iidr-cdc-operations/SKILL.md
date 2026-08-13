---
name: iidr-cdc-operations
description: >
  Field-validated operational guidance for IBM InfoSphere Data Replication (IIDR) CDC engines
  replicating from Oracle sources. Covers encrypting IIDR component-to-component traffic with TLS
  (Access Server and engine-to-engine data channel), configuring an IIDR Oracle CDC agent to decrypt
  TDE-encrypted redo, and handling source DDL/schema changes via mirror_on_ddl_operation. Use when the
  user mentions IIDR, InfoSphere Data Replication, CDC engine, Access Server, chcclp,
  di-subscription-manager, dmset, dmcreateencryptionprofile, dmconfigurets, TDE tablespace, Oracle
  master key rotation, mirror_on_ddl_operation, subscription refresh, or a stalled/failed IIDR CDC
  subscription.
license: MIT
metadata:
  author: GigaSpaces Technologies, Inc.
  version: 1.0.0
---

# IIDR CDC Operations – Security & Resiliency Skill

These notes are field-validated operational procedures for IBM InfoSphere Data Replication (IIDR)
CDC engines, gathered from real troubleshooting sessions rather than the official docs. Where a
reference file's notes disagree with IBM documentation, trust the tested behavior recorded here and
say so explicitly — several of the pitfalls below exist precisely because the docs were wrong or
ambiguous on that point.

**MANDATORY**: Use the `Read` tool to read the relevant reference file(s) below **before answering
any question or proposing a procedure**. Select the file(s) using the Quick Decision Guide, then read
them using paths relative to this skill's own directory (`references/<file>.md`).

## Reference Files

| Reference Path | Covers |
|------|--------|
| `references/skill-iidr-component-tls.md` | Encrypting IIDR-internal traffic: Access Server TLS (`tls.properties`), engine-to-engine data-channel encryption profiles (`dmcreateencryptionprofile`), verification via event log/packet capture, rollback |
| `references/skill-oracle-tde-cdc.md` | Encrypting an Oracle table with TDE and configuring the IIDR Oracle CDC agent (`dmconfigurets`) to decrypt TDE redo, master-key extraction and rotation, troubleshooting dropped/missing decryption |
| `references/skill-iidr-ddl-resiliency.md` | Controlling subscription behavior on source-DDL changes via `mirror_on_ddl_operation` (STOP/IGNORE/RESILIENT), recovering a subscription that failed on DDL |

## Quick Decision Guide

```
User wants to...
  ├── Encrypt AS↔client or engine↔engine traffic       → skill-iidr-component-tls.md
  ├── Put a source table on TDE / fix TDE-related CDC   → skill-oracle-tde-cdc.md
  │   drops or "wallet not open" errors
  ├── Decide what happens when a replicated table's      → skill-iidr-ddl-resiliency.md
  │   schema changes, or recover a DDL-FAILED subscription
  └── Diagnose a stalled/failed subscription generally   → check all three Pitfalls indexes
```

## Scope note

`skill-iidr-component-tls.md` covers the **IIDR-internal** legs only (Access Server ↔ clients,
CDC engine ↔ CDC engine). It does **not** cover encrypting the Kafka message bus itself or Oracle
SQL*Net — those are separate efforts. Don't imply this skill secures a pipeline end-to-end unless
all three legs have actually been addressed.

## Cross-cutting operational rules

These patterns recur across all three reference files — apply them regardless of which one you're
following:

1. **Config changes need an engine restart.** Editing `tls.properties`, running `dmset`, or saving
   `dmconfigurets` changes does not take effect until the instance is stopped and started. Diagnosing
   "it didn't work" without checking for a missing restart is the most common mistake in this domain.
2. **Stop di-subscription-manager first, start it last.** It auto-restarts idle subscriptions every
   ~15s and will fight any maintenance window if left running.
3. **Verify the engine JVM actually died** after `systemctl stop` (`pgrep -f 'dmts64-jav[a]'`) — the
   service manager has been observed to report success while the process is still running.
4. **`dm*` CLI tools can lie about success.** Exit code 0 doesn't guarantee no error was printed, and
   vice versa. Judge success by the tool's output text and by re-verifying the actual state (event log,
   re-exported config), not by exit code or a stale local file like `tsprop`.
5. **Interactive `dm*` tools require a real TTY.** Script them with `expect` for automation.
6. **Run `dm*` tools as the IIDR service account, not root.**
7. **Prefer tested behavior over the reference docs when they conflict.** Both the TDE and DDL-resiliency
   files record cases where IBM's documentation was incomplete or wrong for the validated build/version —
   note the discrepancy rather than silently trusting either source.

## Troubleshooting

Each reference file ends with its own **Pitfalls index** or **Troubleshooting** table (symptom →
cause → fix). Check the table in the relevant file before improvising a diagnosis — most failure modes
in this domain (silent redo-scraper starvation, dropped TDE rows, DDL-FAILED subscriptions) have a
known, non-obvious cause already documented there.
