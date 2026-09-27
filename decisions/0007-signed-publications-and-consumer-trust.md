# ADR-0007: Signed publications and consumer-side trust

**Status:** accepted, amended by [ADR-0008](0008-declared-publication.md) (2026-09-27: the release of declared publication precedes capacity qualification and release composition) · **Date:** 2026-09-16 · **Amends:** [ADR-0001](0001-repo-layout.md), [ADR-0003](0003-naming-policy.md), [ADR-0004](0004-all-rust-stack.md), [ADR-0005](0005-capacity-model-and-baseline.md), [ADR-0006](0006-clave-ingestion-architecture.md)

## Context

The suite verified a Publisher's signed description of a page by having
independent Auditors fetch the page and measure it (WIST-4 §§3–7). That
layer is the largest part of the specification and of core, is unbuilt
as a service (Holden), and its guarantee is narrow: it establishes that
the page carries the text the domain signed, never whether the content
is worth anything. A domain that signs junk and serves junk passes every
audit; reputation measures process, not content (WIST-4 §6.4). Its
false-positive class is real: an honest page rewrite fetched before its
correction seals reads as fabricated content, and one such finding can
quarantine a domain (WIST-4 §§5–7). Systems that index
signed content at scale without crawling — AT Protocol relays over
signed repositories, Nostr relays over signed events, email with
in-protocol authentication and out-of-protocol reputation lists — show
that a signed publication plus a replicated log is enough to build an
index, and that quality is a consumer and labeler matter. Spec
[ADR-0041](../../spec/decisions/0041-signed-publications.md) records the
protocol decision; this record fixes the project consequences.

## Decision

1. **The signed Payload is the publication.** The index carries what a
   domain signed; correspondence with the served page is not measured,
   not sanctioned and not a project goal. Consumers may present the
   signed publication directly.
2. **The Auditor role is retired.** Holden is not built; the `holden/`
   repository is never created; the codename is retired. Every task
   blocked on Holden is closed, and the four-role integration becomes a
   three-role integration: Spake, Clave, Graven.
3. **Trust is assessed at consumption.** Graven ranks with pluggable
   ranking profiles: a documented default that combines text relevance,
   trust propagated from seed domains along the signed link graph,
   distrust seeds, publication age from the Log and chain freshness;
   alternatives selectable per query or per install; profiles are plain
   files anyone can publish. The protocol transports the inputs only.
4. **Labelers are publishers.** A labeler is a domain with a key whose
   publications are signed labels about other domains' pages or domains,
   carried through the same feeds, aggregators and Log. Consumers
   subscribe to labelers explicitly; trust and distrust seed lists are
   labeler publications. Spake gains labeler publishing; Clave and
   Graven carry labels like any other publication.
5. **Order of work.** Spec revision first (WIST-1..4, schemas, vectors,
   reference tools), then removal of the audit machinery from core,
   Clave and Graven, then consumer ranking and labels, then the
   operational tasks already queued: durable ingestion, incremental
   Snapshots, capacity qualification, the expansion decision, release
   composition. The capacity model and ingestion stages lose their audit
   terms and keep everything else.

## Alternatives considered

- **Keep auditors and add a grace window** (settled references, incident
  identity per reference, link containment, small-site gate,
  retractions as scrutiny): removes the false-positive class but keeps
  the whole audit economy, Holden and evidence retention for a
  guarantee consumers rarely need.
- **Aggregator-side checks at ingest and random re-checks with exclusion
  instead of sanctions**: cheaper than auditors, still fetches pages,
  still cannot judge content, and makes one party's fetch the trust
  signal.
- **Reader-side or witness attestations, signed exchanges, immutable
  page histories**: stronger evidence at costs no participant bears
  today; signed exchanges are being withdrawn by their own vendors.

## Consequences

- WIST-4 shrinks to governance and parameters; Audit Records, roster
  and Observer acts, canaries, sampling, verdicts, confirmation,
  reputation, sanctions, notices, appeals and evidence retention leave
  the suite. Quota and inclusion latency become flat.
- core drops its audit modules; Clave drops audit ingestion, derived
  governance state, appeals and submissions of roster acts; Graven
  drops adopted audit state, the act replay and the sanction ledger.
  The end-to-end test and the capacity baseline lose their audit terms.
- Graven gains a link index, trust propagation, ranking profiles and
  label subscriptions; Spake gains labeler publishing.
- Funding assumptions about independent Auditor operators are void;
  independent labelers take that place.
- The guarantee the index offers changes from "verified to be served" to
  "signed by the domain, ranked by a policy you choose". Documentation
  states it that way.
