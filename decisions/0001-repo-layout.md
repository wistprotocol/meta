# ADR-0001: Multi-repo layout, spec separate from implementations

**Status:** accepted · **Date:** 2026-08-02

## Context

WIST spans a protocol specification and several independent
components (aggregator, consumer, publisher tooling, auditor). The
suite's core guarantee is that the aggregator is substitutable and any
independent implementation can interoperate from the documents alone.
History shows that when a spec and its reference implementation share a
repo, the implementation quietly becomes the spec — its bugs turn into
de facto normative behavior (see Matrix's retroactive speccing of
Synapse behavior; cf. CACM, "It Takes a Village"). Mature protocol
projects (Matrix, OpenTelemetry, ActivityPub/W3C, Certificate
Transparency) keep spec and implementations in separate repositories.

## Decision

One repository per concern:

- `spec/` — the protocol documents, JSON Schemas, deterministic test
  vectors, and conformance tooling. Everything an independent
  implementer needs to prove conformance, and nothing else.
- `meta/` — this repo: cross-repo engineering decisions and the project
  map.
- Implementation repos — the shared primitives, the aggregator, the
  consumer, the publisher tooling, and the auditor — each created when
  its implementation starts.

Mirrors get no repo — by design a mirror is any static file server.
Platform plugins (e.g. a CMS publisher plugin) get their own repos on
their platform's release cycle.

## Consequences

- The spec stays implementation-agnostic; conformance is defined by
  documents and vectors, never by "what the aggregator does".
- Spec and implementations version and release independently; licenses
  stay clean per repo (CC-BY 4.0 for documents, code licenses per
  implementation repo).
- Each implementation repo keeps its own ADRs for purely internal
  decisions; cross-repo decisions land here in `meta/`.
- Slightly more repo-management overhead than a monorepo; accepted as
  the cost of the boundary.

## Alternatives considered

- **Monorepo**: lower initial friction, but
  extracting repos later costs history and links, and the
  spec/implementation boundary erodes exactly when adoption depends on
  it. Rejected.
- **Spec + reference implementation together, tools separate**: the
  worst of both — the pairing that creates de facto normativity is the
  one kept together. Rejected.

## Addendum (2026-08-09)

The publisher (Spake) was implemented alongside the aggregator and
consumer rather than after them: the end-to-end pipeline cannot be
exercised without a working publisher. Implementation repos now exist
as `core/`, `spake/`, `clave/`, `graven/`; only the auditor (Holden)
has not been started.
