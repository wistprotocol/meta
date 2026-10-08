# WIST — Project Meta

Cross-repo engineering decisions and the project map for the WIST Protocol:
an open, push-based index of signed web publications for local AI agents.

Protocol design decisions belong in [spec/decisions](../spec/decisions).

**Status: draft under implementation and validation.** Implemented
capabilities below do not imply complete protocol conformance or production
readiness; see the specification's [publication policy](../spec/PUBLICATION.md).

## Project map

| Repo | What it is | Status |
|------|------------|--------|
| `spec/` | Protocol specifications (WIST-1..WIST-5), JSON Schemas, test vectors, conformance tooling | draft |
| `core/` | Rust: shared primitives, signing, Catalogs, Items and tree files, Log replay, Snapshot state, parameter schedules | v0.3.0 |
| `spake/` | Rust: publisher CLI — Emissions from marked pages, Collections, signed Catalogs and Items, key rotation/recovery, Label Feed, ping | v0.2.0 |
| `clave/` | Rust: aggregator — Collection pulls, Epoch sealing, Checkpoints, tier0/tier1 Snapshots, quotas, parameter changes, Labels and withdrawals | v0.2.0 |
| `graven/` | Rust: consumer — Snapshot and incremental sync, tier0/tier1, multiple Logs, Labels, ranking profiles, MCP queries, embedding packs | v0.2.1 |
| `deploy/` | Host layer that turns any Ubuntu 24.04 machine into an aggregator serving over HTTPS, plus provider modules that rent one (Hetzner Cloud today), the same install for every operator | install, upgrade and restore proven on rented hosts |

The publication and query pipeline (fixture site → Spake → Clave → Graven → MCP query)
runs as Graven's end-to-end test, with emitted artifacts validated by
the spec repo's independent Python reference. There is no auditor role
(ADR-0007).

Validation records: [three-role functional integration](validation/three-role-integration.md), [capacity baseline under declared publication](validation/capacity-baseline-2026-10-07.md)
— the revisions, scenarios and specification obligations one
multi-process run exercised, and the limits known at those revisions.

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
- [ADR-0006](decisions/0006-clave-ingestion-architecture.md) — Clave ingestion stages, partitions, workers, transaction boundaries and the many-Log path
- [ADR-0007](decisions/0007-signed-publications-and-consumer-trust.md) — signed publications, no auditor role, labelers as publishers, consumer ranking profiles (amends ADR-0001, 0003, 0004, 0005, 0006)
- [ADR-0008](decisions/0008-declared-publication.md) — publication emitted at the content's source, Collections with an enforced Scope, signed Catalogs, one branch and one release (amends ADR-0005, 0006, 0007)

## License

Documentation in this repo: CC-BY 4.0 (see [LICENSE](LICENSE)).
