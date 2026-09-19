# Three-role functional integration

One reproducible run drives Spake (Publisher and Labeler), Clave
(Aggregator) and Graven (Consumer) as separate processes over HTTP and
asserts the Consumer's state, as its MCP tools report it, after every
step. It establishes that the three roles interoperate functionally at the
revisions below. It is not a load test, and it does not replace the
acceptance evidence the specification's
[CONFORMANCE.md](../../spec/CONFORMANCE.md) requires before stable
publication.

## Revisions

| Repository | Revision |
|---|---|
| `spec/` | `b622603f95332cb410692425740e08cfb13f8f8c` |
| `core/` | `3d500ed7bba71e60f8ee0c4f0c326544cac89fcb` |
| `spake/` | `b3ad50067d56ffef6cf6a6774e51c257e9edb28a` |
| `clave/` | `ab92b9eb10688d3cf8f75cc16976c70b84951b27` |
| `graven/` | `42421a07fdc24d569aea9954df28b1921b8ec423` |

Every working tree was clean. The run passed on 2026-09-19.

## Reproducing it

With the five repositories checked out side by side at those revisions and
the Python packages `jsonschema`, `rfc8785` and `cryptography` installed:

```
(cd spake && cargo build) && (cd clave && cargo build)
cd graven
SPAKE_BIN=$PWD/../spake/target/debug/spake \
CLAVE_BIN=$PWD/../clave/target/debug/clave \
WIST_SPEC_DIR=$PWD/../spec \
cargo test -p e2e --test e2e
```

The test serves the fixture sites through a local HTTP proxy, seals with
`clave seal --at` at explicit hourly grid instants so that Log time does
not depend on the wall clock, and writes `target/tmp/e2e-run.json`: each
repository's revision and whether its tree carried changes, and the
scenarios below with the Epoch each sealed at. Two Logs run; the Epoch
numbers are the first Log's.

## Scenarios

| Epoch | Scenario | What the run asserts |
|---|---|---|
| 0 | `publication` | Sites published by Spake are pulled, sealed and served by the Consumer's `search`, `get_record` and `get_extract` |
| 1 | `revision` | A revised page replaces its record; `prev` chains to the sealed tip |
| 2 | `label_dispute` | A Labeler's Labels, a dispute and a Label definition seal and appear in `get_labels` and `list_labelers` |
| 3 | `payload_withdrawal` | `clave withdraw` seals a `payload_withdrawal` in one Log; the record survives with the other Log's provenance alone, since a withdrawal reaches only the Log that sealed it |
| 4 | `publisher_key_rotation` | `spake rotate` installs the next Key Set; the outgoing key expires at the end of the overlap and Deltas under the new key seal |
| 5–6 | `hijacked_declaration`, `hijack_reversal` | A Declaration published from the web host alone goes pending and is reversed before activation |
| 7–8 | `label_one_epoch_old`, `label_two_epochs_live` | The default ranking profile counts a persistence Label only once it is live at two consecutive heights |
| 9 | `url_deletion` | `spake build --remove` seals a deletion; the record is gone while the domain's other records remain |
| 10 | `aggregator_restart` | Clave is killed with admitted Deltas unsealed and restarted on the same data directory; the Log Anchor is byte-identical, `clave verify-history` passes, the pending work seals and the Consumer's head never moves backwards |
| 11 | `publisher_restart` | Spake publishes again from its persisted state in a cleared environment; the new Delta's `prev` is the sealed tip |
| 11 | `default_profile_persistence` | `graven profile use` selects a ranking profile that a new Consumer process applies to a query naming none |
| 12–14 | `publisher_key_recovery_window`, `delta_queued_inside_the_recovery_window`, `publisher_key_recovery_settlement` | `spake recover` opens a recovery window; Deltas queue while it is open; at settlement the recovered-key Deltas materialize and the pre-recovery Delta is rejected as `WIST1-E13` |
| 15 | `recovery_key_rotation` | `spake recovery-rotate` replaces the recovery key and opens its own window |
| 16 | `log_key_addition` | `clave log-key add` seals an `aggregator_key_add`; the head and archived Checkpoint carry a signature line from the admitting key and one from the admitted key; records sealed afterwards are served |
| 18 | `log_key_removal_of_the_genesis_key` | The Checkpoint sealing the removal and every later one is signed by the remaining key alone; the Consumer follows across it; Clave verifies its history, restarts and seals again |

Also exercised without a scenario name: Label subscription
(`graven subscribe`), per-query profile selection, a Consumer MCP server
restarted between steps, deduplication of one Delta sealed by two Logs,
validation of the emitted artifacts against the specification's schemas by
its Python reference tooling, and an independent tiled-log client verifying
the head Checkpoint and an Inclusion Proof under the genesis key before the
Log-key rotation and under the remaining key at the final head.

## Obligations exercised

Rows name surfaces of CONFORMANCE.md's "Validation still required" table.
"Exercised" means the behavior ran across the three roles in this run; no
row is thereby closed, because each also requires evidence this run does
not supply (independent implementations, boundary and adversarial cases).

| Surface | Exercised here | Not exercised here |
|---|---|---|
| WIST-1 §5.1/§5.2 key directory and activation | Ordinary rotation with overlap; a pending identity supplying no authority and reversed before activation | Activation of a fresh identity at its frozen height; Snapshot resumption inside a pending state |
| WIST-1 §5.2 recovery ownership, heads and settlement | Window opening, queued Deltas, settlement, `WIST1-E13` status effect, actual survivor sealing, recovery-key rotation | Competing recovery chains; Aggregator restart inside an open window; quotas at settlement |
| WIST-2 §§3–5, 7 Feed pulls | Live pulls, Declaration refresh across a rotation, status reporting | Domain mismatch, unusable-Feed classification, redirects |
| WIST-2 §§3.3, 5 Labels | Live Label Feed pulls, `label` and `dispute` Entries, Label tier files, subscription and persistence in ranking | Per-Labeler caps, `WIST2-E06` reporting, expiry |
| WIST-3 §§5–6 publication | Entries retrievable before each Checkpoint; partial tiles and bundle at the head; archived Checkpoints from genesis; publication resumed after a kill | Payload replication to Mirrors |
| WIST-3 §§3.1, 4–5 proofs and head adoption | Every Checkpoint verified in order across 21 Epochs of the first Log; an independent client's Consistency and Inclusion checks | Rollback, Equivocation and sequence failures (covered by vector replay in core and Graven, not by this run) |
| WIST-3 §§7–8 Snapshot position | Cold start from a Snapshot followed by incremental sync | A Snapshot taken after a Log-key act (see below) |
| WIST-3 §§3.4, 5 Log key succession | An addition and the genesis key's removal sealed by the Aggregator and followed by a replaying Consumer; Checkpoints signed by every valid held key | Succession to a new Log through an Anchor `predecessor` |
| WIST-4 §§3, 5.1 governance | Key registration and removal, a withdrawal, sealed, served and replayed | Parameter changes and suffix-list snapshots in this run; per-Registrable-Domain capacity rejection |

Not exercised at all: the Witness quorum (it is 0 throughout and every
acceptance is recorded as unwitnessed), Mirrors, quotas under load, and
scheduling hints.

## Known limits at these revisions

- A Consumer cold-starting from a Snapshot after the genesis key's removal
  fails: Graven verifies the Snapshot index, manifest and state file under
  the Anchor's genesis key alone, and the specification does not yet say
  what authenticates a Snapshot's `aggregator_key` tuples to a party
  holding only the Anchor. A Consumer replaying from genesis is unaffected.
  The run's artifact validation is performed before the rotation for the
  same reason.
- Graven ignores a Log Anchor's `predecessor`, so succession is
  unimplemented.
- Graven applies the transport bound per decoded entry bundle rather than
  while streaming.
- Graven fetches Public Suffix List snapshots, Payloads and Label
  definitions from its first configured source only.
- Clave seals when `clave seal` is invoked and has no scheduler of its
  own, so issuing a Checkpoint at every grid instant depends on the
  operator's timer; a missed instant is not backdated.
- Clave's publication repair checks every tile the head's tree size
  requires on each seal and start, which is linear in tree size.
