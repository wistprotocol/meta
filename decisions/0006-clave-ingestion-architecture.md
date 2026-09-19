# ADR-0006: Clave ingestion architecture

**Status:** accepted, amended by [ADR-0007](0007-signed-publications-and-consumer-trust.md) (2026-09-16: audit duties dropped from the stages); amended by Addendum (2026-09-18: the protocol term Block renamed Epoch); amended by Addendum (2026-09-19: sixteen partitions per store and fencing tokens on partition and sealer leases) · **Date:** 2026-09-16

## Context

Clave today runs one process over one SQLite store. A Ping or the
minute-cadence baseline pass runs a whole pull under the store mutex:
Declaration refresh, Feed walk, Delta and Payload fetches, verification,
admission and queueing, with the full Block history replayed to derive the
parameter schedule and again for every Delta whose predecessor is sealed.
A seal replays the history a third time, rebuilds derived governance state
from genesis and rebuilds the complete Snapshot, and writes the Block and
Checkpoint files before the store commits. ADR-0005 measured the result:
at 10 000 sealed entries an updated Delta costs about 1.5 s to admit,
Pings queue behind the pull that holds the store, an empty seal costs
0.24 ms per record, and a Consumer applies at most 90 000 Deltas per hour.
Those terms, not fetch throughput, bound the aggregator, and every one of
them grows with the sealed history. This record fixes the architecture the
fetch bounds, concurrency, recovery, durable ingestion and incremental
Snapshot work implement, so that each of those lands inside stage
contracts that already support partitioning and workers.

## Decision

### Stages and their contracts

Ingestion is six stages with explicit inputs and results. A stage never
reads another stage's working memory; it reads persisted state by
reference and returns a result the next stage persists.

1. **Scheduling.** Input: Pings (domain, receipt instant), due-time rows
   (`domain`, `due_at`, `reason` ∈ {ping, baseline, resume, retry}) and
   the per-domain admission state (sanction level, quota, budget spent
   today, walk cursor). Result: a bounded batch of pull tasks, at most one
   in flight per domain, ordered by due time with baseline duties never
   starved by Pings. The scheduler owns the due-time index; a Ping only
   moves a due time forward. It reads the store in one short transaction
   and never holds it while a task runs.
2. **Fetch.** Input: a pull task with the domain's Canonical Host, scheme
   policy, `subdomain_scope` snapshot, remaining byte budget and walk
   cursor. Result: the fetched bytes of the Declaration, Feed pages,
   Registry Updates, Deltas and Payloads with their URLs, sizes and
   instants, or a transport failure per object. Fetch enforces the
   destination policy and the per-object byte limits while reading, before
   buffering or parsing, and stops at the budget. It holds no store
   connection.
3. **State-independent verification.** Input: fetched bytes plus the
   Key Sets, accepted schedule and caps the task was issued with (stage 1
   attaches them as versioned references: Declaration hash and height,
   schedule position, cap values). Result: per object, canonical bytes,
   the WIST-1/WIST-2 diagnostics that need no chain state (JSON
   eligibility, field validation, version support, signature under the
   referenced Key Set, Payload commitment and caps, clock allowance against
   the issued clock) and the derived facts admission needs (Delta ID, URL,
   `prev`, `observed_at`, Payload bytes). It holds no store connection and
   is idempotent.
4. **Stateful admission.** Input: verified objects with the references
   they were verified against. Result: accepted, queued, rejected or
   deferred per object, persisted in one short transaction per domain
   batch. Admission revalidates the references: the domain's current
   Declaration hash and recovery-window state, the URL chain tip, the
   quota and sanction level, the schedule position. A stale reference
   re-issues the task rather than admitting under old state. Predecessor
   and chain-tip lookups read indexed persisted state (`url_tips`,
   accepted and sealed Delta indexes); admission never replays Block
   files. The parameter schedule is read from the persisted accepted
   amendments table, extended by each seal, never rederived from Blocks.
5. **Sealing.** Input: the pending entries at the cadence instant, the
   persisted schedule and the derived state as of the previous seal.
   Result: one Block and Checkpoint, committed to the store together with
   the acceptance records, the schedule extension and the derived-state
   extension, before either file is published. Sealing extends derived
   governance state from the previous seal's persisted state rather than
   replaying from genesis; full replay remains the audit path
   (`verify-history`) and the recovery path, not the sealing path.
6. **Artifact distribution.** Input: a committed Block height. Result: the
   Block and Checkpoint files published atomically (write, fsync, rename)
   and, on its own cadence, the Snapshot rebuilt or extended at a
   consistent sealed height. Publication is idempotent by height and
   byte-identical on retry; a Checkpoint is signed only over a Block
   already committed.

### Ownership, partitions and transfer

Every domain belongs to exactly one partition; a partition is the unit
of scheduling and of admission serialization. Partition ownership is a
persisted lease (`partition`, `owner`, `lease_until`) that a scheduler
instance holds while it dispatches that partition's tasks. Transfer means
the lease lapses or is released and another owner takes it; a task
already in flight completes against admission's reference revalidation
and can be re-issued safely because fetch and verification are idempotent
and admission is transactional. The initial deployment runs one
partition owned by the single process; the schema and contracts do not
change when partitions become several or owners become several
processes.

### Log-global state and exclusive signing

The following state is global to one Log and lives only in the primary's
store: the Block chain and Checkpoints, the accepted parameter schedule,
the Auditor roster and Observer registry, reputation, sanctions,
exclusions, quotas, recovery windows, Registry Update identities, and the
Delta-ID uniqueness index. Workers receive references to it and results
about it; they never hold a replica. Exactly one process holds the Log
signing key and seals; the lease that grants it is the primary's and is
never shared, so no two Blocks or Checkpoints can be signed for one
height (WIST-3 §5). A standby that takes over first recovers every
published Block and Checkpoint byte-for-byte and the pending publication
record before it may sign.

### Workers

A fetch-and-verification worker executes stages 2 and 3 for tasks the
scheduler assigns. It retains only the task's inputs, the fetched bytes
up to the task's byte bound, its results and bounded caches (Key Sets
under `keyset_cache_ttl_seconds`, the schedule and cap values it was
issued). It needs no copy of the Log, the store or the index, so its
memory is concurrency × task bound plus cache limits (ADR-0005). Results
carry the references they were computed under; admission on the primary
revalidates them.

### Transaction boundaries

- Scheduling: one read transaction to build a batch; one write per
  dispatched task to record it as in flight with a lease.
- Admission: one write transaction per domain batch covering acceptance,
  queueing, rejections, `url_tips`, budget and walk cursor.
- Sealing: one write transaction covering the Block row, acceptance
  records, schedule extension, derived-state extension and the pending
  publication record; file publication follows and is re-run from that
  record on restart.
- Snapshot: reads a consistent sealed height; writes no admission state.
- Reads by `status`, Pings and Mirrors never wait for a pull.

### Multiple Logs

Global coverage comes from several Logs, each an independent chain with
its own Log-global state (WIST-3 §7). A Publisher may Ping any number of
Logs; nothing assigns a domain to a Log at the protocol level, and no
Log-discovery mechanism exists in the suite today. Clave therefore runs
one Log per primary; an operator running several Logs runs several
primaries, each with its own partitions and workers. Consumers select the
Logs they follow by Anchor and deduplicate by Delta ID, keeping each
Log's derived state to that Log. Log discovery and selection are a
protocol question recorded for the specification, not something this
architecture assumes; changing parameters or adding Snapshot shards does
not lift the per-Log Block cap that makes this a many-Log design.

### Normative changes

None are required for the stages, partitions, workers or transaction
boundaries above; they are implementation structure under the existing
pull, sealing and Checkpoint rules. Two items are filed for the
specification rather than assumed: a Log-discovery and selection
mechanism for Consumers facing many Logs, and the genesis parameter
question ADR-0005 depends on. Snapshot production off the sealing path
uses the Snapshot the specification already allows at any sealed height.

## Consequences

The concurrency work moves fetch and verification off the store mutex
by construction, because those stages hold no connection. The fetch and
admission bounds attach to stage 2's contract. Recovery attaches to
stage 5's pending publication record and stage 6's idempotent
publication. Durable, partitionable ingestion is stage 1's due-time
index and lease table. Incremental Snapshot production is stage 6 on its
own cadence. Removing the three history-dependent costs is the first
implementation step: indexed predecessor lookup and a persisted schedule
for admission, and derived state extended per seal. Full-history replay
stays as verification and recovery, so the audit path is unchanged.

## Addendum (2026-09-18): Block renamed Epoch

The specification's draft ADR-0048 renamed the protocol term Block to
Epoch: since the single-tree Log a Block is no object, only the interval
of the Log between two consecutive Checkpoints, and key-transparency logs
call that interval an epoch. No behavior changed. Read every "Block" in this
record as "Epoch"; the ingestion stages, their order and their bounds
are unchanged.

## Addendum (2026-09-19): partition count and fencing

A store is created with sixteen partitions, and that count never changes
for the store; a domain's partition is the first eight octets of the
SHA-256 of its Canonical Host, read big-endian, modulo the count. A
single process holds all sixteen, so "one partition" above reads as
"one owner"; the fixed count lets a later owner take over part of the
domains without rehashing them. Every partition lease and the Log-wide
sealer lease carry a token that each takeover increments. A pull or a
seal runs with the token it was issued under, and every write
transaction it makes compares that token with the current one after
beginning and rolls back on a mismatch, so a former owner writes
nothing once its lease is taken, whether or not it has noticed. Leases
are renewed while their work runs, so lease duration does not bound pull
or seal duration. A serving process holds its leases under a stable
instance name, locked exclusively in its data directory for the
process's lifetime; on start it re-takes the leases recorded under that
name at once, incrementing their tokens, so a restart neither waits for
its predecessor's leases to lapse nor shares a token with it.
