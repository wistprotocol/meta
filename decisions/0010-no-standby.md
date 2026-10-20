# ADR-0010: no standby for a Log

**Status:** accepted · **Date:** 2026-10-20

## Context

The availability plan kept a passive standby as a conditional investment: a second server receiving the primary's store, pending work, artifacts and signing state, promoted on primary loss. Its purpose was this one Log's freshness during a host failure.

## Decision

No standby is run. A host failure is recovered by restoring the latest backup onto the repaired or a fresh machine; read availability during the outage is a Mirror's job; freshness beyond one operator's uptime comes from other operators' Logs (ADR-0006, Multiple Logs).

## Rationale

- Exactly one process may sign a Log, and two Checkpoints signed for one height with different contents end the Log (WIST-3 §5). A standby exists to move the signing key automatically, so every split-brain, stale-lease or clock fault in the takeover is that failure. The trade is a tolerable availability loss against an intolerable integrity loss, and the takeover would be the most dangerous code in the suite.
- The protocol already budgets an outage: a lost Ping is recovered by the baseline poll within `baseline_poll_seconds` (24 h), and the capacity targets require an outage of *h* hours to drain within *h* hours (ADR-0005). A primary restored within a day loses no publication.
- Restore is proven: a backup restored onto a fresh host served the byte-identical head Checkpoint under the primary's Log identity in seconds. Hardware repair or a replacement machine takes hours, inside the budget above.
- A standby doubles the primary's cost for no ingestion capacity (ADR-0009).

## Consequences

- The active/passive replication, takeover and fencing work is closed, not deferred. Reopening it requires a new record with a takeover design whose exclusive-signing guarantee is demonstrated before any automation.
- Backups must leave the host: the primary's backups are copied to separate storage on a schedule, and restore stays rehearsed.
- The availability path is Mirrors plus restore. Recovery of signing after a restore keeps ADR-0006's rule: every published Epoch and Checkpoint is recovered byte for byte, and the pending publication record is reconciled, before the restored primary signs.
