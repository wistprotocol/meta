# Three-role functional integration

One reproducible test drives Spake (Publisher and Labeler), Clave
(Aggregator) and Graven (Consumer) as separate processes over HTTP. Its 18
test functions each start their own sites, Logs and Consumer stores, and
assert the Log's served files and Snapshot state, the Aggregator's status
endpoint and the Consumer's state, as its MCP tools and index report it,
after every sealed Epoch. It establishes that the three roles interoperate
functionally at the revisions below. It is not a load test, and it does not
replace the acceptance evidence the specification's
[CONFORMANCE.md](../../spec/CONFORMANCE.md) requires before stable
publication.

## Revisions

| Repository | Revision |
|---|---|
| `spec/` | `f4acfefbe7d3cdb8e82ed34053377159d8782890` |
| `core/` | `a110daf90ec193fdf34b871e5cc29256a0b649b7` |
| `spake/` | `47d9b712212ca2d1ff2710686c9e9e8d599e6cf2` |
| `clave/` | `372c5629ea866a9e143259ba54b9c7acd9aefb4d` |
| `graven/` | `deab1f4c6eacd2575c27bb40a58f4444e2d55b1e` |

The run passed on 2026-10-07. The release tag v0.2.1 of `graven/` follows that revision by the release configuration and the version alone.

## Reproducing it

Check out the five repositories side by side at those revisions; Spake,
Clave and Graven resolve `wist-core` from `../core`. Then:

```
cd graven
cargo test -p e2e --test e2e
```

The test reads these settings:

- `SPAKE_BIN` and `CLAVE_BIN` name the binaries. When one is unset, the
  test runs `cargo build -p spake` in `../spake` (or `-p clave` in
  `../clave`) and uses the built binary. Graven is always built from the
  `graven` workspace.
- `WIST_BUILD_PROFILE=release` selects release builds; any other value
  selects debug builds.
- `WIST_SPEC_DIR` names the specification checkout, `../spec` by default.
- The artifact check `e2e/validate_artifacts.py` runs under
  `$WIST_SPEC_DIR/tools/.venv/bin/python3` when it exists and `python3`
  otherwise, and needs `jsonschema`, `rfc8785` and `cryptography`.
- The independent Log client `e2e/tlog-client` runs with `go run`, built on
  Go's `golang.org/x/mod/sumdb/note` and `tlog`.
- Without the Python packages or the Go toolchain, the check prints `SKIP`
  and the test continues. With `CI` set, it fails instead.

Each test serves its sites through a loopback HTTP proxy. Each Log starts
with `clave init --suffix-list` over a fixture Public Suffix List and
`clave serve --no-seal --allow-http`. Epochs seal with `clave seal --at` at
successive whole hours of the hourly cadence grid, starting at the next
whole hour, so Log time does not depend on the wall clock; Catalogs carry
wall-clock `generated_at` values. `e2e-domain-capacity` and `e2e-recovery`
move the grid about 7 days ahead where their tables say so. Each `spake ping` waits for a pull
recorded after it whose status lists no rejection beyond the ones the test
expects. The Consumer cold-starts with `graven follow` and continues with
`graven sync`, and every check starts a new `graven serve` MCP process.

Each test writes a run record, `target/tmp/<record>.json`, naming each
repository's `HEAD` and whether its tree carried changes, and each scenario
with the Epoch that sealed it. Epoch numbers start at 0 in every Log; where
a test runs two Logs, the numbers are the first Log's.

## Scenarios

Rows sharing an Epoch assert different aspects of one sealed Epoch.

### `e2e-publication`

`a_publication_reaches_the_consumer_through_two_logs_with_updates_removals_and_withdrawals`:
Publisher `localhost`, Collection `docs`, two Logs. `payload_withdrawal`
and `aggregator_restart` seal the first Log alone.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | `spake init` serves a Declaration; both Logs seal it |
| 1 | `publication` | `spake collection add`, `spake emit --complete` and `spake publish --stream` publish three marked pages. The Consumer is installed from each Log's Snapshot at Epoch 1. `get_record` reports the Item ID, Collection, Publisher, `generated_at`, `attested_at` and title, with one provenance entry per Log at height 1; `search` agrees with it; `get_extract` returns the page text |
| 2 | `update` | A changed page is a new Item; `get_record`, `search` and `get_extract` return its new title, `generated_at` and text |
| 3 | `url_removal` | A deleted page has no record and no search hit; the domain's other records stay |
| 4 | `payload_withdrawal` | `clave withdraw` on the first Log; the record keeps the second Log's provenance alone |
| 5 | `publisher_withdrawal` | `spake withdraw --unpublish` deletes the Item's Payload file; with the page removed, its record is gone |
| 6 | `aggregator_restart` | An Item the first Log admitted before Clave stopped seals after Clave restarts on the same data directory; the Log Anchor is byte-identical; `clave verify-history` passes; no Consumer head moves to a lower Epoch or tree size, and a head at an unchanged Epoch keeps its root |
| 7 | `publisher_restart` | Spake, run with an environment holding only `PATH`, publishes from its persisted state directory; the new Item is the record and is searchable |

### `e2e-labels`

`labels_disputes_and_link_ranking_reach_the_consumer`: Publisher
`localhost`, Labeler `labeler.localhost`, two Logs, and six linked sites
published to the first Log alone.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `publication` | Every site publishes |
| 1 | `label` | `spake label` signs `wist:spam` (value 900000) on a page and `wist:trust-seed` on `seed.localhost`; `spake define` publishes the `wist:spam` definition with treatment `warn` |
| 2 | `label_dispute` | The labeled Publisher's `spake dispute` seals. After `graven subscribe`, `get_labels` returns the Label with its Labeler, value, Label ID, `subscribed`, treatment, the dispute and provenance from both Logs; `list_labelers` counts two Labels; `list_profiles` lists at least four profiles. Under `default`, the page linked from the trust seed ranks above a keyword-stuffed page linked from three farm sites, with a positive trust signal and an explanation, and the spam-labeled page is dropped; `text-only` reverses the order and returns the spam-labeled page |
| 3 | `label_one_epoch_old` | A `wist:mismatch` Label live at one height does not count under `default` |
| 4 | `label_two_epochs_live` | The same Label, live at two consecutive heights, counts |
| 4 | `default_profile_persistence` | After `graven profile use text-only`, a new MCP process answers a query naming no profile with `text-only`; selecting `default` again drops the spam-labeled page |

### `e2e-publisher-keys`

`publisher_key_rotation_and_a_reversed_hijack_keep_the_records`: Publisher
`localhost`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `publication` | Two pages publish |
| 1 | `publisher_key_rotation` | `spake rotate --overlap-seconds 86400` serves a Declaration, signed by the outgoing key, listing that key with an `exp` and the incoming key without one; a Catalog signed afterwards is searchable |
| 2 | `hijacked_declaration` | A Declaration signed with a key the owner's Declaration does not list, at the next `seq` and chained to it, seals as a `pending_declaration` tuple whose activation height exceeds its sealing height; the `declaration` tuple stays the owner's |
| 3 | `hijack_reversal` | `spake rotate --restore` seals a Declaration at a higher `seq` that takes force; no `pending_declaration` tuple remains; the domain's records stay |

### `e2e-aggregator-keys`

`the_consumer_follows_aggregator_key_rotation_and_refuses_a_snapshot_under_a_retired_key`:
Publisher `localhost`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `publication` | `clave log-key list` shows the Anchor's genesis key admitted at height 0 |
| 1 | `log_key_addition` | `clave log-key add` queues a key with a distinct note key ID; the served head and archived Checkpoint 1 state one note text, signed by the genesis key and the admitted key; the Consumer reaches Epoch 1 |
| 3 | `log_key_removal_of_the_genesis_key` | `clave log-key remove` retires the genesis key at Epoch 3; Checkpoint 3 is signed by the remaining key alone |
| 5 | `cold_start_after_the_genesis_keys_removal` | After Clave restarts, Checkpoint 5 is signed by the remaining key alone, which stays unretired. A new Consumer cold-started from the Snapshot reaches Epoch 5 and holds the following Consumer's Epoch, tree size, root and records, and answers a query with the same URLs, Publishers and Item IDs |
| 5 | `snapshot_signed_by_the_removed_genesis_key` | The Snapshot directory served at Epoch 2, signed by the genesis key, is served again; `graven follow` fails with `WIST3-E04` naming the Snapshot index, and the store registers no Log (WIST-3 §8 step 8) |

### `e2e-declared-collection`

`a_declared_collection_seals_its_emissions_and_refuses_an_item_outside_its_scope`:
Publisher `localhost`, Collection `docs` over `/docs/`, one Log; the site's
served directory holds 100 generated pages.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `first_publication_of_a_generated_site` | `spake emit --complete` over the served directory and `spake publish --stream` publish the 100 pages. The Epoch seals one Catalog and 100 Items of `docs`; the status reports that Catalog as latest with nothing waiting; the Snapshot state's records are the Catalog's page Items and recompute its root; the Consumer holds 100 records in `docs` |
| 2 | `page_edited_and_page_removed` | An edited page and a removed page seal as two Items with the Catalog; the removal's Inclusion Proof verifies against the Catalog; the Log holds a removal state naming the Catalog; the Consumer holds 99 records |
| 3 | `text_emission_of_a_client_rendered_page` | An incremental stream line with text and links seals one Item; its Payload carries the text as `extract` and the links; `get_extract` returns the text and `get_links` the links at positions 0 and 1 |
| 4 | `item_outside_the_scope` | `spake emit` omits a page outside the Scope. A Catalog signed with the `docs` key and listing that page is accepted; the page's Item is refused with `WIST1-E03` naming the Collection and URL; no sealed Entry names the URL or its key |
| 4 | `removal_beside_a_refused_item` | A removal listed in the same Catalog seals with it |
| 5 | `catalog_without_the_refused_item` | Spake's next Catalog omits the refused page and seals with its one changed Item; the Log's records recompute its root; the refused URL has no record |

### `e2e-scope-narrowing`

`a_narrowed_scope_removes_records_at_its_epoch_and_a_reversed_pending_narrowing_removes_none`:
Publisher `localhost`, Collection `docs` over `/docs/`, `/drafts/` and
`/notes/` with one page each, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `publication` | The Log records all three pages |
| 2 | `pending_declaration_narrowing_the_scope` | A Declaration signed with a key the owner's does not list, dropping `/notes/`, seals as pending; the owner's stays in force; all three records stay |
| 3 | `pending_narrowing_before_activation` | The narrowing is still pending and removes nothing |
| 4 | `reversal_of_a_pending_narrowing` | `spake rotate --restore` takes force and discards the pending Declaration; all three records stay |
| 5 | `owner_narrows_the_scope` | `spake collection scope-remove` drops `/drafts/`. The Epoch seals the Declaration and no Item; the `/drafts/` record is gone from that height with no removal state; the Consumer neither returns nor finds it; the other records stay |

### `e2e-collection-keys`

`collection_keys_sign_only_their_own_collection_and_a_removed_delegate_key_signs_nothing`:
Publisher `localhost`, Collections `docs` over `/docs/` and `blog` over
`/blog/`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `first_collection_with_its_own_key` | `docs` is added with its own key and publishes |
| 2 | `second_collection_keyed_by_key_add` | `blog` is added with `--no-key`, then keyed with `spake collection key-add`. Each latest Catalog is signed by its own Collection's key, distinct from the other and from the owner keys; each record carries its Collection |
| 3 | `catalog_signed_by_another_collections_key` | A `blog` Catalog signed with the `docs` key is refused with `WIST1-E02`; nothing seals; the latest `blog` Catalog stays and the added URL has no record |
| 4 | `removal_naming_another_collections_url` | A `docs` Catalog listing a removal of a `blog` URL is accepted with that Item refused as `WIST1-E03`; the `blog` record stays with no removal state |
| 5 | `delegate_key_removed_and_owner_catalog` | `spake collection key-remove` removes the `blog` key; the Declaration and a `blog` Catalog Spake signs with an owner key, with no stream, seal together |
| 6 | `catalog_signed_by_a_removed_delegate_key` | A `blog` Catalog signed with the removed key is refused with `WIST1-E02`; nothing seals; the owner-signed Catalog stays latest |

### `e2e-catalog-order`

`a_stale_catalog_and_one_beyond_the_clock_are_refused_and_an_equal_one_served_again_is_idempotent`:
Publisher `localhost`, Collection `docs`, one Log. In Epochs 3 to 6 the
status keeps the Epoch-2 Catalog as latest and last accepted with nothing
waiting, nothing of the domain seals, and the Consumer's records do not
change.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `publication` | Two pages publish |
| 2 | `edited_catalog` | An edited page's Catalog becomes the latest |
| 3 | `stale_catalog_discarded` | The Epoch-1 Catalog served again is discarded with `WIST2-E05` |
| 4 | `equal_catalog_served_again_is_idempotent` | The latest Catalog served again draws no rejection |
| 5 | `other_catalog_of_an_equal_instant_discarded` | Another Catalog signed at the latest one's `generated_at` is discarded with `WIST2-E05` |
| 6 | `catalog_beyond_the_clock_refused` | A Catalog whose `generated_at` is 70 minutes ahead is refused with `WIST1-E06` |
| 7 | `next_catalog_after_the_refusals` | Spake's next Catalog seals with its one changed Item |

### `e2e-two-logs`

`one_catalog_sealed_by_two_logs_is_one_record_and_the_latest_catalog_decides_between_them`:
Publisher `localhost`, Collection `docs`, two Logs.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | Both Logs seal the Declaration |
| 1 | `one_catalog_sealed_by_two_logs` | Both Logs seal the same Catalog and Item IDs and hold equal record tuples; the Consumer holds equal records from each; `get_record` names both Logs; `search` returns the URL once |
| 2 | `later_catalog_sealed_by_one_log` | Pinged to the first Log alone, the edited page's Item from the later Catalog is the Consumer's record, with the first Log's provenance alone |
| 3 | `later_catalog_sealed_by_the_other_log` | Once the second Log seals it, provenance names both Logs |
| 4 | `removal_sealed_by_one_log` | A removal sealed by the first Log alone removes the Consumer's record while the second Log still holds one |
| 5 | `removal_sealed_by_the_other_log` | Once the second Log seals it, the record stays gone and the other page keeps its Item |

### `e2e-restored-builder`

`a_builder_restored_from_the_served_tree_changes_no_item`: Publisher
`localhost`, Collection `docs`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `publication` | Three pages publish |
| 2 | `builder_restored_from_the_served_tree` | Spake, with a state directory holding only the key seeds, lists the same Item IDs from the served files and signs no new Catalog; nothing of the domain seals; the Consumer's records do not change |
| 3 | `edit_by_the_restored_builder` | An edited page takes a new Item and the others keep theirs; one Item seals |

### `e2e-masked-record`

`a_consumer_resumed_from_a_snapshot_holds_a_masked_record_and_the_replaying_consumers_attested_at`:
Publishers `localhost` and `sub.localhost`, one Log, a replaying Consumer
and, from Epoch 3, a Consumer installed from the Snapshot.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | `localhost`'s Declaration seals |
| 1 | `declaration_scoping_a_subdomain` | A Declaration with `subdomain_scope` naming `sub.localhost`, signed with the owner key, seals |
| 2 | `parent_publishes_a_subdomain_url` | `localhost` publishes text Emissions for its own URL and a `sub.localhost` URL; the Consumer returns `localhost`'s record for the subdomain URL |
| 3 | `subdomain_declares_and_masks_the_parents_record` | `sub.localhost` declares and publishes the same URL. The Log keeps both records, and `localhost`'s records still recompute its Catalog's root. Both Consumers hold the same records and `record`, `removal` and `collection` state, the masked record included. They answer `get_record` alike: the URL with `sub.localhost`'s record, and each record with its Catalog's `generated_at` as `attested_at` |
| 4 | `subdomains_own_record_removed` | `sub.localhost` removes the page; `localhost`'s record stays in the Log; both Consumers agree and find no record for the URL |

### `e2e-label-order`

`a_retracted_label_stays_retracted_against_an_earlier_label_and_both_consumer_paths_agree`:
Publisher `localhost`, Labeler `labeler.localhost`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `publication` | Both sites publish |
| 1 | `label` | Two `wist:spam` Labels on two pages seal |
| 2 | `label_dispute` | A dispute of the first Label seals |
| 3 | `label_removed_by_retraction` | `spake label --retract` seals a retraction of the first Label |
| 4 | `label_with_an_earlier_asserted_at` | A Label with an `asserted_at` earlier than the retraction's, listed in a re-signed Label Feed, seals and applies nothing. The Log's state holds the second page's Label and the dispute alone; the replaying Consumer and one installed from the Snapshot hold that Label and dispute at their sealing heights and answer `get_labels` alike |

### `e2e-withdrawal`

`a_withdrawal_filed_with_two_logs_under_one_legal_basis_is_sealed_by_each`:
Publisher `localhost`, Collection `docs`, two Logs.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | Both Logs seal the Declaration |
| 1 | `publication` | Both Logs serve the page's Payload |
| 2 | `withdrawal_sealed_by_two_logs` | `spake withdraw` deletes the Payload file. `clave withdraw` on each Log, under one legal basis and jurisdiction, seals one `payload_withdrawal` whose details name the Item ID, legal basis and jurisdiction. Each Log holds a `withdrawal` tuple at Epoch 2, keeps the Item as the URL's record and answers 404 for the Payload. The Consumer neither returns nor finds the record; the other page stays |
| 3 | `content_published_again_under_a_fresh_salt` | `spake publish` lists the content as a new Item; each Log seals it; the record names both Logs and is searchable |

### `e2e-domain-capacity`

`a_change_above_the_per_domain_capacity_seals_over_several_epochs_and_each_entry_verifies_alone`:
Publisher `localhost`, Collection `docs`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `labeler_cap_lowered` | `clave param-change` sets `labeler_epoch_entries_max` to 3, effective at the grid instant 170 hours after Epoch 0 |
| 1 | `domain_capacity_lowered` | `domain_epoch_entries_max` is set to 3 at the same instant; the Snapshot state carries both amendments; the grid moves to that instant |
| 2 | `declaration` | The Declaration seals |
| 3 | `publication_within_the_capacity` | One Catalog and two Items seal |
| 4 | `change_above_the_capacity_part_1` | A Catalog removing one page and adding six seals with the removal and one page |
| 5 | `change_above_the_capacity_part_2` | Three pages seal |
| 6 | `change_above_the_capacity_part_3` | Two pages seal |

In Epochs 4 to 6, no Epoch carries more than three Entries of the domain;
each Item names the Catalog and its Inclusion Proof verifies alone; the
status reports the remaining Items waiting with the deferral `capacity`;
the Consumer's records follow each Epoch. At Epoch 6 the Log's records
recompute the Catalog's root, and a Consumer installed from the Snapshot
holds the following Consumer's records.

### `e2e-recovery`

`a_recovery_window_queues_catalogs_of_both_keys_and_settlement_seals_the_latest_survivor`:
Publisher `localhost`, one Log.

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `declaration` | The Declaration seals |
| 1 | `publication_with_a_recovery_key` | `spake recovery-init` declares a recovery key whose seed is written outside the state directory; a page publishes |
| 2 | `recovery_window_opened` | `spake recover` serves a Declaration signed by the recovery key that drops the replaced owner key. The Epoch seals the Declaration and no Catalog or Item; a `recovery_window` tuple stands at height 2 |
| 3 | `catalogs_of_both_keys_queued_in_the_window` | A Catalog signed with the replaced key and Spake's later Catalog under the new key both wait with the deferral `recovery_window`; nothing of the domain seals; the Consumer holds the pre-recovery record alone |
| 4 | `recovery_settlement` | The Epoch sealed 7 days after Epoch 2 seals the surviving Catalog of latest instant with its two Items; the replaced key's Catalog is dropped with `WIST1-E13`; the window closes; the Consumer holds the three pages Spake published and not the page only the replaced key's Catalog lists |
| 5 | `recovery_key_rotation` | `spake recovery-rotate` serves a Declaration signed by the recovery key naming a new one; it takes force and opens a `recovery_window` at Epoch 5 |

### Change-list chains

Three test functions share a setup: Publisher `localhost`, Collection
`docs` with three pages, one Log, `declaration` at Epoch 0 and
`publication` at Epoch 1. Spake then signs several Catalogs, each with a
change list naming its predecessor, before one pull. At Epoch 2 each seals
one Catalog, the Log's records recompute its root, and the Consumer's
records are the latest Catalog's page Items.

| Record | Epoch | Scenario | What the run asserts |
|---|---|---|---|
| `e2e-change-list-chain` | 2 | `list_obtained_by_a_chain_of_change_lists` | Of 16 Catalogs (`change_chain_max`), the pull reads the 16 change lists from the latest back, fetches no tree file and accepts the latest Catalog |
| `e2e-change-list-discarded` | 2 | `chain_discarded_and_list_walked` | Of two Catalogs, the first one's change list is not served; the pull reports `WIST2-E08` with condition `fetch` naming that change list, walks the tree and accepts the latest Catalog |
| `e2e-change-list-chain-beyond-the-bound` | 2 | `chain_beyond_change_chain_max_and_list_walked` | Of 17 Catalogs, the pull either reports `WIST2-E08` with condition `chain`, naming the second Catalog's change list after reading the 16 latest, or leaves the chain unreported after reading at most 16 of them (WIST-2 §5.3); it walks the tree and accepts the latest Catalog |

The test functions are
`a_pull_obtains_the_latest_list_by_a_chain_of_change_chain_max_change_lists_and_fetches_no_tree_file`,
`a_chain_missing_a_change_list_is_discarded_with_wist2_e08_and_the_walk_accepts_the_catalog`
and
`a_base_behind_change_chain_max_change_lists_is_left_or_discarded_for_chain_and_the_walk_accepts_the_catalog`.

## Also exercised without a scenario name

- After every `graven follow` and `graven sync`, each Log's `records` rows
  equal the records the stored Replay materializes when resumed.
- A store's first Log is installed with Tier 1 import (`--tier1`).
- In `e2e-aggregator-keys`, Epoch 2 seals an Item while the Log holds two
  keys and Epoch 4 an Item after the removal; both are searchable,
  Checkpoint 4 is signed by the remaining key alone, and
  `clave verify-history` passes before Clave restarts.
- Objects Spake does not produce are signed by the test with the core
  library and served in place of Spake's: the hijacked, pending and
  `subdomain_scope` Declarations; the out-of-Scope, cross-Collection,
  equal-instant, beyond-the-clock, removed-key and replaced-key Catalogs
  with their tree files and change lists; and the earlier Label with its
  Label Feed.
- `validate_artifacts.py` checks the Publisher's or Labeler's served files
  and the first Log's data directory with the specification's schemas and
  its reference modules in `spec/tools`: the Declaration, Catalogs, tree
  files, Items and Payloads; the Anchor, head and archived Checkpoints,
  entry bundles and Entries; the Snapshot index, manifests, state files and
  `content_digest`; the Aggregator key tuples authenticated from the
  Anchor; every Checkpoint verified under the keys valid at its own height
  and the Snapshot documents under the keys valid at the head. It runs at
  the end of each test, except in `e2e-scope-narrowing` (after Epoch 4)
  and `e2e-recovery` (after Epoch 1).
- The independent Log client verifies the head Checkpoint's signature,
  recomputes the root from the tiles, and checks an Inclusion Proof for
  every leaf and a Consistency Proof from every smaller tree size: on the
  first Log of `e2e-publication` at Epoch 7 under the genesis key, and in
  `e2e-aggregator-keys` at Epoch 5 under the remaining key.

## Obligations exercised

Rows name surfaces of CONFORMANCE.md's "Validation still required" table.
"Exercised" means the behavior ran across the three roles and an
assertion of this run depends on it. No row is thereby closed: each also
requires evidence this run does not supply, such as vector consumption,
independent implementations and boundary or adversarial cases.

| Surface | Exercised here | Not exercised here |
|---|---|---|
| WIST-1 §§3, 4 Items, Catalogs, roots and proofs | Item IDs, Catalog IDs, roots and Inclusion Proofs of served Catalogs recomputed by the core library and by the specification's reference modules | Consumption of the vector files; independent JCS, SHA-256 and Ed25519 implementations |
| WIST-1 §5.1 Collections and Scopes | Collections with one or several Scope prefixes and their own keys under live pulls; an Item outside its Collection's Scope and a removal of another Collection's URL refused with `WIST1-E03`; a Catalog under another Collection's key refused with `WIST1-E02`; narrowing at its sealing height; a pending narrowing removing nothing | Form, count and Entry-bound failures (`WIST1-E14`, `WIST1-E16`, `WIST1-E04`); port coverage; key uniqueness across key lists; a Declaration signed by a Collection key |
| WIST-1 §5.2 Declaration key binding | Admission and replacement of Spake's Declarations under live pulls, and their replay by the Consumer | Duplicate keys; thumbprint mismatches |
| WIST-1 §5.1/§5.2 key directory and activation | A rotation with an overlap; a pending identity supplying no authority and discarded on reversal before activation | Activation at the frozen height or under a zero delay; `next_keys` enforcement; Snapshot resumption inside a pending state |
| WIST-1 §5.1/§5.2 Catalog bindings under recovery | A Catalog under the replaced key queued in the window and dropped at settlement | Each frozen source read alone; the `WIST1-E14`, `WIST1-E02` and `WIST1-E01` distinctions under recovery |
| WIST-1 §5.2 recovery ownership and heads | One recovery Declaration replacing the owner key; a recovery-key rotation | Competing recovery chains; conflicting Declaration groups |
| WIST-1 §5.2 recovery settlement | Window opening; Catalogs of both keys queued; settlement sealing the survivor of latest instant with its Items; the `WIST1-E13` status effect; a recovery-key rotation opening a window | Durable queue recovery across an Aggregator restart; quotas at settlement |
| WIST-2 §§3, 5, 7 Collection pulls | Pulls on Ping; the status object's `collections` (latest, accepted, waiting with deferrals) and rejections; a chain of `change_chain_max` change lists read without tree files; a chain discarded at `fetch` with `WIST2-E08`; a chain beyond `change_chain_max`; `WIST2-E05` regression; an idempotent re-serve | A 304 answer for the Declaration; the 16 384-octet `catalog.json` bound; `WIST2-E01` retries; budget and suspension; `WIST2-E07`; the refusal of a list dropping a held record's URL; `WIST2-E03` as an asserted rejection |
| WIST-2 §3.2 Label Feeds | Label Feeds under live pulls, one re-signed with an added Label ID | Pages; regression state; the target rule |
| WIST-2 §§3.3, 5 Labels | `label` and `dispute` Entries sealed; a definition's treatment; subscription; a retraction; an earlier `asserted_at` applying nothing; `label` and `dispute` tuples; the Consumer's Label and dispute tables from replay and from a Snapshot agreeing | The ingest budget; `WIST2-E06`; the per-Labeler cap with Labels sealed under it; expiry; the `delta` binding; the Tier 1 Label, dispute and Labeler files |
| WIST-3 §§5–6 publication | Archived Checkpoints from genesis; the head equal to its archived Checkpoint; tiles and entry bundles at the head; Payloads served by the Log until a withdrawal | Payload replication to Mirrors |
| WIST-3 §§3.2–3.3, 6.1, 7 sealing and records | Candidates judged at their Epoch with refused Items left unsealed; status deferrals for capacity and recovery; the capacity order; a replaying Consumer and one installed from a Snapshot holding the same latest Catalogs, records, removal states and materialized records, a masked record included; two Logs combined per Publisher and URL | No Item sealed in the Epoch sealing its Payload's withdrawal; the Payload check at an Item's turn; Payload duties over time; the hold |
| WIST-3 §§3.1, 3.3, 6 transport parsing | Tiles and entry bundles decoded by Graven and by the independent client, the root recomputed from the tiles | Bound refusals; leap seconds; the full year range |
| WIST-3 §§3.4, 5 Checkpoint notes | A rotation Checkpoint with two signature lines; Checkpoints after the removal with the remaining key's line alone; distinct note key IDs | Malformed notes; key-act collisions; a Witness's configuration of a rotated verifier key |
| WIST-3 §§3.1, 4–5 proofs and head adoption | Incremental syncs across every sealed Epoch; the independent client's Inclusion and Consistency Proofs | Rollback, equivocation and sequence failures |
| WIST-3 §§7–8 Snapshot position | Cold starts at the Snapshot of the sealed Epoch followed by incremental sync | Re-fetch across sources; a file at the selected path stating another Epoch |
| WIST-3 §§3.4, 5 Log key succession | A key added and the genesis key removed in an operating Log that keeps serving and producing Snapshots; `clave verify-history` passing; sealing under the remaining key after a restart; a replaying Consumer following throughout | Succession to a new Log through an Anchor's `predecessor`; the Aggregator's refusal to seal a key-act failure |
| WIST-3 §§3.4, 7–8 Snapshot key authentication | A cold start from a Snapshot produced after the genesis key's removal; a Snapshot signed by the removed key rejected with `WIST3-E04`, the Log left unregistered | A signer admitted above the Snapshot's Epoch; re-fetch across Mirrors; a sharded Snapshot |
| WIST-4 §5 inclusion | The per-domain capacity, lowered by parameter change, splitting one Catalog's Items over three Epochs, Catalog first, then removals, then pages | Labels and disputes under the capacity or the per-Labeler cap; backlog and overload; the inclusion ceiling; the discovery sealing deadline |
| WIST-4 §§3, 5.1 governance | Aggregator key registration and removal; parameter changes held in Snapshot state; withdrawals sealed by one Log and by two Logs; all replayed by the Consumer and restored by Consumers installed from Snapshots | Suffix-list snapshots beyond the one each Log pins at Epoch 0; per-Registrable-Domain capacity rejection in a replaying Consumer; a repeated Registry Update ID |
| WIST-5 §§3–7 Emissions | `spake emit` over marked pages and over a served directory, omitting an out-of-Scope page; complete and incremental streams written by the test, text and removal lines included; a due Catalog signed with no stream; unchanged content keeping its Item across Spake runs, a restored state directory included | The refusals of §3.3 and §6.3; an unmarked page emitting nothing |

No assertion of this run concerns WIST-1 §4 canonicalization, the WIST-2
§7 and WIST-4 §5 quotas, WIST-2 §§6, 8 scheduling and redirects, WIST-3
§§6–7 Snapshot directories and sharding, the WIST-3 §5 Witness quorum or
the WIST-3 §5 Mirror list.

## Known limits at these revisions

- Every role uses plain HTTP to loopback hosts (`--allow-http` in Clave,
  in `spake ping` and in Graven), which WIST-2 §8 forbids; Clave's and
  Spake's READMEs list it as a known deviation.
- The run seals with `clave seal --at` against `clave serve --no-seal`, so
  Clave's own sealing at each cadence grid instant is not exercised.
- Graven installs a Snapshot only for a Log it has no sync record for;
  a Consumer holding state never catches up through a Snapshot.
- Graven's source reads no Log Anchor `predecessor`, so succession to a
  new Log is unimplemented.
- Graven reads no Mirror list, and the run configures no Mirror. Clave
  writes `/log/mirrors.json` only on `clave mirror`, so the artifact
  check's Mirror list verification does not run.
- Graven does not implement a Consumer holding a subset of shards, and the
  run configures no sharding.
- The run configures no Witness: Graven's roster is empty, the quorum is
  0, and every acceptance is recorded as unwitnessed.
- `e2e-change-list-chain-beyond-the-bound` accepts either outcome WIST-2
  §5.3 permits, so the run does not record which one Clave takes.
