# WIST — Project Meta

Cross-repo engineering decisions and the project map for the WIST Protocol:
an open, verifiable, push-based web index protocol for local AI agents.

This repo answers "why do we build it like this?". Protocol decisions —
"why is the protocol like this?" — live in the spec repo's own
`decisions/` folder and never here.

## Project map

| Repo | What it is | Status |
|------|------------|--------|
| `spec/` | Protocol specifications (WIST-1..WIST-4), JSON Schemas, test vectors, conformance tooling | v1.0.0-draft |
| `core/` | Rust: shared primitives — JCS envelopes, Ed25519, Merkle, block parsing | v0.1.1 (WIST-1..3 primitives + signing helpers) |
| `spake/` | Rust: publisher CLI — keygen, sitemap-driven delta generation, feed maintenance, ping | in development (signed path, sitemap only) |
| `clave/` | Rust: aggregator — ingest+verify, block sealing, checkpoints, tier0 snapshots, status endpoint | in development (no quotas/sanctions/tier1) |
| `graven/` | Rust: consumer — verified log sync, local index materialization, MCP server; ships as npm-wrapped binary | in development (tier0, single log) |
| `holden/` | Rust: auditor — sampling, re-fetch, similarity scoring, WARC evidence, audit records | planned |

The full pipeline (fixture site → Spake → Clave → Graven → MCP query)
runs as Graven's end-to-end test, with emitted artifacts validated by
the spec repo's independent Python reference.

Mirrors need no repo: a mirror is any static file server.

## Decisions

Amendment convention: a substantive change lands as a new ADR carrying
an `**Amends:**` header; an additive clarification lands as a dated
`## Addendum (YYYY-MM-DD)` section inside the ADR it extends. In both
cases the amended ADR's `**Status:**` line gains
`amended by <ref> (date: one-line summary)`, so a stale decision is
never read as current.

- [ADR-0001](decisions/0001-repo-layout.md) — multi-repo layout, spec separate from implementations
- [ADR-0002](decisions/0002-implementation-stack.md) — Rust for services and core, Python for data-side components
- [ADR-0003](decisions/0003-naming-policy.md) — protocol-named libraries, codenamed services
- [ADR-0004](decisions/0004-all-rust-stack.md) — first-party stack all-Rust (amends ADR-0002)

## License

Documentation in this repo: CC-BY 4.0 (see [LICENSE](LICENSE)).
