# Capacity qualification on rented shapes, 2026-10-19

The first capacity figures measured on the production primary's plan rather than on a development machine, and the measurement the expansion decision ([ADR-0009](../decisions/0009-clave-storage-and-expansion.md)) rests on. Scenarios and targets: [ADR-0005](../decisions/0005-capacity-model-and-baseline.md); harness: the `sustained` binary of Graven's `e2e` crate (its README gives every scenario's shape and checks).

## Setup

- Hetzner fsn1. Primary shape: aggregator on `ccx23` (4 dedicated vCPU, 16 GB, 160 GB disk, EUR 0.1626 per hour) plus a load generator on `cpx32` (4 shared vCPU, 8 GB, EUR 0.0673 per hour), both under `sslip.io` names with Let's Encrypt certificates. Comparator shape: aggregator on `ccx33` (8 dedicated vCPU, 32 GB, 240 GB disk, EUR 0.2612 per hour), rented after the primary shape was destroyed, installed and built the same way.
- Aggregator installed by the production automation (`deploy`, Clave v0.2.2: the static `x86_64-unknown-linux-musl` release with mimalloc). Revisions: clave `fa94258` (the v0.2.2 tag), core `682b98a`, graven `fbd3385`, spake `539e2d2`, spec `8643e7a`, the revisions of the last development-machine measurement, so every difference between the two machines is the hardware.
- Harness placement: the harness has no remote mode (it spawns its own `clave serve --no-seal` on loopback, seals with `clave seal`, serves the publisher sites through its own proxy and reads the aggregator's `/proc`), so it, its 1 002 loopback sites and their Spake republishing ran on the aggregator's VM, with the installed release binary as the aggregator under test. The VM's cores therefore also served the sites and ran Spake; pull and publish figures carry that load, and nothing here measures the aggregator fetching over a real network. Only the Consumer measurement used the generator and the installed service.
- Comparator runs were sequential in the same window, not side by side.

## Pilot size on ccx23

`skew` at 1 000 domains × 200 pages, 2 Labelers, 3 rounds of 20 000 changed pages with `--labels-per-round 1100` and `--seal-during-pulls`. Every check passed; 0 rejected pulls; 0 re-plans. Development-machine figures at the same revisions in parentheses.

- Whole run 2 982 s (2 905 s); publish 162 s (123 s).
- Genesis: 208 000 Items over 1 002 domains; pings 1 103 s with 2 789 admission refusals, all pulled 74 s after the last ping (83 s); aggregator peak 98 MB. Seal 772 s with 387 s under the write lock (668 s, 294 s); sealer peak 5.8 GB (5.7 GB); Epoch 207 MB of Entries, Payloads 347 MB, store 681 MB, data directory 2.45 GB; Snapshot 22.5 s (16.0 s), 388 MB of shards.
- Rounds 1–3: pulls 157 / 171 / 163 s (170 / 194 / 167 s); seals 21 / 101 / 99 s (34 / 94 / 99 s) holding the lock 2.2 / 16.9 / 15.4 s (2.6 / 9.2 / 8.6 s), merging 0 / 463 / 405 publishers (0 / 224 / 150); sealer peak 1.5 / 4.5 / 4.7 GB (1.8 / 4.3 / 4.5 GB); 129 / 811 / 848 pulls committed during the seal; Snapshot 27.4–28.2 s (15.7–17.0 s); aggregator peak 141 / 130 / 127 MB; status p99 7.0 / 7.0 / 7.8 ms (3.2–3.7 ms), 5.4–6.0 ms over the seal and Snapshot alone; round publish 63–69 s.
- Reading: the dedicated core is within 15 % of the development machine on the seal's compute and on the pulls; its disk is slower (Snapshot 1.7×, genesis lock hold 1.3×, merge lock hold about 36 ms per merged publisher against 30–40 ms); status p99 doubles but stays under 8 ms against the 500 ms bound.

## Default-size scenarios on ccx23

Every check of every scenario passed.

- `backfill` (100 domains × 10 pages plus one 20 000-page domain, 3 rounds): 320 s in all; the backfill's 20 100 Items pulled in 169 s, sealed in 23.7 s holding the lock 7.7 s, sealer 370 MB, aggregator 202 MB, status p99 10.4 ms; the backfill drained over exactly 3 Epochs (10 000, 10 000 and 1 Entries) under `domain_epoch_entries_max`.
- `hosts` (100 domains × 20 pages; 5 slow at 2 s, 5 failing, 2 hanging 20 s): every healthy host pulled within 15.2 s of the last ping (bound 60 s); the 7 failing and hanging hosts show the WIST2-E01 rejection and were requested again without a new Ping; status p99 6.1 ms; aggregator peak 44 MB.
- `saturation` (1 000 ingested domains × 5 pages plus 1 000 fresh sites, 8 ping threads): the first-contact flood refused 881 hosts (992 refusals), every one admitted on a later Ping and pulled within 2.3 s of the flood; the update flood pulled within 77.9 s; under a local `epoch_cap_bytes` of 1 285 510 octets the Entries spread over 7 Epochs, each Item and Catalog sealed exactly once (10 000 Items, 0 lost, 0 duplicated); status p99 5.1 ms during the seals; aggregator peak 84 MB.
- `drainage` (100 domains × 20 pages; 2 steady, 3 outage and 3 recovery rounds): mean steady round 9.2 s; no Ping admitted during the outage; after the restart every domain pulled within 13.1 s (bound 27.5 s); each recovery round's changed Items sealed in the next Epoch; status p99 35.0 ms at most (the restart round).
- `memory` (cells of 1 000 and 10 000 records × 1 and 20 Epochs, plus 10 000 Labels at 1 000 records): the probe pull's resident increase 600 kB at 1 000 records against 852 kB at 10 000 (1 Epoch) and 60 kB against 1 128 kB (20 Epochs, within the 8 MB allowance); the empty seal's peak 34.8 MB at 1 000 records and 110.4 MB at 10 000, unchanged from 1 to 20 Epochs; with 10 000 sealed Labels the probe pull's increase 1 464 kB and the empty seal 79.5 MB, 4.5 kB per sealed Label (bound 10 kB).

## Consumer over a real network

Against the installed service on the primary (cadence 3 600 s), Graven `fbd3385`. A 2 000-page Spake site on the generator was pulled on its Ping and sealed as Epoch 0 (tree size 2 003); a change round of 1 000 pages sealed as Epoch 1 (tree size 3 004); both heads verified from the operator machine. Two vantages: the generator in the same datacenter (34 ms p50 per fresh TLS connection) and a machine on another continent (1.4 s p50 per fresh connection, about 220 ms per request on a kept-alive connection).

- Cold start from the Snapshot (2 003 Entries): 9.3–11.4 s from either vantage whatever `--payload-fetches`; the cold start applies the Snapshot's records and fetches no Payload, so about 5 ms per record is the Consumer store's writes.
- Catch-up of 1 000 changed Items, by `--payload-fetches` 1 / 4 / 8 / 16 / 32: in the datacenter 3.1 / 1.1 / 1.8 / 1.6 s; from the far vantage 257.2 / 55.0 / 29.7 / 18.5 / 13.7 s. A Payload fetch is one round trip; concurrency divides it almost linearly to 16 and less beyond. Graven's `--payload-fetches` default is 16 from this measurement: a far Consumer then catches up about 195 000 applied Items per hour, 260 000 at 32, so the Regional scenario's Consumer cap sits in the store's writes, not the fetch.

## Comparator: ccx33 at the same Pilot shape

Every check passed; 0 rejected pulls; 0 re-plans. ccx23 figures in parentheses.

- Whole run 3 232 s (2 982 s); genesis seal 805 s with 368 s under the lock (772 s, 387 s), sealer peak 5.81 GB (5.81 GB); Snapshot 23.1 s (22.5 s).
- Rounds: pulls 194 / 218 / 221 s (157 / 171 / 163 s); seals 25 / 104 / 105 s (21 / 101 / 99 s) holding the lock 2.2 / 11.9 / 11.9 s merging 0 / 260 / 286 publishers (about 44 ms per merged publisher against 36 ms); sealer peak 1.51 / 4.39 / 4.70 GB; Snapshot 24.9–26.3 s; aggregator peak 129 / 128 / 172 MB; status p99 7.7–9.3 ms; round publish 39–40 s (62–69 s), the harness's Spake runs using the eight cores.
- Reading: twice the cores and memory seal nothing faster. The seal is one process on one core and a pull is one thread per domain, so the extra cores served only the harness's own publishing; per-core speed and disk are equal or slightly slower (pulls within a shape's run-to-run spread of 151–201 s). A larger plan buys memory for the sealer's term, not throughput, at 1.6× the price.

## Declared

1. Tested operating envelope, on `ccx23` with the v0.2.2 musl release: 1 000 publisher domains of 200 pages plus 2 Labelers, a 208 000-record genesis, and hourly rounds of 20 000 changed Items and 2 200 Labels across 1 002 domains, sealed while pulls run; per round the pulls take 157–171 s and the seal plus Snapshot 50–130 s of the 3 600 s cadence; the sealer peaks at 4.7 GB (5.8 GB at genesis), the aggregator at 141 MB, status p99 under 8 ms; every bounded-queue, progress, drainage, responsiveness and recovery check of the six scenarios passes. The Regional scenario of ADR-0005 is not claimed: its sealer memory term exceeds the plan, and nothing above tests it.
2. Cost per million sealed publisher Deltas on `ccx23` (EUR 103.69 a month for the primary with automatic backups and an IPv4 address): EUR 105 at ADR-0005's Pilot workload (32 600 Deltas a day, the server mostly idle); EUR 7.2 at the tested round rate (20 000 Items an hour, 14.4 M a month) with the mix above (11 % Labels, 1.8 KB per Item). The `ccx33` at the same rate costs 1.6× for no gain.
3. The bottleneck of one Log is the sealer, its memory while planning and its single-core seal, not fetch: the pulls use under 5 % of the cadence.
4. Development-machine figures compare only with each other; declared figures come from rented shapes. The musl release's seal and lock hold run about 20 % longer than a glibc build, a retained cost; a glibc release is the mitigation if a seal approaches the cadence.
