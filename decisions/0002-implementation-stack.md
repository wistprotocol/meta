# ADR-0002: Rust for services and core, Python for data-side components

**Status:** accepted, amended by [ADR-0004](0004-all-rust-stack.md) (2026-08-09: consumer and auditor moved to Rust; PyO3 dropped unless third-party demand); amended by Addendum (2026-09-18: the protocol term Block renamed Epoch) · **Date:** 2026-08-02

## Context

The original split placed shared primitives, the aggregator and publisher
CLI in long-running services/tooling, and the consumer and auditor near
Python data libraries: SQLite FTS5, pyarrow/Parquet, local embeddings, MCP,
trafilatura, warcio and shingle similarity. ADR-0004 supersedes the latter
placement and its dependency assumptions.

The deciding factors, in order, were shared native primitives, static-binary
deployment and maintainer fluency. All candidates were memory-safe, so
hostile-input safety alone did not distinguish them.

## Decision

- **Rust** for `core/`, the aggregator, and the publisher CLI.
  - `core/` implements the primitives (JCS canonicalization, Ed25519
    envelopes, Merkle trees, block parsing) once. **PyO3** exposes them
    natively to Python through maturin/abi3 wheels. **WASM** remains
    optional: browser verification is not required and may benefit from
    an independent TypeScript implementation.
  - Services keep concurrency deliberately simple (seal hourly, serve
    static files); async Rust surface is minimized.
- **Python** for the consumer and the auditor — chosen for ecosystem gravity:
  their critical dependencies have no equivalent elsewhere.
- **Conformance independence**: a small pure-Python implementation of
  the primitives (grown from the spec repo's `tools/validate_examples.py`)
  is maintained as an independent conformance reference, never production
  code.
- Tier 0 embeddings: a small multilingual model quantized to int8 ONNX,
  so consumers embed queries locally without a GPU (the snapshot
  manifest already declares model/version/quantization).

## Consequences

- One production implementation of the primitives everywhere (via PyO3)
  plus one independent reference (pure Python): consistency in
  production, independence in verification.
- Static binaries simplify aggregator and publisher installation.
- Every parser of hostile input (deltas, blocks, feeds) gets Rust's
  stricter modeling — exhaustive matching, no nil, refactoring safety
  under continuous change. Incremental over Go, not decisive.
- Costs accepted: Rust compile times; a smaller contributor pool than
  Go/Python; PyO3 packaging adds build complexity to the Python repos
  (maturin + abi3 wheels bound it); crates-ecosystem dependency churn —
  the dependency tree stays minimal and pinned.

## Alternatives considered

- **Go for services** (the conventional CT-lineage choice — Trillian,
  Sunlight, Sigsum): equal on static binaries and memory safety, ahead
  on build speed and contributor pool. Rejected for weaker Python
  packaging through cgo/c-shared and lower maintainer fluency.
- **Pure-Python primitives for the data side** (pynacl for Ed25519,
  own JCS/Merkle, validated against the shared test vectors): adequate
  performance at consumer/auditor volume. Rejected: a third implementation to maintain,
  and it would blur the conformance reference's independence by
  putting near-identical Python primitives in production.
- **Rust everywhere**: would relocate the consumer and auditor away from
  best-in-class extraction/data libraries for no gain. Rejected.
- **TypeScript for services**: fine for a future browser verifier or
  npm publisher tooling; wrong operational profile for the aggregator.
  Revisit only if publisher-side web tooling demands it.

## Addendum (2026-09-18): Block renamed Epoch

The specification's draft ADR-0048 renamed the protocol term Block to
Epoch: since the single-tree Log a Block is no object, only the interval
of the Log between two consecutive Checkpoints, and key-transparency logs
call that interval an epoch. No behavior changed. Where this record says
"block parsing" and "blocks" among the hostile inputs, read the Log's
Epochs, served since the same revision as tiles and entry bundles.
