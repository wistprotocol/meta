# ADR-0003: Naming policy — protocol-named libraries, codenamed services

**Status:** accepted · **Date:** 2026-08-05

## Context

The value of a transparency protocol is independent implementations
checking each other. Where a spec and a binary share a name, the binary
quietly becomes normative and second implementations feel like forks.
CT got this right — RFC 6962 stayed neutral while Trillian, Sunlight and
Sigsum interoperate without anyone asking whose code is authoritative.
IPFS and Matrix had to walk it back (go-ipfs → Kubo, js-ipfs → Helia)
purely to break "the Go one is IPFS".

The ecosystem splits cleanly by artifact type. Libraries named after
the protocol they implement are the norm and have never become
normative — and the precedent holds even first-party: matrix.org wrote
the Matrix spec and publishes `matrix-sdk`, Bluesky wrote AT Protocol
and publishes `@atproto/*`, yet independent competitors coexist
(`matrix-sdk` and `ruma`). Third-party cases (rust-bitcoin's `bitcoin`
crate, `http`, `webpki`, `libp2p`) show the weaker half: protocol-named
libraries are not mistaken for their specs. A library's authority comes
from conformance. The normativity collapse happens to daemons people
*run* — Synapse, go-ipfs — never to protocol-named libraries
implementers consume.

## Decision

Hybrid stance:

- **WIST** is the protocol brand only: the spec suite, the version wire
  field, the domain — never the name of a repo, service, or company.
- **Libraries** carry the protocol name. The primitives crate is
  `wist-core`. In Rust, a language suffix belongs to the repo, not the
  crate (cf. rust-lang/git2-rs publishing `git2`).
- **Runnable services** (aggregator, publisher CLI) take distinct
  codenames, chosen when their repos start. The repo carries the
  codename (`matrix-org/synapse`, `google/trillian`,
  `letsencrypt/boulder` pattern), because the repo name is the public
  identity of the software.
- **Operated products** (services anyone runs as a product, first-party
  included) take their own codenames, distinct from the reference
  implementations, so "running a WIST log" never collapses into
  "running our software".
- **Guard on "core"** (the Bitcoin Core failure mode): the spec repo
  maintains an independent pure-Python conformance reference
  (ADR-0002), so `wist-core` can never quietly become the oracle.

## Consequences

- `cargo add wist-core` stays self-explanatory; discoverability for
  libraries is preserved.
- Second implementations of any service start from a level field: no
  reference service occupies the protocol's name.
- Naming work (codename search, collision check) happens when each
  service repo starts.
- Reserved crate names (`wist-verify`, `wist-log`, `wist-client`) are
  placeholder-published with a README pointing at the spec.

## Alternatives considered

- **Full codename family** (everything, including libraries, under one
  codename): strongest independence signal, matches CT precedent
  exactly, but forfeits library discoverability for no observed risk —
  protocol-named libraries have not exhibited the normativity failure.
  Rejected.
- **Everything wist-prefixed, including services**: simplest, but
  reproduces the go-ipfs failure — namespacing alone did not prevent
  "the Go one is IPFS". Rejected.
