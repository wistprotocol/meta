# ADR-0004: First-party stack goes all-Rust (consumer and auditor)

**Status:** accepted, amended by [ADR-0007](0007-signed-publications-and-consumer-trust.md) (2026-09-16: no auditor is built) · **Date:** 2026-08-09 · **Amends:** ADR-0002, ADR-0003

## Context

ADR-0002 placed the consumer (Graven) and auditor (Holden) in Python
for ecosystem gravity. Re-examined before any
Python code existed for them, both rationales failed verification:

- Consumer: rusqlite covers FTS5, arrow-rs/polars cover Parquet,
  ort/fastembed cover ONNX embeddings, and an official Rust MCP SDK
  exists. Over MCP the language is invisible to the agent client
  (stdio subprocess); what differs is installation, and a prebuilt
  binary is the most foolproof artifact.
- Auditor: WIST-2 §12 makes the observed-text procedure a mandatory
  byte-level scan — trafilatura or any boilerplate extractor would be
  non-conforming, so ADR-0002's central Python argument is void.
  Three duties actively favor Rust: untailored UAX #29 segmentation
  for the WIST-4 similarity metric (Python stdlib lacks UAX #29;
  PyICU's dictionary segmentation would violate the spec),
  ECVRF-EDWARDS25519-SHA512-TAI (RFC 9381 crates are more mature in
  Rust), and the 128-bit sampling arithmetic (native u128). The WARC
  duty is write-only, format unpinned, hand-rollable; warcio's
  strength (reading/tooling) is never exercised. The auditor is also
  a decades-running service parsing hostile web input, operated by
  third parties — ADR-0002's own service criteria (static binary,
  boring operations) apply to it.

## Decision

- Graven (consumer) and Holden (auditor) are Rust, consuming
  `wist-core` directly.
- Graven distribution: prebuilt binaries (linux/mac/win × x86/arm)
  via a cargo-dist release matrix, wrapped in an npm package
  (`wist-graven`) whose postinstall fetches the platform binary; npx
  is the launcher. Registry coordinates move from PyPI to
  crates.io + npm (updates the ADR-0003 addendum's PyPI placement).
  Holden ships as a static binary for auditor operators.
- PyO3 bindings are not pursued; they return only if third-party
  Python demand appears.
- The pure-Python conformance reference in `spec/tools/` (ADR-0002)
  is unchanged and becomes more important: it is the independence
  guard keeping an all-Rust first-party stack honest.

## Consequences

- One toolchain, one CI pattern, no wheel matrix, no maturin — the
  binary release matrix is the only packaging pipeline.
- Full cold-start verification in the consumer from day one (no
  bindings gap).
- Embedding/query/crawl-heuristic iteration happens in Rust —
  accepted as slower than Python; mitigated by keeping components
  thin.
- ADR-0002's "Rust everywhere" rejection is reversed for first-party
  components; its bindings-story argument against Go (the decisive
  one) is retroactively weakened, but the fluency and hostile-input
  arguments stand.
