# Where the suite stands

Status of 2026-09-26. What has been built of the suite proposed in [suite-design.md](suite-design.md),
what has not, the decisions taken while building it, and what was measured on real libraries. The
other notes in this folder are snapshots of 2026-09-25: they describe the existing tools and the
proposal as they were, with short "since this snapshot" notes where the proposal has moved.

---

## 1. Summary

- **Built:** the data layer and the formal side of the views.
  - The three specifications, with conformance vectors.
  - An annotation package for the attributes the suite reads.
  - The extractor, released for three toolchains.
  - The evidence core.
  - Evidence stores in GitHub repositories, with intake from issues and comments by people and
    AI agents.
  - Two front ends of our own: a Referee-style site, and a single page for one claim with its
    social review. A third is trust's own front end, forked to read our data.
- **Tried on:** LeanMachineLearning (daily), Tau Ceti (the Reviewed-by pilot), and a small test
  project where AI reviewers found a deliberately planted definition error.
- **Not started:** most of what *produces* evidence (analyzers, generators, formal challenges),
  and most of what serves people over time (review workspace, pull-request bot, editor
  integration, dashboard), as well as signing and federation.

In terms of the phases of suite-design.md §10:
- **phase 1** is done except the self-checks;
- **phase 2** is mostly done, but the pull-request bot is not built and one pilot still has its
  own intake;
- **phase 3** has only its first pieces (example and non-example attributes, evidence shown on
  pages);
- **phase 4** is not started.

---

## 2. The repositories

All in the [LeanTrustBuilders](https://github.com/LeanTrustBuilders) organization.

| repository | piece (suite-design.md §4) | what it is |
|---|---|---|
| [specs](https://github.com/LeanTrustBuilders/specs) | the three specifications | S1 declaration key, S2 dataset (`ltb-dataset/0`), S3 evidence records and stores (`ltb-evidence/0`), JSON schemas, conformance vectors checked in CI |
| [annotations](https://github.com/LeanTrustBuilders/annotations) (`TrustAnnotations`) | 1, annotation packages | the core package: one generic environment extension with JSON payloads, and on it `@[claim]`, `@[specifies]`, `@[characterization]` (ported from Characterization), `@[example_of]`, `@[nonexample_of]`. Lean core only |
| [meaning-graph](https://github.com/LeanTrustBuilders/meaning-graph) (`MeaningGraph`) | the dependency engine inside 2 | moved from `RemyDegenne/meaning-graph` with its history. Statement, meaning and term dependencies of every declaration, with the four recoveries of dependency-testing.md §4.3; now fast, and with options for trust's choices (§4 below). Lean core only |
| [extractor](https://github.com/LeanTrustBuilders/extractor) (`trust-extract`) | 2, extractor | a compiled library to an S2 dataset in one pass: nodes, three hashes, edges by notion, and facets (docstrings, source ranges, axioms and `sorry`, statements taken apart with the constant each identifier names, signatures, every annotation). Release 0.6.0 for Lean 4.35.0-rc2, 4.34.0 and 4.34.0-rc2 |
| [evidence-core](https://github.com/LeanTrustBuilders/evidence-core) | 6, evidence core | Python, no dependencies: record validation, statuses against a dataset, threads, coverage under a reader's policy, the review queue, revision diffs, migration from Reviewed-by, Referee and trust, and evidence stores with their append-only check |
| [evidence-store](https://github.com/LeanTrustBuilders/evidence-store) | 7, evidence store | the GitHub side of a store: issue forms, intake from issues and comments, the check on changes, commands for agents, and `init` to set a repository up |
| [referee-site](https://github.com/LeanTrustBuilders/referee-site) (`trust-site`) | 10, views | a static site in Referee's image (claims, claims-only builds, statement anatomy with hovers, graphs, a private audit, changes between builds, provenance); a single page for one claim with its reviews (`trust-site claim`); and the index trust-web reads (`trust-site trust-index`) |
| [trust-web](https://github.com/LeanTrustBuilders/trust-web) | 10, views (the explorer) | a fork of chrisflav/trust-web that reads indexes made from our datasets |
| [site-pilot](https://github.com/LeanTrustBuilders/site-pilot) | pilot | [LeanMachineLearning](https://leantrustbuilders.github.io/site-pilot/), rebuilt daily as the Referee-style site and in trust's front end, plus claims demos of two paper formalizations |
| [reviewed-by-pilot](https://github.com/LeanTrustBuilders/reviewed-by-pilot) | pilot | [Reviewed-by for Tau Ceti](https://leantrustbuilders.github.io/reviewed-by-pilot/), fed by datasets instead of regular expressions, with marks keyed by the meaning hash and kept as an S3 store |
| [review-sandbox](https://github.com/LeanTrustBuilders/review-sandbox) | test project | a small library with one claim, an evidence store with live intake, and [the claim's page](https://leantrustbuilders.github.io/review-sandbox/) |

Nothing in the suite depends on Characterization, ChallengeGen or the original MeaningGraph
repository. Its only Lean dependency outside the organization is semantic_hash, pinned at `0496f6d`.

---

## 3. The pieces, one by one

| # | piece | state | notes |
|---|---|---|---|
| 1 | annotation packages | **partly** | option a of suite-design.md §3.2, as proposed: a new attribute becomes a facet at the next extraction, with no extractor release. Missing: the `value`, `agreement` and `known result` kinds, `@[junk_value]`, domain annotations, `@[landmark]`, a reader for Mathlib's cross-reference tags |
| 2 | extractor | **built** | Tau Ceti at 8befae0 (7,432 modules on Mathlib, 81,999 declarations) in about 80 seconds, split into parts to stay under Linux's memory-mapping limit. Datasets are byte-identical between runs and machines. `--upstream-closure` follows dependencies into the libraries underneath. Missing: incremental extraction per module, and a run on Mathlib itself |
| 3 | analyzers | **not started** | junk values, choice, instances and generality, inhabitation and consistency |
| 4 | standalone files and certification | **not started** | ChallengeGen and Comparator are not integrated; Comparator configs are only read to find claims |
| 5 | self-checks | **not started** | no comparison with the flat printer, no kernel replay. What exists: MeaningGraph tested against its previous implementation and on its options, the extractor tested for determinism and for independence from how it is split, and the hash measurements of §5 |
| 6 | evidence core | **built, in Python** | the proposal named TypeScript, Rust or Lean. Python matched the site builders that consume it, and the logic is small enough to port |
| 7 | evidence store and intake | **built** | stores in repositories, filled from GitHub issues and comments or by pull request. The Tau Ceti pilot's records are in a store, but it still takes input through Reviewed-by's own forms and comment syntax |
| 8 | signing and federation | **not started** | S3 reserves a `signature` field. trust's certificates are keyed by the proof-relevant semantic hash at the same pinned revision, which is our `content` hash, so a certificate can already be matched to a dataset's declaration |
| 9 | evidence generators | **not started** | AI agents do take part as reviewers (§6), but nothing generates examples, disproofs or challenges |
| 10 | views | **partly** | the formal layer of I1, and a first review page (§6). Missing: the math-language layer and conventions panel, a review workspace with a personal queue, the pull-request bot and policy gate, editor integration, an MCP interface, the dashboard, an explorer across libraries |

The self-checks (piece 5) are the gap that matters most for trust in the suite itself. Coverage,
staleness and "rests on" are only as good as the dependency lists, and those are still checked only
by their own tests.

---

## 4. Decisions taken while building

- **One dependency engine, with options** (suite-design.md §9, question 1). MeaningGraph is the
  engine. trust draws its graphs differently: it follows dependencies past the project, counts
  every constant completion offers, and treats proofs as leaves while unfolding definitions whole.
  Those choices are options of MeaningGraph (`Boundary`, `Display`, `Context.closure`), not a
  second computation. On LeanMachineLearning, trust's closure reaches 10,288 upstream
  declarations, and the whole extraction with it takes 19 seconds.
- **Three notions of dependency in datasets:**
  - `statement`: what the statement mentions;
  - `meaning`: the statement, plus the data of a definition's value with proofs skipped;
  - `term`: everything, proofs included.

  The source dependencies of dependency-testing.md §2 (notation, coercion instances) are folded
  into all three rather than given a file of their own.
- **Hover data instead of code shards.** S2 carries statements taken apart and signatures, each
  with the constant every identifier names. This is what trust's code shards were for, and a
  converter writes trust's shards from it.
- **The generic extension's payload is JSON** (question 7), under `annotation/2`, which keeps every
  application of an attribute.
- **Records are never anonymous.** Every S3 record names the GitHub account it came from, or is
  labelled as an AI agent's (`{tool, model, session}`), or both, for an agent acting through an
  account. This replaces suite-design.md §2's three levels of accountability (anonymous, GitHub,
  signed). A reader's private audit stays private, in the browser; it exports under the reader's
  GitHub account. Signing is for later.
- **S3 gained a `comment` kind and statuses.** Discussion and answers are records, and a record's
  state is changed by `status` records:
  - `withdrawn`, by its author;
  - for a problem, `fixed` (with its commit), `intended` or `invalid`, by its reporter or a
    maintainer;
  - for a question, `answered`, by its asker or a maintainer;
  - `reopened`, for problems and questions.

  A review by the same reviewer supersedes their earlier acceptance.
- **Stores live in git repositories:** `evidence/store.json` and JSONL records, append-only. Only
  two writers are allowed: an intake bot, writing each record under the account whose issue or
  comment it read, and pull requests, whose records must be by their author. A store can sit in
  the library's own repository or elsewhere.
- **Static first** (question 6). Every view is a static site. Changes go through GitHub issues,
  prefilled by the page, rather than through a service.
- **Releases per toolchain:** `main` follows the newest Lean, branches `lean-v<toolchain>` carry
  the same code for older ones, and extractor releases are tagged `v<version>-lean-v<toolchain>`.

---

## 5. What was measured

- **Hash stability, on Tau Ceti** (S1).
  - Across 428 commits in 29 hours, 92.4% of reviews would have stayed current, 1.6% gone stale
    underneath and 0.5% stale.
  - Across a Lean and Mathlib bump, 26% would have gone stale underneath, almost all of them
    because of 74 rewritten upstream declarations.
  - The local hash sees elaboration details. A declaration whose text did not change can read as
    stale when a constant it uses changed its signature. S1 records this as a known limit, with a
    candidate fix.
- **The content hash depends on how an extraction is split.** Lean generates some auxiliary lemmas
  in several modules, and which copy a split sees varies: 287 of 97,944 content hashes differed
  between 4 and 8 parts. The meaning and local hashes, which key records, did not. Datasets record
  the number of parts.
- **MeaningGraph's performance.** Its notation recovery was exponential on some terms and never
  finished on one quarter of Tau Ceti. Rewritten, that quarter takes 0.3 seconds for the tables and
  0.6 seconds for 20,000 declarations' dependencies, with identical results.
- **Hover data costs about two thirds more dataset** on LeanMachineLearning: 3.9 MB becomes 6.4 MB.
  Each part can be left out.

---

## 6. Social review, tried out

The review sandbox is a small library whose claim is Euclid's theorem, stated with three
definitions of its own. The first version of `IsPrime` admitted 1, on purpose.

**The page.** The claim's page shows:
- the claim and what it rests on, as a graph and one card per declaration;
- for each declaration, what Lean checks about it and its review threads, with who reviewed, what
  they compared it with, which failure modes they checked, their caveats, the replies and the
  status changes;
- coverage under the reader's policy: whether reviews by AI agents count, reviews made before
  something underneath changed, acceptances with caveats, authors' own reviews;
- what to review next, including the failure modes nobody has checked yet.

Every "Review", "Report a problem", "Ask a question", "Withdraw" or "Mark fixed" opens the store's
issue form, prefilled.

**The test.** Two AI reviewers ran independently, on different models, and submitted from a
terminal.
- One found the planted error (1 counted as prime), confirmed it by compiling a counterexample, and
  reported it as an edge-case problem.
- The fix was committed and the problem marked fixed. The earlier reviews then showed as made on
  an earlier version, and the claim's own review as changed underneath `IsPrime`.
- The other reviewer asked a question, which the first answered and the asker closed.
- A person then submitted a review through the web form, which intake recorded under their account.

**Two problems found live, both fixed:**
- concurrent intake runs could conflict on the store;
- a review made at a commit whose dataset was not built yet was reported as the reviewer's
  mistake, rather than simply read again later.

---

## 7. What comes next

In the order that seems most useful:

1. **Self-checks** (piece 5).
   - Compare MeaningGraph's closures with the flat printer's on the three corpora of
     dependency-testing.md.
   - Replay claimed closures through the kernel in the extractor's CI.
2. **Move the Tau Ceti pilot's intake to evidence-store.** Keep its write-back into docstrings as
   one more view.
3. **An evidence store for LeanMachineLearning,** with a claim page per claim.
4. **The pull-request bot** (I4): what a change made stale, from two datasets and a store.
5. **The review workspace** (I2): a queue and progress that belong to the reader, over the shared
   store.
6. **The missing `@[specifies]` kinds,** and the missing-examples report
   (trusting-definitions.md §6, recommendations 1 and 2).
7. **Formal challenges** through ChallengeGen and Comparator, stored as S3 records.

Still open from suite-design.md §9:
- **hash migrations** (records carry the hasher revision and read as `incomparable`, but nothing
  migrates them);
- **extraction at Mathlib scale;**
- **write policy** beyond "authors and maintainers";
- **the math-language layer;**
- **Mathlib's cross-reference attributes.**
