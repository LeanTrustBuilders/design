# Where the suite stands

Status of 2026-09-29. Since 2026-09-28:
- **Stores import each other's records** (S3 "Imported records"). This is the first part of
  federation: it is static, it works at build time, and nothing is signed. A store names other
  stores in its `store.json`, and its pages show their records about declarations it has. Two uses:
  - a library shows reviews of the Mathlib declarations it rests on;
  - a store about Mathlib gathers what libraries built on Mathlib say about its declarations.

  The shared-evidence demo about Mathlib's probability theory and the two pilots' stores now import
  each other, and LeanMachineLearning has a store of its own.
- **Reviews name a rubric** (`ltb-evidence/2`, reviews.md §7, question 4).
- **The Tau Ceti pilot follows Tau Ceti again**, on Lean 4.35.0-rc3.
- **Packages are labelled by their Lake name** (extractor 0.13.0). A Mathlib declaration now has
  the same key, package included, in every dataset made on the same Mathlib commit (§5).

Since 2026-09-27: every hash is the suite's own. The content hash is
MeaningGraph's (`ltb-content/1`, the meaning hash's walk with proofs kept), semantic_hash is no
longer a dependency, and datasets (`ltb-dataset/2`) no longer carry the hashes of version 0.

Since the status of 2026-09-26: the first part of
well-definedness. Authors, and a catalogue for Mathlib, now declare what a definition is meant to
be, and pages show it. The first analyzer checks each use of a definition with a declared domain
in a statement.

Before that: the self-checks, the rule `ltb-meaning/1` and its release, the Tau Ceti pilot on the
suite, and the front ends made thin over the tools.

This note says what has been built of the suite proposed in [suite-design.md](suite-design.md), what
has not, the decisions taken while building it, and what was measured on real libraries. The other
notes in this folder are snapshots of 2026-09-25: they describe the existing tools and the proposal
as they were, with short "since this snapshot" notes where the proposal has moved. The exception is
[well-definedness.md](well-definedness.md), of 2026-09-27.

---

## 1. Summary

- **Built:** the data layer and the formal side of the views.
  - The three specifications, with conformance vectors.
  - An annotation package for the attributes the suite reads.
  - The extractor, released for three toolchains.
  - One rule for what a declaration's meaning rests on (`ltb-meaning/1`): the dependency graph and
    the hashes that decide staleness come from the same walk, so they agree by construction.
  - The first two self-checks: graph against hash, and Lean's kernel checking every dependency
    closure. On Tau Ceti, all 95,688 closures check.
  - The evidence core.
  - Evidence stores in GitHub repositories, with intake from issues and comments by people and
    AI agents. A store can import other stores' records, fetched when its pages are built.
  - Two front ends of our own: a Referee-style site, and a single page for one claim with its
    social review. A third is trust's own front end, forked to read our data.
  - What a definition is meant to be, declared with an attribute on the definition itself:
    - where it is meant to apply (`@[domain]`);
    - what it is determined up to (`@[up_to]`);
    - what characterizes it (`@[characterization]` on one theorem, with no predicate to add).
  - These shown on a definition's page, and its graph through any characterization the reader
    picks, instead of the construction.
  - A catalogue declaring these for Mathlib's definitions from outside Mathlib, merged into a
    Mathlib site.
- **Tried on:**
  - LeanMachineLearning (daily, with a store of its own);
  - Tau Ceti (the Reviewed-by pilot: Reviewed-by's page, with everything behind it on the suite);
  - Mathlib itself:
    - the Mathlib Explorer;
    - a daily site for its probability theory, with the catalogue's declarations;
    - a store of reviews of its probability theory, with three front ends. The two pilots' stores
      import it, and it imports them;
  - alpha-rar, a paper formalization whose definitions carry domains and characterizations;
  - a small test project, where AI reviewers found a deliberately planted definition error.
- **Not started:**
  - most of what *produces* evidence (analyzers, generators, formal challenges). One analyzer
    exists: uses of definitions against their declared domains, in statements;
  - most of what serves people over time (review workspace, pull-request bot, editor
    integration, dashboard);
  - signing, and federation beyond importing stores at build time.

In terms of the phases of suite-design.md §10:
- **phase 1** is done, with the first two self-checks (graph against hash, and the kernel check);
- **phase 2** is mostly done, but the pull-request bot is not built;
- **phase 3** has only its first pieces: the example, non-example, domain, up-to and
  characterization attributes, the Mathlib catalogue, and evidence shown on pages;
- **phase 4** has only the static part of federation: stores import each other's records.

---

## 2. The repositories

All in the [LeanTrustBuilders](https://github.com/LeanTrustBuilders) organization.

| repository | piece (suite-design.md §4) | what it is |
|---|---|---|
| [specs](https://github.com/LeanTrustBuilders/specs) | the three specifications | S1 declaration key (version 2), S2 dataset (`ltb-dataset/2`), S3 evidence records and stores (`ltb-evidence/2` since 2026-09-29: records name the rubric their checklist and categories are axes of, `ltb-rubric/1` suggested; `ltb-evidence/1` of 2026-09-28 gave the compatibility rules, per-kind fields and one `text` field; the stores were rewritten to each once; since 2026-09-29 also "Imported records": a store's `imports` and what another store's records mean here, §4), JSON schemas, conformance vectors checked in CI against the schemas and evidence-core |
| [annotations](https://github.com/LeanTrustBuilders/annotations) (`TrustAnnotations`) | 1, annotation packages | the core package: one generic environment extension with JSON payloads, and on it `@[claim]`, `@[specifies]`, `@[characterization]` (ported from Characterization; since 2026-09-27 also on a single theorem with no predicate, including a type characterized up to isomorphism), `@[example_of]`, `@[nonexample_of]`, `@[domain]` and `@[up_to]` (2026-09-27, well-definedness.md). Each checks what it can: a characterization's existence is proved from the definition's `@[specifies]` lemmas, a domain and a relation are elaborated against the definition's arguments. Lean core only |
| [meaning-graph](https://github.com/LeanTrustBuilders/meaning-graph) (`MeaningGraph`) | the dependency engine inside 2 | moved from `RemyDegenne/meaning-graph` with its history. Statement, meaning, term and source dependencies of every declaration, with the four recoveries of dependency-testing.md §4.3; now fast, and with options for trust's choices (§4 below). `MeaningGraph.Hash` draws the `meaning` graph and computes the meaning and local hashes in one walk (meaning-hash.md), and the content hash in a second walk that keeps proofs. Lean core only |
| [extractor](https://github.com/LeanTrustBuilders/extractor) (`trust-extract`) | 2, extractor | a compiled library to an S2 dataset in one pass: nodes, three hashes, edges by notion, and facets (docstrings, source ranges, axioms and `sorry`, statements taken apart with the constant each identifier names, signatures, every annotation); and `trust-extract check`, the kernel check of a dataset's closures. Version 0.13.0 for Lean 4.35.0-rc2 and 4.35.0-rc3 (`ltb-dataset/2`; since 0.13.0 packages are named by Lake's manifest); 0.7.3 for 4.34.0 and 4.34.0-rc2 (`ltb-dataset/1`, with semantic_hash's content hash). The GitHub action `extract` (newest release for the library's toolchain, extraction, examples, publication as a release) is what every pilot's workflow uses |
| [evidence-core](https://github.com/LeanTrustBuilders/evidence-core) | 6, evidence core | Python, no dependencies (0.14.0): record validation, statuses against a dataset, threads, tests and challenges (whether a test passes at the dataset's commit), what pages show (record views, where each declaration stands under every policy, claims, changes between datasets, the provenance ledger, source text, the dataset's analyses, what pins a definition down: specifications, characterizations, examples, domains and up-to relations, each with its source: author, reviewer or catalogue), coverage under a reader's policy, the review queue, revision diffs, migration from Reviewed-by, Referee and trust, evidence stores with their append-only check and the stores they import, the self-checks over datasets (`check-graph`, `compare-rules`), and `merge`, which adds a catalogue's dataset to its library's after checking that they agree on every shared meaning hash |
| [evidence-store](https://github.com/LeanTrustBuilders/evidence-store) | 7, evidence store | the GitHub side of a store (0.6.0): issue forms (review, problem, question, proposed test, test, name, status), intake from issues, comments, closes and reopens by hand, and a bulk issue, the check on changes, commands for agents, fetching datasets from their releases (`dataset`), adding a file of records (`add`), fetching the stores a store imports at their current commits (`fetch-imports`), and `init` to set a repository up |
| [referee-site](https://github.com/LeanTrustBuilders/referee-site) (`trust-site`) | 10, views | (0.8.0) lays out what evidence-core computes: a static site in Referee's image (claims, claims-only builds, statement anatomy with hovers, graphs, a private audit and the community's reviews, changes between builds, provenance; on a definition's page, where it is meant to apply, what it is determined up to and what pins it down; the graph through characterizations the reader picks), of a whole library or of a slice of one (`--modules`); a single page for one claim with its reviews (`trust-site claim`); and the index trust-web reads (`trust-site trust-index`). Each can include the records of imported stores (`--imports`), marked with the store they come from |
| [trust-web](https://github.com/LeanTrustBuilders/trust-web) | 10, views (the explorer) | a fork of chrisflav/trust-web that reads indexes made from our datasets |
| [site-pilot](https://github.com/LeanTrustBuilders/site-pilot) | pilot | [LeanMachineLearning](https://leantrustbuilders.github.io/site-pilot/), rebuilt daily as the Referee-style site and in trust's front end, plus claims demos of two paper formalizations, rebuilt at every run from their datasets, and [Mathlib's probability theory](https://leantrustbuilders.github.io/site-pilot/mathlib-probability/) (a slice of 6,720 declarations), from Mathlib Explorer's dataset merged with the catalogue's. Since 2026-09-29, LeanMachineLearning's evidence store (`evidence/`, with issue forms and intake), which imports the probability store |
| [well-defined](https://github.com/LeanTrustBuilders/well-defined) (`WellDefined`) | 3, analyzers | the well-definedness analyzer: each use of a definition with a declared domain in a statement, and whether what is in scope shows its arguments to be in the domain (discharged, irrelevant, refuted, open, unapplied), with dischargers named as tactics. Lean core and TrustAnnotations. The extractor runs it (`trust-extract welldefined`, 0.8.0) |
| [mathlib-catalogue](https://github.com/LeanTrustBuilders/mathlib-catalogue) | 1, a catalogue (well-definedness.md §6) | what Mathlib's definitions are meant to be, declared from outside Mathlib: the domains of the Bochner integral, conditional expectation and the Radon–Nikodym derivative; the last two determined up to a.e. equality; characterizations of the real integral, of those two and of `ℝ` up to isomorphism. Mathlib's theorems cannot carry an attribute written elsewhere, so the catalogue restates them, each proved by the one it restates. CI publishes a small dataset per commit |
| [reviewed-by-pilot](https://github.com/LeanTrustBuilders/reviewed-by-pilot) | pilot | [Reviewed-by for Tau Ceti](https://leantrustbuilders.github.io/reviewed-by-pilot/): Reviewed-by's page as it was, with every tool behind it replaced by the suite (datasets, evidence-store's forms and intake, an S3 store, evidence-core); proposed tests are S3 challenges, the roadmaps' and Voyager's named results S3 records by agents. Follows Tau Ceti's main (163ce80, Lean 4.35.0-rc3, 106,734 declarations); imports the probability store, whose reviews of Mathlib declarations Tau Ceti rests on appear on the page |
| [mathlib-explorer](https://github.com/LeanTrustBuilders/mathlib-explorer) | 10, views (a front end for readers) | [Mathlib Explorer](https://leantrustbuilders.github.io/mathlib-explorer/): Mathlib for readers who know mathematics but not Lean: search in words, subjects, and a page per concept and theorem with the concept map (what it is built from) and the proof map (what its proof uses); the famous theorems, the undergraduate curriculum, the bibliography, a map of the subjects. Laid out from evidence-core's `docs`, `catalogs` and `graphs` |
| [mathlib-probability-evidence](https://github.com/LeanTrustBuilders/mathlib-probability-evidence) | shared-evidence demo | [one store about Mathlib's probability theory](https://leantrustbuilders.github.io/mathlib-probability-evidence/), keyed by Mathlib Explorer's datasets, with three front ends built from it (the Referee-style site, a claim page, trust's front end). It imports the two pilots' stores and they import it: their reviews of Mathlib declarations appear here, and its reviews appear on their pages. Reviewed-by's page, which it showed until 2026-09-29, is now Tau Ceti's own, importing this store |
| [review-sandbox](https://github.com/LeanTrustBuilders/review-sandbox) | test project | a small library with one claim, an evidence store with live intake, and [the claim's page](https://leantrustbuilders.github.io/review-sandbox/) |

Nothing in the suite depends on Characterization, ChallengeGen or the original MeaningGraph
repository, and since 2026-09-28 (extractor 0.9.0) not on semantic_hash either: it has no Lean
dependency outside the organization.

---

## 3. The pieces, one by one

| # | piece | state | notes |
|---|---|---|---|
| 1 | annotation packages | **partly** | option a of suite-design.md §3.2, as proposed: a new attribute becomes a facet at the next extraction, with no extractor release. Domains are declared with `@[domain]` (well-definedness.md replaces `@[junk_value]`), relations with `@[up_to]`, and characterizations on a single theorem. A catalogue declares them for a library it cannot edit. Missing: the `value`, `agreement` and `known result` kinds, `@[noncanonical]`, `@[landmark]`, a reader for Mathlib's cross-reference tags |
| 2 | extractor | **built** | Tau Ceti at 8befae0 (7,432 modules on Mathlib; 95,688 declarations under the rule `ltb-meaning/1`, 81,999 before it counted private ones) in about 80 seconds with 0.6.0; with 0.7.2, whose rule walks each declaration's meaning down to Lean core (21 to 28 seconds per part), 2 minutes 34 seconds without the statement and signature facets. The work is split into parts to stay under Linux's memory-mapping limit. Datasets are byte-identical between runs and machines. `--upstream-closure` follows dependencies into the libraries underneath. Mathlib itself (v4.35.0-rc2: 314,129 declarations, 6.0 million meaning and 12.1 million proof edges, 954 MB with statements and signatures) in 6 minutes on 32 cores. A script adds the attributes read from the sources: `attributes.py` (`@[stacks]`, `@[wikidata]`, `@[deprecated]`, …). Missing: incremental extraction per module |
| 3 | analyzers | **started** | [well-defined](https://github.com/LeanTrustBuilders/well-defined): each use of a definition with a declared domain in a statement, checked against what is in scope (well-definedness.md §2.5), as the facet `welldefined/1` (`trust-extract welldefined`). Run on Mathlib's probability theory with the catalogue's domains and discharger. Not started: definitions' bodies, choice, instances and generality, inhabitation and consistency |
| 4 | standalone files and certification | **not started** | ChallengeGen and Comparator are not integrated; Comparator configs are only read to find claims |
| 5 | self-checks | **partly** | check 1, graph against hash (`evidence-core check-graph`), which led to the rule `ltb-meaning/1` (meaning-hash.md) and is now an invariant; check 2, the kernel checks each dataset closure (`trust-extract check`), along `meaning` on libraries of any size, along `term` (every proof) only on small ones; a comparison of rules (`evidence-core compare-rules`). The extractor's action runs the kernel check before publishing a dataset (LeanMachineLearning, the sandbox, the demos: every closure passes, along both notions), and referee-site shows the result on declarations, claim pages and the site. Not yet: checks 3 (extra dependencies, beyond dropping edges one at a time) and 4 (statement fidelity), the comparison with the flat printer, graph against hash in the workflows, and the kernel check for Tau Ceti |
| 6 | evidence core | **built, in Python** | the proposal named TypeScript, Rust or Lean. Python matched the site builders that consume it, and the logic is small enough to port |
| 7 | evidence store and intake | **built** | stores in repositories, filled from GitHub issues and comments (commands, closing and reopening by hand, a bulk issue taking one record per line in Reviewed-by's syntax) or by pull request. Every pilot and demo with a store takes all its input through it: the review sandbox, Tau Ceti, LeanMachineLearning and Mathlib's probability theory |
| 8 | signing and federation | **started** | federation's static part (2026-09-29): a store lists the stores it imports in `store.json`, `evidence-store fetch-imports` fetches them at their current commit when pages are built, and the pages show their records about declarations here, marked with the store they come from (rules in §4). Not built: signing, and servers that exchange records. S3 reserves a `signature` field. trust's certificates are keyed by semantic_hash's proof-relevant hash, which our datasets no longer carry: trust-web's index carries our content hash (`ltb-content/1`) instead |
| 9 | evidence generators | **not started** | AI agents do take part as reviewers (§6), and anyone can propose a test (an S3 challenge), but nothing generates examples, disproofs or challenges |
| 10 | views | **partly** | the formal layer of I1, a first review page (§6), and what a definition is meant to be on its page, with its graph through a characterization. Missing: the math-language layer and conventions panel, a review workspace with a personal queue, the pull-request bot and policy gate, editor integration, an MCP interface, the dashboard, an explorer across libraries |

The self-checks (piece 5) were the gap that mattered most for trust in the suite itself: coverage,
staleness and "rests on" are only as good as the dependency lists. The first check found that the
graph and the meaning hash disagreed (339 Tau Ceti declarations), because they erased different
proofs. The hash is now derived from the graph's own rule, and the kernel confirms that every
closure is enough to check its declaration: see dependency-testing.md §9 and
[meaning-hash.md](meaning-hash.md).

---

## 4. Decisions taken while building

- **One dependency engine, with options** (suite-design.md §9, question 1). MeaningGraph is the
  engine. trust draws its graphs differently: it follows dependencies past the project, counts
  every constant completion offers, and treats proofs as leaves while unfolding definitions whole.
  Those choices are options of MeaningGraph (`Boundary`, `Display`, `Context.closure`), not a
  second computation. On LeanMachineLearning, trust's closure reaches 10,288 upstream
  declarations, and the whole extraction with it takes 19 seconds.
- **One rule for the graph and the hashes** (`ltb-meaning/1`, meaning-hash.md §3): proofs erased
  everywhere, private declarations are declarations, helpers looked through, and a Merkle meaning
  hash computed in the walk that draws the graph. The meaning hash therefore changes exactly when
  something in the `meaning` closure does. The content hash is the same walk with proofs kept
  (`ltb-content/1`, 2026-09-28): semantic_hash is not used any more, and trust's certificates,
  keyed by its hash, no longer match ours.
- **Four notions of dependency in datasets:**
  - `statement`: what the statement mentions, proofs erased;
  - `meaning`: the rule's graph: the statement, plus a definition's value, proofs erased;
  - `term`: everything, proofs included;
  - `source`: the notation and coercion instances a declaration's source relies on (dependency-testing.md
    §2). Before `ltb-dataset/1` they were folded into the other three.
- **No backward compatibility for the hashes of version 0** (2026-09-28). Datasets carried each
  node's hashes of before the rule (`legacy`) from 2026-09-26 to 2026-09-28, so that older records
  could be re-keyed. `ltb-dataset/2` drops them: records keyed by semantic_hash's hashes (most of
  the Tau Ceti pilot's, and the review sandbox's) now read as incomparable.
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
- **S3 gained `challenge`, a proposed test** (from Reviewed-by's suggested tests): a property a
  declaration should have, in words and optionally as a Lean statement, with what it would catch.
  It is open until a declaration of the library proves it (`met`, naming the declaration, which
  pages then check as a test: there at the dataset's commit and without `sorry`), or `failed`,
  `declined`, `withdrawn`, `reopened`. A `test` record lists a declaration already in the library
  as a test of another; `named` records say which declarations are the named results.
- **Examples are not tests** (2026-09-28). Reviewed-by counted the `example`s whose statement names a
  declaration as its unit tests, and the suite had carried that over (an `examples` facet, pins "in
  the code"). An example that mentions a declaration need not be about it (`example : Finite (∅ :
  Set α)` is no test of `Set`), and counting them made definitions look pinned down. The facet, its
  script and its pins are gone; tests are declarations listed as tests, and proposed tests met.
- **Stores live in git repositories:** `evidence/store.json` and JSONL records, append-only. Only
  two writers are allowed: an intake bot, writing each record under the account whose issue or
  comment it read, and pull requests, whose records must be by their author. A store can sit in
  the library's own repository or elsewhere.
- **Stores import each other, statically** (2026-09-29, S3 "Imported records"). This is the part of
  federation that needs neither servers nor signatures. Records need no translation, because keys
  agree across datasets (§5) and ids are content hashes. What the rules settle is what a record
  from another store means here:
  - **the store's maintainers choose what to import,** in `store.json`, so that the site, the claim
    page and the trust index show the same records. Pages record the commit each imported store
    was read at, beside the dataset's;
  - **relevance:** an imported record about a declaration this dataset does not have is left out,
    with its thread, instead of showing as orphaned. So are records keyed by another hasher;
  - **who may change a state:** a status on an imported record counts only if it comes from the
    record's own store or from the record's author. A maintainer here cannot mark a problem
    reported elsewhere as fixed; a reply from here is only a comment;
  - **one level:** a store's own imports are not followed, so what a page shows is what its
    maintainers listed;
  - **the reader decides whether imported reviews count,** with a fifth policy switch (on by
    default), which makes 32 precomputed policies;
  - **nothing is written into an imported record.** Its store is kept beside it in memory, and
    pages show it ("from mathlib-probability-evidence"), with no action buttons: acting on it
    happens in its own store.

  Imports may be mutual: the probability store imports the two pilots' stores, and they import it.
  Because imports are one level deep, this cannot loop.
- **Front ends compute nothing.** The front ends are provisional (none is final): the Referee-style
  site and claim page, the Reviewed-by page, trust-web. Whatever produces or processes evidence or
  datasets is in the tools, and a front end only lays out what they computed: record views, statuses,
  where a declaration stands under each policy, claims, changes, provenance, source text and the
  dataset's analyses come from evidence-core; forms and dataset fetching from evidence-store;
  extraction, in CI too, from the extractor's action. Moving them there fixed three bugs the copies
  had grown (withdrawn reviews counted on the site, no trust marks at all in trust-web's index, broken
  links for records read from a bulk comment). Two things stay in pages by necessity: the reader's
  private audit in the browser (which records verdicts, keyed by what evidence-core wrote, and leaves
  record ids to `evidence-store add`), and apply.py's reading of a fork's sources to write lines
  into them.
- **Static first** (question 6). Every view is a static site. Changes go through GitHub issues,
  prefilled by the page, rather than through a service.
- **Intent is declared on the definition, with no predicate to add.** Earlier characterizations
  and junk-value annotations needed a predicate, which cluttered the library with definitions
  nobody uses. Now:
  - `@[domain P]` and `@[up_to R]` go on the definition, and are stored as hidden declarations
    (`d._domain`, `d._upTo`) that tools can use;
  - a characterization is one theorem, with the candidate a bound variable (`m = double n ↔ …`, or
    `P g → g =ᵐ[μ] condExp …`). A predicate can still be tagged.
- **Where a characterization holds is recorded whole.** Every assumption of the theorem that is
  not about the candidate is listed, with no judgment of what matters: a missing one would make it
  look more general than it is. So are the arguments it fixes rather than quantifies over, shown
  as "only for" (`G := ℝ`).
- **Existence is proved, not claimed.** That the definition has the characterizing property is
  proved by a small search from its `@[specifies]` lemmas. A premise it needs about the theorem's
  own variables is assumed and shown ("it has the property when …"), rather than silenced.
- **Catalogues for libraries one cannot edit.** A separate package declares intent for another
  library's definitions. Its small dataset is merged into the library's (`evidence-core merge`),
  which checks that both describe the same library, so a catalogue changes without re-extracting
  Mathlib.
- **A characterization replaces a construction only when the reader asks, one definition at a
  time.** A characterization can hold on a smaller domain than a use: the real integral's does
  not cover the vector-valued integrals inside `condExp`. Substituting every characterized
  definition would claim more than is known.
- **A characterization pins what its statement names.** An isomorphism `K ≃+*o ℝ` names `+`, `*`,
  `≤`; `ℝ`'s other operations are Mathlib definitions of their own, so the catalogue's statement
  names them too (well-definedness.md §4.5).
- **Slices of large libraries.** A site of all of Mathlib is too large for GitHub Pages, so a site
  can be built for some modules and what they rest on (`--modules`).
- **Releases per toolchain:** `main` follows the newest Lean, branches `lean-v<toolchain>` carry
  the same code for older ones, and extractor releases are tagged `v<version>-lean-v<toolchain>`.
  One exception since 2026-09-29: Tau Ceti's main went from Lean 4.34.0 to 4.35.0-rc3, so
  `lean-v4.35.0-rc3` branches (MeaningGraph, TrustAnnotations, WellDefined, the extractor) carry
  a newer toolchain than `main`. Moving `main` to rc3 would also move the catalogue and
  LeanMachineLearning, which are on rc2.

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
- **Keys across libraries** (2026-09-29): what federation relies on. A Mathlib declaration was
  compared across the datasets of different libraries built on the same Mathlib commit (`0653561`),
  each extracted separately with extractor 0.12.1.
  - LeanMachineLearning and the catalogue share 3,785 declarations; Mathlib's own dataset shares
    5,285 with LeanMachineLearning and 3,681 with the catalogue. Every meaning, local and content
    hash agreed, over Lean core declarations, 1,044 instances and 368 classes. So a review keyed
    in one library's store resolves as current in any other.
  - Across 136 Mathlib commits (LeanMachineLearning before and after a bump, 5,285 shared
    declarations), 99.5% of meaning hashes and 99.8% of local hashes were unchanged, and 78.9% of
    content hashes. Each of the 24 meaning changes had a cause: 5 declarations were rewritten, and
    19 had something rewritten underneath.
  - Under extractor 0.7.3 (`ltb-local/2`, still used on the Lean 4.34 branches), two libraries on
    one Mathlib commit agreed on every meaning and content hash, but disagreed on 56 of 396 local
    hashes, mostly of classes and instances: the bug that `ltb-local/3` fixed.
  - The key's `package` differed: `Mathlib` in Mathlib's own dataset and `mathlib` downstream,
    because the project was labelled by its root module. Since extractor 0.13.0 every package is
    labelled by its Lake name, symlinked dependencies included.
- **The content hash depends on how an extraction is split.** Lean generates some auxiliary lemmas
  in several modules, and which copy a split sees varies: 287 of 97,944 content hashes differed
  between 4 and 8 parts. The meaning and local hashes, which key records, did not. Datasets record
  the number of parts.
- **MeaningGraph's performance.** Its notation recovery was exponential on some terms and never
  finished on one quarter of Tau Ceti. Rewritten, that quarter takes 0.3 seconds for the tables and
  0.6 seconds for 20,000 declarations' dependencies, with identical results.
- **Hover data costs about two thirds more dataset** on LeanMachineLearning: 3.9 MB becomes 6.4 MB.
  Each part can be left out.
- **The self-checks** (dependency-testing.md §9, meaning-hash.md §5).
  - Graph against semantic_hash, on Tau Ceti across a bump: 4 declarations stale underneath with
    nothing changed in their closure, 246 current although their closure changed. Traced to the two
    erasing different proofs.
  - Under the rule `ltb-meaning/1`: nothing, on LeanMachineLearning across a Mathlib bump; and where
    semantic_hash marked 67 declarations stale underneath because Mathlib changed proofs inside
    `Prop`-valued instances, no meaning hash moved.
  - The kernel check: every closure of LeanMachineLearning is enough for Lean's kernel, under both
    rules and both notions; dropping edges, it caught every removal the kernel needed. On Tau Ceti
    (95,688 declarations), every `meaning` closure checks.
  - The rule's graph against the old one: closures 7% smaller on average on LeanMachineLearning, 23%
    on Tau Ceti, which also gains 13,689 declarations (its private ones);
    the edges that went away were to proofs (instances of `Prop` classes, lemmas inside statements,
    proofs inside helpers), to projections and constructors (now their types), and to notation and
    coercions (now `source`).
  - Tau Ceti found three bugs that the smaller libraries had not: the rule's hash did not memoise
    compound terms, and never finished on terms that are trees of 10⁸ nodes over a few hundred
    distinct ones (fixed in 0.7.1, no hash changed); and the kernel check left proofs inside
    inductive types unerased and copied every proof of the environment along `term` (fixed in
    0.7.2). Along `term`, the check's memory still grows until the machine runs out: `term` is kept
    for small libraries (dependency-testing.md §9).
- **Sites of Mathlib.**
  - The whole of Mathlib as a site: 1.7 GB, over GitHub Pages' 1 GB limit.
  - The probability slice: 6,720 declarations, built in about 30 seconds.
- **Graphs through characterizations**, on the probability slice.
  - `condExp` from its characterization: 993 declarations instead of 1,196, and nothing of its
    construction left.
  - `ℝ` from an isomorphism `K ≃+*o ℝ` alone: 7 declarations left out of `condExp`'s graph, and 43
    of the Cauchy construction still reached, through the operations Mathlib defines separately.
  - With the operations named in the statement: 15 remain, all through `Real.commRing`, whose casts
    are defined on the construction.

---

## 6. Social review, tried out

The review sandbox is a small library whose claim is Euclid's theorem, stated with three
definitions of its own. The first version of `IsPrime` admitted 1, on purpose.

**The page.** The claim's page shows:
- the claim and what it rests on, as a graph and one card per declaration;
- for each declaration, what Lean checks about it and its review threads, with who reviewed, what
  they compared it with, which axes of the rubric they checked, their caveats, the replies and the
  status changes;
- coverage under the reader's policy: whether reviews by AI agents count, reviews made before
  something underneath changed, acceptances with caveats, authors' own reviews;
- what to review next, including the axes nobody has checked yet.

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

1. **The well-definedness analyzer, first part: domains at uses.** Done on 2026-09-27
   (well-definedness.md §2.5): on Mathlib's probability theory, 1,092 obligations in 4,181
   theorems. The strong law's and the central limit theorem's are shown, and one of optional
   stopping's is not. Done on 2026-09-28: hypotheses under binders (`∑ i ∈ s`, `∀ᵐ x ∂μ`), and the
   inside obligation for definitions' bodies, with the catalogue's domains for the logarithm and
   the moments.
2. **Domains and relations in staleness** (well-definedness.md §8). Changing a declared domain does
   not change the definition's meaning hash, so a review does not go stale. A review should record
   the domain it was made against.
3. **An evidence store for LeanMachineLearning,** with a claim page per claim: the suite in front
   of reviewers other than us. The store was made on 2026-09-29, in site-pilot, with issue forms
   and intake. It imports the probability store, so the site already shows reviews of `IndepFun`,
   `IdentDistrib` and `variance`. To do: a claim page per claim, and reviewers.
4. **The pull-request bot** (I4): what a change made stale, from two datasets and a store.
5. **The self-checks in the pilots' workflows** (piece 5).
   - Done: the kernel check runs in the pilots' workflows (not yet Tau Ceti's), and referee-site
     shows it.
   - To do: graph against hash after every new dataset; re-key old records through datasets of
     their commits, so that they follow the new rule rather than the old hash. Then checks 3 and 4
     (dependency-testing.md §9).
6. **The review workspace** (I2): a queue and progress that belong to the reader, over the shared
   store.
7. **The rest of well-definedness:**
   - `@[noncanonical]` and the obligations of choice;
   - invariance under a definition's declared relation;
   - whether a characterization's assumptions follow from the definition's declared domain;
   - the catalogue's growth, to the operations of well-definedness.md §2.1.
8. **The missing `@[specifies]` kinds,** and the missing-examples report
   (trusting-definitions.md §6, recommendations 1 and 2).
9. **Formal challenges** through ChallengeGen and Comparator, stored as S3 records: a proposed
   test (S3 `challenge`) whose statement a generator writes, or a disproof it finds.

Still open from suite-design.md §9:
- **the kernel check along `term` on large libraries:** its memory grows without bound on Tau Ceti
  (dependency-testing.md §9); it is kept as an optional check for small libraries;
- **hash migrations** in general: the change to `ltb-meaning/1` was first handled by legacy hashes
  and re-keying, then those were dropped; a future change of rule would need a decision again;
- **write policy** beyond "authors and maintainers";
- **the rest of federation:** signing, and whether stores should exchange records through
  servers rather than be read from git at build time. Imports are static today, so a page shows
  another store's records only after it is rebuilt;
- **the math-language layer;**
- **Mathlib's cross-reference attributes.**
