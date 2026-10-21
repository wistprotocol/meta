# Sealer memory and write-lock remedy, measured 2026-10-21

The code remedy that [ADR-0009](../decisions/0009-clave-storage-and-expansion.md) named as the next capacity work, measured on the production primary's cloud plan against the [2026-10-19 qualification](capacity-qualification-2026-10-19.md) at the same Pilot shape. Harness: the `sustained` binary of Graven's `e2e` crate; scenario flags as on 2026-10-19 (`skew` at 1 000 domains of 200 pages, 2 Labelers, 3 rounds of 20 000 changed pages and 2 200 Labels, sealed while the pulls run).

## Setup

- Hetzner fsn1, shape primary: aggregator on `ccx23` (4 dedicated vCPU, 16 GB) plus a load generator on `cpx32`; rented 20:30Z, destroyed 23:13Z.
- Aggregator under test built on the host as the static `x86_64-unknown-linux-musl` release with mimalloc (the release workflow's recipe) at clave `98f613a` (a first run at `4c3c710` is noted where it explains a figure); core `27b88b9`, graven `5ee937f`, spake `539e2d2`, spec `8643e7a`. Harness placement as on 2026-10-19: everything on the aggregator's VM.

## What changed in the aggregator

1. A seal loads only the state its Epoch can touch (the publishers with waiting work, the domains with an open recovery window or a pending Declaration, the subjects of pending withdrawal acts, the sealed Items those acts name, the sealed Label IDs of the scope's labelers and the subjects of the Labels its waiting disputes name) instead of resuming a replay of the whole Log; a replay-agreement test compares the derived state it persists with a replay from genesis.
2. The records of the planned publishers are held once: the parsed Item is shared across the states before and after the Epoch, the planned state is moved into the write, and the planner's last judgment is kept and persisted at the seal instead of a second judgment over a second copy of every record inside the sealing transaction.
3. Under the write lock, the sealed Items are indexed once instead of searched per Item, and every per-publisher read of the merge walks the publisher's key range.
4. `clave seal` and `clave serve` report per-phase timing (planning, lock wait, merge, entries, replay, state, commit, publication, retention, witness) with each phase's statement-execution time, resident size and high-water mark, and the longest wait in `BEGIN IMMEDIATE`; the harness reads them.
5. The parameter schedule is cached in the store and replayed only above the cache; the rejections, pending entries, records by Item, parameter changes and Payload duties gain indexes; the seal's row writes use prepared statements; distribution reads only unserved duties and leaves the full tiles and bundles of a published Epoch alone.
6. Retention scans in a read-only snapshot and re-checks each candidate under a short write transaction.

## Pilot size on ccx23

Every check passed; 0 rejected pulls; 0 re-plans. 2026-10-19 figures in parentheses.

- Genesis of 208 000 Items: seal 477 s (772 s) holding the lock 112 s (387 s), sealer peak 4.50 GB (5.81 GB); Snapshot 21.9 s (22.5 s). Of the seal, 92 s is planning, 16 s the state writes and 359 s the Payload publication after the commit, one fsynced file per sealed page Item.
- Rounds: pulls 169 / 167 / 170 s (157 / 171 / 163 s); seals 12.0 / 65.2 / 70.3 s (21 / 101 / 99 s) holding the lock 0.87 / 4.29 / 4.59 s (2.2 / 16.9 / 15.4 s) while merging 0 / 183 / 230 publishers at 4.4 ms each (36 ms); sealer peak 0.48 / 2.28 / 2.49 GB (1.5 / 4.5 / 4.7 GB); the pulls' longest wait for the write lock 0.08–0.23 s; Snapshot 27–29 s (27–28 s); aggregator peak 118–143 MB (127–141 MB); status p99 9.6–17.3 ms (7.0–7.8 ms; bound 500 ms).
- Composition of the round-3 lock hold (4.59 s): merge 1.02 s, entries 0.37 s, replay 0.02 s, state 2.86 s, commit 0.31 s; 48 % of it is statement execution and commit, nearly all in the state writes. Of the seal's 70 s, planning (34 s) and the Payload publication (25 s) run outside the lock.
- The first run, before the planner's judgment was reused at the seal, held the lock 1.5 / 9.9 / 10.9 s with a 5–6 s `replay` phase and peaked at 0.41 / 2.42 / 2.65 GB; the second run removes that phase.

## Memory scenario on ccx23

Every check passed, including the new check that the empty seal's peak stays flat in the record count. Empty seal 28.6 MB at 1 000 records and 32.7 MB at 10 000 (0.45 kB per record), unchanged over 20 Epochs; 28.7 MB with 10 000 sealed Labels; probe pull increase 0.2–1.6 MB.

The 2026-10-19 figures for the empty seal (34.8 and 110.4 MB, "8.4 kB per sealed record"; 79.5 MB with 10 000 Labels, "4.5 kB per Label") carried a harness defect: a measured process's peak was read from the rusage of `wait4`, and a child spawned with a shared address space until exec carries the parent's resident size into that figure, so the harness's own size at spawn (growing with the cell) was the floor of every short-process peak. About 6 kB per sealed record of the earlier slope was the aggregator's whole-Log replay, now removed; the rest was the harness. Graven `5ee937f` reads the child's VmHWM instead. The sealer peaks of the Pilot rounds, hundreds of MB to GB, were not affected.

## Declared

1. A seal's memory no longer grows with the Log: the Log term of ADR-0009's model is gone. The remaining term is about 12.5 kB per record of a publisher with waiting work (14 kB above the Log term before): the planned state's lists, waiting URLs and Collections, and the copy of each judged record's Item in the judgment Replay. The target of 2 GB for a Pilot round is not met (2.28–2.49 GB); the Regional round's term is about 14 GB, within the dedicated primary's 36 GB memory trigger.
2. The longest write hold at Pilot shape is 4.6 s (target 5 s; 17 s before), half of it the engine's statement execution and commit. ADR-0009's storage triggers (hold ≥ 30 s with the engine's share above half, pulls ≥ 900 s, status p99 ≥ 250 ms) are far from reached; no storage candidate reopens.
3. The seal's time at Pilot shape is planning and the Payload publication, both outside the lock; the per-Item work on one core (about 3.3 ms per Item) is the next bound, and the Payload publication is three quarters of a genesis seal.
4. Every earlier declared figure stands except the memory scenario's short-process peaks, superseded by the figures above.
