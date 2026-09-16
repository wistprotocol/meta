# WIST — Project Meta

Cross-repo engineering decisions and the project map for the WIST Protocol:
an open, verifiable, push-based web index protocol for local AI agents.

Protocol design decisions belong in [spec/decisions](../spec/decisions).

**Status: draft under implementation and validation.** Implemented
capabilities below do not imply complete protocol conformance or production
readiness; see the specification's [publication policy](../spec/PUBLICATION.md).

## Project map

| Repo | What it is | Status |
|------|------------|--------|
| `spec/` | Protocol specifications (WIST-1..WIST-4), JSON Schemas, test vectors, conformance tooling | v1.0.0-draft |
| `core/` | Rust: shared primitives, signing, WIST-4 audit math and governance replay | v0.2.0; service integration incomplete |
| `spake/` | Rust: publisher CLI — sitemap/RSS discovery, signed deltas, feeds, key rotation/recovery, appeals, ping | in development; removal requires explicit or origin-confirmed deletion |
| `clave/` | Rust: aggregator — ingest, Block sealing, checkpoints, tier0/tier1 snapshots, quotas and governance | in development; parameter-schedule replay implemented, sanction replay and audit integration incomplete |
| `graven/` | Rust: consumer — snapshot and incremental sync, tier0/tier1, multiple Logs, MCP queries, embedding packs | in development; governance and audit-state verification incomplete |
| `holden/` | Rust: auditor — sampling, re-fetch, similarity scoring, WARC evidence, audit records | planned |

The publication and query pipeline (fixture site → Spake → Clave → Graven → MCP query)
runs as Graven's end-to-end test, with emitted artifacts validated by
the spec repo's independent Python reference. It does not exercise a live
Holden implementation.

Mirror and plugin repository policy: [ADR-0001](decisions/0001-repo-layout.md#decision).

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
- [ADR-0005](decisions/0005-capacity-model-and-baseline.md) — Clave capacity scenarios, targets, cost model and baseline procedure

## License

Documentation in this repo: CC-BY 4.0 (see [LICENSE](LICENSE)).
