# ADR-0003: Naming policy — protocol-named libraries, codenamed services

**Status:** accepted, amended by [ADR-0004](0004-all-rust-stack.md) (2026-08-09: Graven's registry coordinates move from PyPI to crates.io + npm; Holden to crates.io), amended by [ADR-0007](0007-signed-publications-and-consumer-trust.md) (2026-09-16: the Holden codename is retired with the auditor role) · **Date:** 2026-08-05

## Context

Naming supports [ADR-0001](0001-repo-layout.md)'s separation of specification
authority from implementations. A service sharing the protocol's name can
make alternatives appear to be forks; Certificate Transparency instead has
Trillian, Sunlight and Sigsum, while go-ipfs/js-ipfs became Kubo/Helia.
Protocol-named libraries retain discoverability without claiming service
identity: `matrix-sdk` and `ruma` illustrate independent coexistence.

## Decision

- **WIST** is the protocol brand only: the spec suite, the version wire
  field, the domain — never the name of a repo, service, or company.
- **Libraries** carry the protocol name. The primitives crate is
  `wist-core`. In Rust, a language suffix belongs to the repo, not the
  crate (cf. rust-lang/git2-rs publishing `git2`).
- **Runnable services** (aggregator, publisher CLI) take distinct
  codenames, chosen when their repos start. The repo carries the
  codename as the software's public identity.
- **Operated products** (services anyone runs as a product, first-party
  included) take their own codenames, distinct from the reference
  implementations.
- **Conformance independence** follows
  [ADR-0002](0002-implementation-stack.md#decision): `wist-core` is checked
  against the independent reference, not treated as the specification.

## Consequences

- `cargo add wist-core` stays self-explanatory; discoverability for
  libraries is preserved.
- Second implementations of any service start from a level field: no
  reference service occupies the protocol's name.
- Naming work (codename search, collision check) happens when each
  service repo starts.
- Reserved crate names (`wist-verify`, `wist-log`, `wist-client`) are
  placeholder-published with a README pointing at the spec.

## Addendum (2026-08-08): service codenames chosen

- **Spake** — publisher
- **Clave** — aggregator
- **Holden** — auditor
- **Graven** — consumer

Published artifacts may carry `wist-` prefixes as registry coordinates
(`wist-spake`/`wist-clave` on crates.io, `wist-holden`/`wist-graven`
on PyPI, per the ADR-0002 language split); the codename remains the
software's name in all prose. The prefix supplies registry provenance.

## Alternatives considered

- **Full codename family** (everything, including libraries, under one
  codename): strongest independence signal, matches CT precedent
  exactly, but forfeits library discoverability for no observed risk —
  protocol-named libraries have not exhibited the normativity failure.
  Rejected.
- **Everything wist-prefixed, including services**: simplest, but
  reproduces the go-ipfs failure — namespacing alone did not prevent
  "the Go one is IPFS". Rejected.
