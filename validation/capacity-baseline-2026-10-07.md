# Capacity baseline under declared publication, 2026-10-07

Measured per the baseline procedure of [ADR-0005](../decisions/0005-capacity-model-and-baseline.md) after the adoption of declared publication ([ADR-0008](../decisions/0008-declared-publication.md)): a Publisher serves Catalogs, Items, tree files and change lists; an Aggregator obtains a Collection's list by its chain of change lists or by walking the tree. Figures in parentheses are the 2026-09-23 measurement of the same workload under the previous publication path, on the same machine; both sets are valid only relative to each other. Everything ran on loopback with release builds, so fetch costs are lower bounds. Revisions: spake 6f6c8e2, clave 52b0e3a, core f45d885, spec f4acfef, graven 6ec50cb with the baseline binary of graven 85186ff; release profile.

`clave snapshot --rebuild` answered "snapshot current" and rebuilt nothing at the aggregator revision measured, so no full-rebuild figure exists here.

## Run A — 100 domains × 100 pages, update round by change list

- Publish: 3.04 s for 10 000 Items (0.30 ms per Item), served files 22.05 MB.
- Ingest: all pulled 4.53 s after the last ping (13.5 s); 1.20 site requests and 1 945 B per Item (2.03 requests per Delta): 10 000 Payloads 16.60 MB, 1 696 tree files 2.77 MB, 100 Catalogs, 100 Declarations; aggregator peak 21 MB.
- Log octets per Entry: Item 913 B (484 B per Delta entry; the Item carries its Inclusion Proof against the Catalog), Catalog 507 B, Declaration 442 B, withdrawal 436 B.
- Seal 1: 6.80 s, 253 MB (3.24 s, 218 MB); Snapshot 1.91 s, 171 MB, 7.92 MB written; entry bundles 9.24 MB (5.26), tiles 9.57 MB (5.59), Payloads 16.6 MB, SQLite 32.5 MB (12.8).
- Consumer cold start: 4.38 s, 121 MB peak, store 42.1 MB (2.47 s, 36.1 MB).
- Update round, 1 000 changed pages: republish and ping 97.0 s (0.97 s per 100-page site, about 10 ms per listed page); all pulled 0.97 s after the last ping (353.6 s); 1.40 requests and 2 022 B per changed page: 100 change lists 276 KB, 1 000 Payloads, 100 Catalogs, 100 Declarations, no tree file; aggregator peak 673 MB.
- Seal 2: 2.17 s, 249 MB (0.49 s, 68 MB). Empty seals: 1.18 s then 1.25 → 1.78 s over ten Epochs, rising 42 % (0.18 s, rising 5 %); Snapshot of an unchanged head 1.36 s, 4.28 MB written (0.25 s, 1.32 MB).
- Consumer catch-up over 12 Epochs with 1 000 applied Items: 6.30 s, 88 MB (42.7 s). Withdrawal seal 1.84 s; SQLite 41.6 MB after.

## Run B — as A, update round by walk

Every served change list deleted before the pings; identical to A through seal 1.

| Update round | By change list (A) | By walk (B) |
|---|---|---|
| Requests per changed page | 1.40 | 2.26 |
| Response octets per changed page | 2 022 B | 3 284 B |
| Tree files fetched | 0 | 861 (1.54 MB) |
| Pulled after the last ping | 0.97 s | 0.98 s |
| Aggregator peak | 673 MB | 565 MB |
| WIST2-E08 reports | 0 | 100 |

At 100 pages per Collection and 10 % changed, the change list saves 38 % of the requests and octets of the update pull.

## Run C — 100 domains × 10 pages beside 2 domains × 5 000 pages

- Ingest: 1.50 requests per Item overall; the large domains 1.51 requests per Item, pulled 22.2 s after the last ping, each pull suspended once at the ingest bound and resumed; the small domains pulled in 1.13 s; aggregator peak 116 MB.
- Seal 1 (11 000 Items): 121.2 s, 300 MB, against 6.8 s for 10 000 Items in 100 Collections: the seal's cost grows with Collection size, not Item count. Item Entries average 1 249 B (913 B in A).
- Update round, 1 100 changed pages: the large domains 1.008 requests per changed page by chain (change lists 261 KB), pulled 4.30 s after the last ping; the small domains 5.0 requests per changed page (Catalog, Declaration, change list, Label Feed probe and one Payload per site); aggregator peak 434 MB. Seal 2: 13.8 s, 274 MB.

## Run D — a tree at the parameter bounds

One Collection of 3 000 Items whose tree the harness built itself: 13 228 tree files (2.48 MB) over levels 1 to 16; 1 024 Items each alone in a bucket at level 16 (`tree_depth_max`) under a prefix trie of single-child files; 1 976 Items filling 8 buckets at level 2 of 65 377–65 407 octets (`tree_file_cap_bytes` 65 536). Full buckets at the depth bound were not built.

- Walk: 16 239 requests, 7.44 MB (13 227 tree files, 2 999 Payloads), 4 pull runs at the 4 096-object stop; all pulled 12.9 s after the ping; aggregator peak 61 MB.
- Seal of the 3 000 Items: 22.3 s, 96 MB; Items average 1 243 B. Consumer catch-up 2.35 s, 55 MB.

## Change against 2026-09-23

- Update pull 353.6 s → 0.97 s after the last ping; consumer catch-up 42.7 s → 6.30 s: the Catalog lists the Items and the change list names the changed ones, so neither party fetches one object per record.
- Log octets per record 484 B → 913 B, plus 507 B per Catalog per changed Collection per Epoch; entry bundles for 10 000 records 5.26 MB → 9.24 MB.
- Seal at 10 000 Items 3.24 s → 6.80 s; per empty Epoch 0.18 s → 1.2–1.8 s, rising 42 % over ten Epochs; unchanged Snapshot 0.25 s → 1.36 s; SQLite 12.8 MB → 32.5 MB.

Findings carried to capacity qualification: seal time growing with Collection size; the aggregator's peak memory in the update round (434–673 MB against 21–116 MB at ingest); the empty-seal rise; the Publisher's run cost per listed page; the Snapshot rebuild that rebuilds nothing.

Unmeasured: network latency, Labels, Mirrors, workers, sustained runs, multi-Log consumption, a consumer reading a sharded Snapshot, full buckets at the depth bound, a Collection at `catalog_items_max`, a full Snapshot rebuild.
