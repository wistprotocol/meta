# ADR-0008: Declared publication

**Status:** accepted · amended by Addendum (2026-10-07: decision 4 taken, the branches integrated) · **Date:** 2026-09-27 · **Amends:** [ADR-0005](0005-capacity-model-and-baseline.md), [ADR-0006](0006-clave-ingestion-architecture.md), [ADR-0007](0007-signed-publications-and-consumer-trust.md)

## Context

Spake derives a site's publications from the site itself: it reads a
sitemap or an RSS feed, fetches every listed page and takes the text of
the whole document. It has no list of permitted paths, reads no
exclusion a page declares, and removes a URL for one build only. A page
assembled in the browser yields its shell. Three objections follow. A
site owner must trust a tool's choice of pages and of content, and a
wrong choice lasts, because a sealed URL never leaves the Log. Every
site would depend on one tool, since platform integrations were planned
to share its publication code. Sites rendered by script from a backend
are not served.

Since [ADR-0007](0007-signed-publications-and-consumer-trust.md) the
signed Payload is the publication, so nothing in the protocol requires
deriving it from a served page. The protocol decisions are spec
ADR-0050 (Emissions and `links`), ADR-0051 (Collections, Scope and
keys) and ADR-0052 (Items and Catalogs). This record fixes the project
consequences.

The Catalog design adopted here leaves out four features whose rules
left defects open: removal proved by absence, a Payload shared by Items
of equal content, completeness as protocol state, and a withdrawal
request signed by the Publisher. Its rules on the validity and
succession of Catalogs are fixed in the conformance reference before
the normative text is written (decision 5).

## Decision

1. **Publication is emitted at the source.** Spake reads Emissions
   (spec ADR-0050) and signs what they state. It reads no page and
   chooses no content. Its sitemap, fetch and extraction modules are
   rebuilt as an Emitter for pages that declare themselves: it emits
   only from pages that carry the marker, and only the delimited
   region. Emitters for content platforms are platform plugins under
   [ADR-0001](0001-repo-layout.md) and get their own repositories.
2. **Collections, Scope and Collection keys** (spec ADR-0051) are
   implemented in core, Spake, Clave and Graven.
3. **Items and Catalogs replace Deltas and Feeds** (spec ADR-0052).
   The conformance reference implements every rule and every open
   point of that decision, each exercised by a discriminating vector,
   before the normative text is restated.
4. **One branch, one release.** spec, core, Spake, Clave, Graven and
   deploy change together, on one branch of each repository, are
   validated together by the three-role end-to-end test with a Labeler,
   and are published together. Whether the branches are integrated into
   the main lines is decided once, after that validation and the
   repeated capacity baseline, on what they measured. Until then the
   published main lines implement the present contract.
5. **Order of work.** The conformance reference and its vectors; the
   normative text; core; Spake, Clave and Graven; integrated validation
   and the capacity baseline repeated; the decision on integration; the
   release. Capacity qualification and the validation of publication
   and query utility follow the release, because both measure the
   publication path.
6. **Left for decisions of their own:** a withdrawal request signed by
   the Publisher; Labels and disputes as Items of a Catalog; a
   Collection served from a host outside the Publisher's authority.

## Amendments

- **[ADR-0005](0005-capacity-model-and-baseline.md).** Its scenarios
  count "one Delta per changed page, and one `attest` per unchanged
  page per attestation interval", and note that "attestation dominates
  every steady state". The attestation term is one Catalog per
  Collection per refresh interval, and the entry, request
  and storage terms are measured again by the baseline. Its unsupported
  path on sitemap index documents is void, since discovery leaves
  Spake.
- **[ADR-0006](0006-clave-ingestion-architecture.md).** Scheduling,
  admission state, budget and walk position are kept per Collection.
  Stage 2 fetches a Catalog and the tree files whose hash the
  aggregator does not hold in place of Feed pages and Deltas,
  stage 3 recomputes the Catalog's root, and stage 4 reads the
  Collection's latest sealed Catalog in place of `url_tips`.
- **[ADR-0007](0007-signed-publications-and-consumer-trust.md).** In
  its order of work, the release of declared publication precedes
  capacity qualification and release composition.

## Alternatives considered

- **Keep the crawl and add a list of permitted paths.** Leaves the
  choice of content to the tool and serves no script-rendered site.
- **Release Emissions and Collections first and Catalogs later.** After
  the first Log is consumed by a third party the signed format changes
  only with a new major version of the suite.
- **The Catalog design with those four features.** Its
  withdrawal request names a Payload commitment, which is bound to no
  Publisher, so a signer could ask for the erasure of content another
  signer published.

## Consequences

- A site owner chooses what is published where the content is stored,
  limits it by Scope, and keeps the recovery key while a platform
  signs.
- The entry path that asked for no change to a site is replaced by one
  that asks for an output template, a hook or a marker.
- The change reaches about half of the Rust code: 51 149 of 92 572
  lines sit in files that depend on Deltas or Feeds.
- A Publisher signs once per update and serves files in proportion to
  its publications, a Log seals one Catalog per Collection where it
  sealed one attestation per URL, and a sealed publication is 1.5 to
  2.9 times the octets of an updated Delta.
  Copying a Collection between Logs then needs the site's files, and a
  Log that starts later holds no earlier state.
- Erasure stays a request to each aggregator's operator.
- Labelers keep the Label Feed, so Feed and Page rules stay in the
  suite for them.
- Graven's query tools return an Item ID where they returned a Delta
  ID.

## Addendum (2026-10-07)

Decision 4 is taken: the branches are integrated into the main lines and
released together, core 0.3.0 and Spake, Clave and Graven 0.2.0 against
specification revision f4acfef. The basis, measured on one machine over
100 sites of 100 pages with 10 % of pages changed per round, the earlier
figure being the Delta path's at the same sizes:

- The pull of an update round completes 0.97 s after the last ping
  against 353.6 s, and a Consumer catches up over twelve Epochs in 6.3 s
  against 42.7 s. A change list saves 38 % of the requests and octets of
  a walk.
- A sealed Item Entry is 913 octets against 484 per Delta, plus 507 per
  Catalog per changed Collection per Epoch; the seal of 10 000 records
  takes 6.80 s against 3.24 s and an empty seal 1.2 to 1.8 s against
  0.18 s.
- The three-role end-to-end test passes with a Labeler, and the audits of
  the sealing, recovery and withdrawal paths leave no confirmed
  divergence open.

The costs that rose are constants at a given scale; the cost that fell
was the dominant term of the earlier path. Carried into capacity
qualification: seal time grows with Collection size (121 s for two
Collections of 5 000 Items), the aggregator's peak memory in an update
round (434 to 673 MB), the empty-seal rise over consecutive Epochs, and
the Publisher's run cost of about 10 ms per listed page.
