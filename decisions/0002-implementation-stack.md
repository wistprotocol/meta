# ADR-0002: Rust for services and core, Python for data-side components

**Status:** accepted · **Date:** 2026-08-02

## Context

The components split into two natural groups. Services and tooling that
parse untrusted network input and must run for decades: the shared
primitives (`core/`), the aggregator, and the publisher CLI.
Data-side components whose
critical dependencies live in the Python ecosystem: the consumer
(SQLite FTS5, pyarrow/Parquet, local embeddings, MCP SDK) and the
auditor (trafilatura extraction, warcio WARC evidence, shingle
similarity).

For infrastructure meant to run for decades, the deciding factors
are, in order: (1) one implementation of the primitives,
consumed natively by both the Rust services and the Python components;
(2) boring operations — single static binaries, trivial deploys;
(3) maintainer fluency, which outweighs marginal technical
differences between otherwise adequate languages. Memory
safety on hostile-input parsing matters but does not discriminate
among the candidates considered: all are memory-safe.

## Decision

- **Rust** for `core/`, the aggregator, and the publisher CLI.
  - `core/` implements the primitives (JCS canonicalization, Ed25519
    envelopes, Merkle trees, block parsing) once. The load-bearing
    extra target is **PyO3**: the Python components consume the same
    primitives natively, and the Rust→Python path (PyO3 + maturin,
    abi3 wheels) is mature with heavy precedent (cryptography,
    pydantic-core, polars, tokenizers). **WASM** is kept as cheap
    optionality, not as justification: no spec document requires
    browser verification, and if a browser verifier ships it may be
    better served by an independent TypeScript implementation —
    independent implementations checking each other is the protocol's
    value proposition.
  - Services keep concurrency deliberately simple (seal hourly, serve
    static files); async Rust surface is minimized.
- **Python** for the consumer and the auditor — chosen for ecosystem gravity:
  their critical dependencies have no equivalent elsewhere.
- **Conformance independence**: a small pure-Python implementation of
  the primitives (grown from the spec repo's `tools/validate_examples.py`)
  is maintained as the conformance reference, so the spec is always
  checked by an implementation independent of the production Rust core.
  The reference is never production code — that independence is its
  entire function.
- Tier 0 embeddings: a small multilingual model quantized to int8 ONNX,
  so consumers embed queries locally without a GPU (the snapshot
  manifest already declares model/version/quantization).

## Consequences

- One production implementation of the primitives everywhere (via PyO3)
  plus one independent reference (pure Python): consistency in
  production, independence in verification.
- Single static binaries for the aggregator and the publisher CLI —
  cheap, boring operations; zero-dependency tooling for site operators.
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
  on build speed and contributor pool, with ready CT precedent to
  borrow from. Loses decisively on the bindings story — cgo/c-shared
  into Python is hostile territory with no maturin equivalent — and on
  maintainer fluency. Those two factors carried the decision.
- **Pure-Python primitives for the data side** (pynacl for Ed25519,
  own JCS/Merkle, validated against the shared test vectors): adequate
  performance at consumer/auditor volume, and the vectors — not a
  shared binary — are the protocol's real cross-implementation
  consistency mechanism. Rejected: a third implementation to maintain,
  and it would blur the conformance reference's independence by
  putting near-identical Python primitives in production.
- **Rust everywhere**: would relocate the consumer and auditor away from
  best-in-class extraction/data libraries for no gain. Rejected.
- **TypeScript for services**: fine for a future browser verifier or
  npm publisher tooling; wrong operational profile for the aggregator.
  Revisit only if publisher-side web tooling demands it.
