# ADR-0005: Clave capacity model, targets and baseline procedure

**Status:** accepted, amended by [ADR-0007](0007-signed-publications-and-consumer-trust.md) (2026-09-16: audit workload terms dropped); amended by Addendum (2026-09-18: the protocol term Block renamed Epoch) · **Date:** 2026-09-16

## Context

Clave is one process over SQLite that pulls Publisher Feeds on Pings and on
a baseline poll, seals hourly Blocks, refreshes derived governance state
and rebuilds a Snapshot at every seal. The ingestion architecture, fetch
bounds, concurrency and recovery work that follows needs explicit workload
scenarios, targets and a reproducible baseline to size against, so that
capacity claims rest on measured unit costs rather than on the fixture
scale of the end-to-end test. This record fixes the scenarios, the targets,
the model and the baseline procedure. It measures nothing distributed and
no live audit; those costs stay unmeasured until the later qualification.

## Decision

### Workload scenarios

Every scenario assumes 200 URLs per active domain, a page body under the
Payload caps (extract 32 KiB, summary 2 KiB, links 4 KiB), one Delta per
changed page, and one `attest` per unchanged page per attestation interval
(Spake default 7 days). Content churn and attestation are counted
separately because attestation dominates every steady state.

| Scenario | Active domains | URLs | Churn per day | Attest interval | Deltas per day | Deltas per hour |
|---|---|---|---|---|---|---|
| Pilot | 1 000 | 200 000 | 2 % | 7 d | 32 600 | 1 360 |
| Regional | 100 000 | 20 M | 1 % | 7 d | 3.06 M | 127 000 |
| Web-wide | 10 M | 2 000 M | 0.5 % | 7 d | 296 M | 12.3 M |

Bursts: a Publisher backfill publishes its whole inventory at once, bounded
per Block by `domain_block_entries_max` (10 000 entries per domain per
Block) and by the Feed window (1 000 IDs per Feed page); a domain with a
200 000-URL backfill needs 20 Blocks at the cap. Skew: assume 1 % of
domains produce 50 % of Deltas. Retries: a lost Ping defers a pull to the
next baseline poll (`baseline_poll_seconds`, 24 h); a rejected Feed retries
on the next Ping.

Per Delta the aggregator makes about 2 site requests (Feed, Declaration
and Registry Update probe per pull, then one Delta and one Payload fetch
per Delta; 2.03–2.06 measured with the baseline below); Audit Records
add 0.02–0.50 Records per Delta per admitted Auditor (`sampling_floor`
to `sampling_ceiling`), each with its own fetch of the audited page.

### Targets

- Freshness: a pulled Delta seals in the next Block, so the median
  Ping-to-seal latency is half a cadence (30 min) and the maximum one
  cadence plus the pull backlog; the inclusion ceiling is the protocol's
  `max_inclusion_blocks` (4 Blocks) counted from admission.
- Backlog recovery: after an outage of *h* hours the aggregator drains the
  accumulated pulls within *h* hours at no more than half its sustained
  ingest capacity, so steady traffic keeps flowing during the drain.
- Storage: the Log and Checkpoints are retained forever; Payloads for
  `payload_window_days` (180 d) and Mirror artifacts for
  `mirror_retention_days` (90 d) at least; the Snapshot of the current
  seal plus the previous day's; the SQLite store grows with URLs, sealed
  Deltas and derived state.
- Bandwidth: outbound is dominated by Consumer cold starts (one Snapshot
  each) and catch-up (Blocks plus Payloads); inbound by Delta and Payload
  fetches. Budget both from the scenario's Deltas per day and Consumer count.
- Operating cost: the pilot runs on one 4-vCPU, 8–16 GB, 160 GB SSD virtual
  server plus backups; a scenario that needs more than one dedicated
  4-vCPU server per 100 000 active domains fails its cost budget until the
  measured unit costs improve.

### Per-Log capacity bounds

The Block cap (`block_decompressed_cap_bytes`, 256 MiB) at the default
hourly cadence bounds a Log at 74.6 KB/s of canonical entry bytes. At the
measured 483 bytes per Delta entry that is at most 556 000 Deltas per
hour, 13 M per day, including Registry Updates and Audit Records. Under
weekly attestation alone a Log therefore carries at most about 93 M URLs;
the web-wide scenario needs 5.5 GiB of entries per hour, 22 times the cap,
so it is a many-Log deployment by construction (WIST-3 §7 multi-Log
Consumers), not a larger single Log. The Regional scenario fits one Log at
23 % of the cap; the Pilot at 0.25 %. Blocks compress 3.9× under
Zstandard; each Delta adds about 1.7 KB of Payload, 2.0 KB of Snapshot,
0.9 KB of SQLite and 3.5 KB of Consumer store.

Governance and audit obligations scale the same bound: every accepted
Audit Record, coverage attestation, Observer checkpoint and canary act is
an entry, so a roster of *A* Auditors adds up to 0.5 × *A* Records per
Delta at the sampling ceiling.

Full-history retention grows the Log by the sealed entry bytes: 7 GiB per
year uncompressed at Pilot, 620 GiB at Regional (about a quarter of that
after Zstandard).

### Cost model

Let *B* be the number of sealed Blocks, *U* the tracked URLs, *R* the
records in the Snapshot and *D* the Deltas of one pull. The current
implementation costs:

- Pull: *c_hist* × *E* to derive the parameter schedule, plus *c_hist* ×
  *E* again for every Delta whose predecessor is already sealed, because
  predecessor resolution replays the Block history from disk; plus
  *c_delta* × *D*. *E* is the number of sealed entries (about 20 µs each
  when replayed), so a pull of *D* updates costs *D* × *E* × 20 µs: 1.5 s
  per Delta at 10 000 sealed entries, and a 100-Delta pull against one
  day of Regional history (3 M entries) would take hours. Pings queue
  behind the pull holding the store, so every Publisher sees that latency.
  Four pulls run concurrently under one store mutex.
- Seal: *c_snap* × *R* for the full Snapshot rebuild and derived-state
  replay, about 0.24 ms per record (2.4 s at 10 000 records, 4 min at 1 M,
  the whole cadence at 15 M), plus *c_hist* × *E* for the history replay
  and *c_entry* per new entry (0.5 ms at 10 000 entries per Block).
- Consumer cold start: Snapshot download proportional to *R* (2 KB per
  record, 2.2 s per 10 000 records with tier 1); catch-up: one sequential
  Payload fetch per applied Delta, 40 ms each on loopback, so a Consumer
  keeps up with at most 90 000 applied Deltas per hour and cannot follow
  the Regional Log as implemented.

These three terms, not fetch throughput, bound the aggregator: at every
scenario the per-Delta history replay and the per-seal rebuild exceed the
fetch and verification cost of the same Deltas, and both grow with the
sealed history. The ingestion architecture must remove them before any
concurrency work: predecessor and schedule lookups read persisted indexed
state, derived state extends from the previous seal instead of replaying
from genesis, and Snapshot production is incremental or decoupled from
the cadence. Consumer catch-up needs bounded concurrent Payload fetches.

### Per-worker resources

A fetch and verification worker holds, per in-flight pull, one Feed page
(1 000 IDs), one Delta and one Payload at a time, bounded by the Payload
caps (under 40 KiB JCS) and the fetch byte limits; its memory is
concurrency × (Feed page + Payload + verification state) plus the Key Set
cache (`keyset_cache_ttl_seconds`), independent of *U*, *B* and *R*.
Authoritative storage (SQLite, Log files, Payloads, Snapshots) lives with
the primary and grows with the scenario. Workers keep only task inputs,
results and bounded caches.

### Unsupported paths

Recorded so that measured support is never assumed:

- Snapshot sharding is specified (WIST-3 §7 per-shard digests) and Clave
  writes `snapshot_shard_count` shards, but Graven reads only unsharded
  Snapshots.
- Multi-Log consumption is specified and Graven follows several Logs with
  Delta-ID deduplication, but no Log discovery or selection exists and
  Clave runs exactly one Log per process.
- Sitemap index documents are not read by Spake; discovery is one sitemap
  or one RSS feed per domain.
- Audit Records, Observer acts and canary acts are replayed by Clave but
  not produced by any live Auditor; their ingestion and fetch costs are
  modeled, not measured.
- Mirrors, standby replication and worker distribution are not implemented.

### Baseline procedure

`graven/e2e`'s `baseline` binary publishes *N* loopback sites of *M* pages
through the real Spake, Clave and Graven executables, ingests them through
one aggregator behind a request-counting proxy, seals, cold-starts a
consumer, changes a share of the pages, seals again, seals *K* further
empty Blocks, verifies the history and catches the consumer up. Each seal
stage runs sealing, incremental Snapshot production and, on request, a full
rebuild at the same head as separate processes, reporting each one's wall
seconds, peak resident set size, bytes written, bytes reused and Payload
reads, and can end with a withdrawal seal. It reports
wall seconds, bytes and request counts per stage with the revision of every
repository it ran, so a measurement is reproducible at a named revision on
any machine (`WIST_BUILD_PROFILE=release`). Model comparisons use:

- publish seconds per Delta, well-known bytes per Delta;
- seconds until all domains are pulled, site requests per Delta;
- seal seconds at Block 0 and per empty Block as history grows (*c_hist* +
  *c_ext* from the slope); Snapshot production seconds, bytes rewritten and
  Payload reads per build, incremental against a full rebuild (*c_snap* × *R*
  is the full rebuild; an unchanged build's cost is the state file and
  manifest);
- Block, Payload, Snapshot, SQLite and store bytes per Delta;
- consumer cold-start and catch-up seconds and store bytes.

Machine-dependent timings are recorded with the deployment planning
notes for the environment they were measured on, not in this record;
the model above uses the byte and request ratios, which do not depend on
the machine.

## Consequences

The ingestion architecture decision must start from the three findings
above before any fetch-throughput work, because already at the Pilot
scenario the aggregator's own replay and rebuild costs exceed its fetch
costs. The
web-wide scenario is a multi-Log design question for the architecture
decision, not a capacity target for one Clave. Full and partial Consumers
have a reproducible baseline; the later qualification repeats it with live
audit traffic and distributed workers and carries every cost this record
leaves unmeasured.

## Addendum (2026-09-17): audit terms void after ADR-0007

[ADR-0007](0007-signed-publications-and-consumer-trust.md) retired the
Auditor role, and the audit machinery has since left core, Clave and
Graven. The following terms of this record are void and are not
replaced:

- Workload: the Audit Record term (0.02–0.50 Records per Delta per
  admitted Auditor, each with its own page fetch). Per Delta the
  aggregator makes about 2 site requests: Feed and Declaration per pull,
  then one Delta and one Payload fetch per Delta; the Registry Update
  probe is gone.
- Per-Log capacity bounds: the roster term (0.5 × *A* Records per Delta).
  A Log carries Deltas, Declarations, Labels and the four governance acts;
  the 556 000-Deltas-per-hour bound at 483 bytes per entry stands.
- Cost model: the seal term's derived-state replay. A seal rebuilds the
  Snapshot (*c_snap* × *R*) and replays the Block history (*c_hist* × *E*);
  no derived governance state is refreshed.
- Unsupported paths: the line on replayed Audit Records, Observer acts
  and canary acts.
- Consequences: the later qualification repeats the baseline with
  distributed workers only; there is no live audit traffic to carry.

The baseline procedure is unchanged; it pings each domain until the
aggregator admits the Ping, honoring the admission gate's 503, and
reports the refusals per stage. The 2026-09-17 re-measurement at the same
scales is recorded with the deployment planning notes beside the
2026-09-16 record.

## Addendum (2026-09-18): Block renamed Epoch

The specification's draft ADR-0048 renamed the protocol term Block to
Epoch: since the single-tree Log a Block is no object, only the interval
of the Log between two consecutive Checkpoints, and key-transparency logs
call that interval an epoch. No behavior changed. Read every "Block" in this
record as "Epoch", and its Parameter Registry identifiers under their
new names:

- `domain_block_entries_max` → `domain_epoch_entries_max`
- `max_inclusion_blocks` → `max_inclusion_epochs`
- `block_decompressed_cap_bytes` → `epoch_cap_bytes`

Values, units and the capacity model are unchanged.
