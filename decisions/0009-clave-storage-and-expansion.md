# ADR-0009: Clave storage and expansion decision

**Status:** accepted, amended by Addendum (2026-10-20: the production primary is a dedicated AX42; cloud plans remain the homologation shapes) · **Date:** 2026-10-20 · **Amends:** [ADR-0006](0006-clave-ingestion-architecture.md)

## Context

[ADR-0005](0005-capacity-model-and-baseline.md) set the scenarios and targets, [ADR-0006](0006-clave-ingestion-architecture.md) the stages, partitions and workers that a larger deployment would run, and the qualification of 2026-10-19 ([validation record](../validation/capacity-qualification-2026-10-19.md)) measured Clave v0.2.2 on the production primary's plan and on a plan with twice its cores and memory. This record decides, from those measurements, what is retained, what is deferred and what reopens each decision, for the three expansion paths the architecture left open: a different store, fetch workers beside the primary, and a larger single server. The storage candidates were fixed before measuring: SQLite in one file with short transactions (the current store), SQLite partitioned into one file per partition plus one Log-global file, PostgreSQL 18 in a container on the host, and redb for the task queue only; MongoDB, RocksDB, MySQL and DuckDB were excluded (no ordered transactional advantage, a C++ dependency against [ADR-0004](0004-all-rust-stack.md), no advantage over PostgreSQL, an analytical engine). The candidates beyond the current store were to run only when a declared target was unmet and the measurement attributed the miss to storage rather than to replay, Snapshot production or fetch.

### What the measurements attribute

- Fetch is not the bound. At Pilot size the pulls take 157–171 s of the 3 600 s cadence; a plan with twice the cores pulled no faster.
- The seal is one process on one core. A round's seal takes about 100 s for 20 000 Items (about 5 ms per Item: the Payload check, canonicalisation and the Catalog inclusion proof of every Item, then one fsynced file per sealed Payload after the commit); the genesis seal of 208 000 Items takes 772 s. Twice the cores sealed nothing faster.
- The write lock is a SQLite write transaction (`BEGIN IMMEDIATE`, WAL, `synchronous=FULL`, 60 s busy timeout), and what holds it is mostly the aggregator's own computation inside the transaction, not the engine: at genesis the whole seal runs inside one transaction (387 s); in a round the 2–17 s hold covers one indexed read per publisher, the merge of publishers a pull changed during planning (each one reloaded from the store, parsed from JSON and spliced by scans over its whole in-memory state, 36–44 ms each), the tree extension and entry rows, a full replay of the sealed Epoch's entries to derive state, a per-Item linear search over all sealed Items (quadratic in the Epoch), and the state diff written back as re-serialised JSON. The commit is one WAL fsync. Nothing times the engine's share of the hold; the longest hold is the only figure reported.
- Status reads never take the lock and stay under 8 ms p99 through seals. Pulls block for up to the busy timeout at their next write while the seal holds the lock; no pull was rejected in the Pilot runs.
- The sealer's memory is in-process Rust data, not the store's page cache: every record of every publisher with any waiting work is loaded and parsed into about seven copies (three copies of the planning state, the history replay, the judgment replay, the list cache and the Epoch entries), about 2.5–3 KB each parsed, and in addition every seal resumes a history replay that holds every sealed record of the whole Log. Measured: an empty seal costs about 8.4 KB per sealed record of the Log (34.8 MB at 1 000 records, 110.4 MB at 10 000) plus 4.5 KB per sealed Label, and a Pilot round peaks at 4.5–4.7 GB with 208 000 sealed records and about 200 000 records belonging to publishers with waiting work, about 14 KB per such record above the Log term; genesis peaks at 5.8 GB.
- Store growth is linear and indexed on the paths that matter: records, list items, waiting URLs, Collections and held lists are keyed lookups. Three scans grow with the Log and are implementation findings, not engine limits: the parameter schedule is replayed from every Epoch row on every status request, pull and dispatch check (linear in Log height, 8 760 Epochs a year); the rejections table has no index and is scanned by status and looked up per Collection on every pull and seal; retention scans every payload duty, list item, record and waiting URL under the lock.

### Prices

Hetzner, consulted 2026-10-20, net per month, dedicated vCPU, fsn1 and nbg1: `ccx23` 4 vCPU 16 GB EUR 101.49; `ccx33` 8 vCPU 32 GB EUR 162.99; `ccx43` 16 vCPU 64 GB EUR 325.49; `ccx53` 32 vCPU 128 GB EUR 629.49; `ccx63` 48 vCPU 192 GB EUR 1 006.99. A primary IPv4 is EUR 0.60 a month and automatic backups 20 % of the server price, so the primary on `ccx23` costs EUR 122.39 a month: EUR 8.5 per million sealed Items at the tested round rate (14.4 M a month) and EUR 125 at ADR-0005's Pilot workload (0.98 M a month). All plans were in stock in fsn1, nbg1 and hel1 on that date. A rescale to another plan keeps the server and its disk, must stay within the architecture, may keep the current disk size so that a later downgrade stays possible, and cannot move to a plan with a smaller disk.

## Decision

### 1. Storage: SQLite retained; no other candidate measured

The current store is retained. The partitioned-SQLite, PostgreSQL and redb candidates are not run, because no declared target is unmet for a reason attributable to the engine: the lock is held by computation the aggregator performs inside the transaction, the memory is the sealer's own copies, the seal's wall time is single-core computation and file fsyncs after the commit, and the pulls use under 5 % of the cadence. A client/server store would add a second process, a connection pool, backup and replication procedures and a network hop per transaction without shortening any of those terms; row-level locking only pays once the aggregator's computation has left the transaction and pulls are shown to wait on each other. A partitioned SQLite store removes the single write lock only for admission, which is not waiting; and it needs the pending-publication record plus idempotent per-partition marking in place of one cross-file transaction, because a transaction over attached files is atomic per file under WAL. redb keeps the single writer and turns every relational query into hand-maintained index tables; its promotion condition (the lock held mostly by engine CPU after prepared statements, integer keys and batched statements have been measured) is not met, because the engine's share is not what holds the lock.

Remedies inside the current store come first and are ordered by what the measurements show: move the derived-state replay, the quadratic sealed-Item search and the merge's whole-state scans out of the transaction or make them linear; add per-phase timing (planning, merge, commit statements, derived-state replay, store writes, Payload publication, retention) and the engine's share of each hold, since no candidate can be compared without it; cache prepared statements for the entry rows; index rejections, pending entries by domain and records by Item; persist the parameter schedule instead of replaying it per request. None of these changes the schema's meaning or the stage contracts.

Triggers that reopen a candidate, each measured per round of the `skew` scenario at Pilot shape on the production plan after the remedies above:

- Longest write hold in a round at or above 30 s (half the busy timeout, beyond which pulls fail their write and retry at most three times; 17 s today): if per-phase timing attributes more than half of it to statement execution and commit, measure PostgreSQL first, partitioned SQLite second; if to computation, the remedy is code and no candidate runs.
- Pulls taking more than 900 s of the cadence (a quarter; 171 s today) with lock waits, not fetch or verification, the majority of a pull's time: measure partitioned SQLite first, as the candidate without operating cost, then PostgreSQL.
- Status p99 at or above 250 ms (half its bound; 8 ms today) after the parameter-schedule replay is persisted: a store-side cause is investigated before any candidate.
- A standby adopted under the availability requirements: PostgreSQL's streaming replication is compared against SQLite WAL shipping at that point, on recovery time and exclusive signing, not on throughput.

Migration path if a candidate is ever adopted: the storage boundary is the six transactional contracts of ADR-0006 (schedule a batch, record a dispatch with its lease, admit a domain batch, seal an Epoch, read a consistent sealed height, serve status and Ping reads), extracted as a trait only when the second implementation exists; the new store is populated by replaying the Log into it and comparing derived state with the SQLite store at the same height; a live store migrates by that replay plus transport of the unsealed state the Log does not carry (pending entries, cursors, budgets, leases); one implementation survives the decision and the other is deleted.

### 2. Workers: deferred, reopened by a fetch-bound round

Fetch-and-verification workers (ADR-0006, Workers) would take fetch and state-independent verification off the primary. Measured, those stages use under 5 % of the cadence while the seal, which stays on the primary by construction (exclusive signing), is the bound. Their implementation, the queue and coordination resources they need, and the primary-plus-one and primary-plus-two cost comparison are deferred. The decision reopens when a round's pulls exceed 900 s with the seal and Snapshot under 900 s, or when the aggregator's own fetching over a real network, which no run has measured, is shown to bound a round; a measurement that attributes the bound to the seal instead confirms the deferral. ADR-0006's partition, lease and fencing contracts stay as implemented (one partition owner, sixteen partitions per store) so that the deferral costs no schema change later.

### 3. Plan: the production primary stays on `ccx23`; a larger plan buys memory only, within stated limits

Twice the cores and memory (`ccx33`) sealed nothing faster at 1.6× the price, so no throughput target moves a plan decision. The sealer's memory is the only term a larger plan serves. Budgeting 75 % of a plan's memory for the sealer (the aggregator peaks near 200 MB; the rest is the system and page cache) and the measured terms above (8.4 KB per sealed record of the Log plus about 14 KB per record of every publisher with waiting work in the round, plus 4.5 KB per sealed Label), the plans hold:

| Plan | Sealer budget | Log records with no waiting work | Log records at the Pilot round's waiting term (2.85 GB) |
|---|---|---|---|
| `ccx23` | 12 GB | 1.4 M | 1.1 M (about 5 000 domains of 200 pages) |
| `ccx33` | 24 GB | 2.9 M | 2.5 M |
| `ccx43` | 48 GB | 5.7 M | 5.4 M |
| `ccx53` | 96 GB | 11.4 M | 11.1 M |
| `ccx63` | 144 GB | 17.1 M | 16.8 M |

These are estimates from two measured slopes; a seal's peak resident size is the figure that decides.

Triggers and actions:

- The sealer's peak resident size at or above 9 GB in any seal on `ccx23` (75 % of its budget; 4.7 GB in a Pilot round, 5.8 GB at genesis): move to `ccx33`; at 18 GB on `ccx33`, to `ccx43`. Each step is a memory step at 1.6× and then 2.0× the price, decided and executed before the next larger genesis-sized backfill is admitted.
- Executable migration: `clave-backup` on the running primary; apply the deployment module for a fresh server on the larger plan; `clave-restore <archive>` there (proven 2026-09-26 on rented hosts); stop the old service before the new one's first seal, because exactly one process may sign (WIST-3 §5); move the primary IPv4 and its hostname to the new server. Hetzner's in-place rescale (power off, change the plan keeping the disk size, power on) keeps the data directory and the signing host and is the cheaper path; it is used once a rehearsal on a rented shape has proven it.
- The Regional scenario (100 000 domains, 20 M records) is unmet on every plan: its Log term alone is about 168 GB, above the largest plan, and its waiting-work term about 16 GB an hour (about 5 700 publishers with waiting work per round: a seventh of a week's attestations plus the churn's skewed and unskewed domains). This is attributed to the sealer holding the whole Log's records in memory and several parsed copies of each record it plans, not to the plan or the store. The remedy is code: derive the seal's inputs from persisted derived state instead of a resident whole-Log replay, read sealed Items once by lookup, and hold one parsed copy per planned record; after it, the Regional round's term fits `ccx33`, and `ccx43` is the margin. Until that remedy is measured, no plan is chosen for Regional, and the ADR-0005 cost budget (one dedicated 4-vCPU server per 100 000 active domains) stays unmet for it.
- The seal's duration: a round's pulls plus seal plus Snapshot at or above 1 800 s (half the cadence, the headroom the backlog-recovery target needs; 210–300 s today) is a code trigger, never a plan trigger: parallelise the per-Item planning work and the Payload publication, which are independent per Item, before any cadence change, which is a Log parameter.

## Consequences

- The production primary stays one `ccx23` with the SQLite store; the deployment automation, backups and restore procedure are unchanged. The cost basis is EUR 122.39 a month.
- The Capacity expansion work of ADR-0006 (workers, partitions across hosts, storage migration) is conditional on the triggers above, not scheduled; the architecture is kept so that it needs no schema change when a trigger fires.
- The sealer's memory and the computation it performs under the write lock are the next capacity work, both code, each with a numeric trigger here; per-phase timing is a prerequisite of any later storage comparison.
- The Regional scenario is reclassified from "a larger single server is the memory path" to unmet pending the sealer remedy, with the plan table above giving the Log size each plan carries meanwhile.
- A later measurement that attributes a miss to the engine reopens the storage comparison with the candidate order stated above; the comparison then runs on rented shapes with the PostgreSQL container the qualification procedure installs, never on a development machine.

### Amendment to ADR-0006

Workers (ADR-0006, Workers) are a deferred path reopened by the fetch-bound trigger of decision 2; the stage contracts, partition leases and fencing stand unchanged. The transaction boundary for sealing ("one write transaction covering the Epoch row, acceptance records, schedule extension, derived-state extension and the pending publication record") is kept as the boundary, with the computation now performed inside it moved out under decision 1.

## Addendum (2026-10-20): the production primary is a dedicated AX42

Decision 3 chose among cloud plans. Hetzner's dedicated AX42 (8 cores with multithreading, 64 GB DDR5 ECC, two 512 GB NVMe, 1 Gbit/s, unlimited traffic; EUR 99.00 a month plus EUR 49.00 setup, hourly billed, no minimum term; FSN1 and HEL1; consulted 2026-10-20) costs the same as `ccx23` with four times its memory and a local NVMe, where the cloud disk was the slow part of the seal and Snapshot. It is therefore the production primary's plan from creation and replaces `ccx33` and `ccx43` as the memory step: at the measured terms it holds about 5.4 M Log records at the Pilot round's waiting term (a 48 GB sealer budget), and the Regional round's waiting-work term fits it with margin once the sealer stops holding the whole Log.

Consequences:

- The primary is created by a rehearsal on the machine itself: installimage with the two NVMes in software RAID 1; the install by the deployment README's manual path, since the Terraform module provisions cloud servers only; a host firewall in place of the cloud firewall; a restore of a rehearsal backup; verification from outside; one Pilot load run under a throwaway Log identity, recorded beside the 2026-10-19 figures. The machine is then reinstalled with the production Log's identity and keys and kept, so the setup fee is paid once and nothing the rehearsal signed survives.
- Backups go to a Storage Box (EUR 3.20 a month for 1 TB) through the backup script, in place of automatic cloud backups, which dedicated servers lack; that is the independent backup the plans already require.
- A hardware fault is repaired by part replacement with hours of downtime; the standby question is unchanged and remains governed by the availability requirements.
- Routine qualification stays on cloud `ccx23` shapes: the seal's compute and the memory slope transfer, and the cloud disk figures are conservative for the NVMe. A declaring run before a release rents an AX42 by the hour, about EUR 50 with the setup fee.
- The memory trigger of decision 3 stands at 75 % of the sealer budget: 36 GB on the AX42. Reaching it reopens the sealer remedy before any larger machine.

