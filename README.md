# DeltaCommons — Project Meta

Cross-repo engineering decisions and the project map for DeltaCommons: an
open, verifiable, push-based web index protocol for local AI agents.

This repo answers "why do we build it like this?". Protocol decisions —
"why is the protocol like this?" — live in the spec repo's own
`decisions/` folder and never here.

## Project map

| Component | What it is | Status |
|-----------|------------|--------|
| `spec/` | Protocol specifications (DC-1..DC-4), JSON Schemas, test vectors, conformance tooling | v1.0.0-draft |
| `core/` | Rust: shared primitives — JCS envelopes, Ed25519, Merkle, block parsing (WASM + PyO3 targets) | planned |
| aggregator | Rust: ingest endpoint, validation, block sealing, snapshots, status endpoint | planned |
| consumer | Python: log sync, local index materialization, MCP server | planned |
| publisher | Rust: site operator CLI — keygen, delta generation, feed maintenance | planned |
| auditor | Python: sampling, re-fetch, similarity scoring, WARC evidence, audit records | planned |

Mirrors need no repo: a mirror is any static file server.

## Decisions

- [ADR-0001](decisions/0001-repo-layout.md) — multi-repo layout, spec separate from implementations
- [ADR-0002](decisions/0002-implementation-stack.md) — Rust for services and core, Python for data-side components

## License

Documentation in this repo: CC-BY 4.0 (see [LICENSE](LICENSE)).
