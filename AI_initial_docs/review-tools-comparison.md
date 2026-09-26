# Referee, trust, and Reviewed-by for Tau Ceti: a comparison

Snapshot of 2026-09-25: the tools as they were. What the suite built from them since is in
[status.md](status.md). Repositories read at:

| tool | repository | commit |
|---|---|---|
| Referee | [LeanMachineLearning/exposition](https://github.com/LeanMachineLearning/exposition) | `bb51dc2` (2026-09-22) |
| trust | [chrisflav/trust](https://github.com/chrisflav/trust), plus `trust-web`, `trust-cli`, `trust-server`, `trust-action` | `7fdbd6a` (2026-09-12) |
| Reviewed-by for Tau Ceti | [CBirkbeck/tauceti-reviewed-by-test](https://github.com/CBirkbeck/tauceti-reviewed-by-test), published at [cbirkbeck.github.io/tauceti-reviewed-by-test](https://cbirkbeck.github.io/tauceti-reviewed-by-test/) | `63a81e6` (2026-09-24) |

I also read the linked repositories: MeaningGraph, Characterization, ChallengeGen, LeanExposition,
semantic_hash, aftk, Comparator, TauCetiRoadmap, TauCetiReview, and the `CBirkbeck/TauCeti` fork.
For section 9 I also read Mathlib's `Mathlib/Tactic/CrossRefAttribute.lean`.

The tools compute dependency lists in five separate ways over the compiled environment, compared in
section 6.2. Reviewed-by has a sixth, simpler one over source text. A companion note,
[dependency-testing.md](dependency-testing.md), reviews how all six compute and test their lists.

---

## In short

All three tools start from the same observation. The kernel checks proofs, but nothing checks that
a *definition* or a *statement* means what its name says. So a human judgement has to be recorded
for each declaration, tied to a version of that declaration, and shown to the next reader. They
diverge on **who that judgement is for**:

- **Referee** is for **one reader auditing one project**. It is a static site, generated from the
  compiled environment, that separates claims from machinery. It shows what each statement rests
  on, including upstream packages, `sorry`s and axioms. Private, unauthenticated verdicts are kept
  in the browser. Most of its code goes into the *derived* analysis: dependencies, revision diffs,
  provenance, extraction, specifications and claims.
- **trust** is for **many people vouching for declarations across libraries**. It exports a
  dependency index that an interactive web graph reads. Its main contribution is the social layer:
  OpenPGP-signed certificates keyed by semantic hash, which federate between independent servers.
  Each reader keeps a personal, non-transitive trust list.
- **Reviewed-by for Tau Ceti** is a **lightweight community ledger for one AI-written library**.
  A reviewer submits a GitHub issue form or a comment. A GitHub Action appends a record to a JSONL
  ledger, and a Pages site shows the marks, tests, problem reports and "named" results for all of
  Tau Ceti's declarations (about 75k). It never touches `.olean` files: it parses source text in
  Python.

Referee is broad and deep for a single reader. trust has the most detailed multi-party trust
model. Reviewed-by is the cheapest way to collect reviews from many people and agents right now.

---

## 1. At a glance

| | Referee | trust | Reviewed-by for Tau Ceti |
|---|---|---|---|
| Author | Rémy Degenne | Christian Merten | Chris Birkbeck |
| Status | Alpha. CI action and releases per toolchain | "Experimental, LLM-generated, and not reviewed by a human" (README banner). Instance live at `trust.merten.dev` | "A test", started 2026-09-21. 4 marks recorded so far |
| Main question | *What does this result claim, what does it rest on, and what must I take on faith?* | *What does this statement definitionally rest on, what rests on it, and who vouches for it?* | *Who has checked this declaration, what tests does it pass, and is anything reported wrong?* |
| Reader | a referee, auditor or maintainer of one project | anyone, on any indexed library (core, Mathlib, …) | Tau Ceti's community, humans and AI agents |
| Input | compiled `.olean`s (run under `lake env`), source files, optional `semantic_hash` JSONL, `formalization.yaml`, Comparator configs, git history | compiled `.olean`s (under `lake env`) | Tau Ceti **source text** at a pinned commit, TauCetiRoadmap `STATUS.md` files, GitHub issues and comments |
| Output | multi-page static Verso site, `data.json`, standalone `.lean` files, audit JSON and Markdown report | static index (`decls.jsonl`, binary edge files, sharded code), `trust-marks.json`, signed certificates | static HTML plus JSON on GitHub Pages, `reviews.json`, docstring lines written into a fork of Tau Ceti |
| Stack | Lean 4 + Verso (plus D3 and about 2.7k lines of site JS). About 19k lines of Lean | Lean core library/exporter/CLI/server (about 4.4k lines in core), React/TypeScript frontend (about 9.7k lines), SQLite, `gpg` | Python (about 1.75k lines plus 1.3k lines of tests), vanilla HTML/JS, GitHub Actions |
| Dependency analysis (section 6.2) | own package [MeaningGraph](https://github.com/RemyDegenne/meaning-graph). The flat extraction ([ChallengeGen](https://github.com/RemyDegenne/challenge-gen)) computes its own list | its own statement and body edges (`Trust.Deps`), and [aftk](https://github.com/mathlib-initiative/aftk) for reverse dependencies | none (regex name resolution for `example`s only) |
| Where judgements live | browser `localStorage`, exportable JSON | `trust-marks.json` (local, can be committed) and certificates on federated nodes | JSONL ledgers committed to the repository by a bot |
| Authentication | none, by design | OpenPGP signatures (federated), or GitHub sign-in ("attested", local to one node only) | the GitHub account that opened the issue or wrote the comment |
| Toolchain coupling | binary must match the target's toolchain | a release per toolchain, matched by tag | none (reads text) |

---

## 2. Referee (`LeanMachineLearning/exposition`)

**What it is.** A Lean executable that reads a compiled project and builds a site. Its stated
purpose: *"a reader who did not write the library should be able to decide, for any result it
states, what that result says and what it rests on — without opening Lean."* It explicitly does
not aim to be API docs (doc-gen4), a textbook or a proof browser. It was first inspired by Matthew
Ballard's [LeanExposition](https://github.com/mattrobball/lean-exposition): a Verso site with
declaration cards, a D3 graph and Comparator-aware "trusted formalization base" tags.

**Pipeline.** There is one hard boundary: whatever needs a Lean environment produces *data*, and
rendering is a pure function of that data.

`collect` → `data.json` · `provenance` (git only) · `extract` / `extract-flat` → one `.lean` file
per declaration · `highlight-extracted` (also a compile check) · `build-site` (no environment).

Render-time flags (`--trust`, `--baseline`, …) can therefore change without re-importing the
project. It ships a composite GitHub Action (`LeanMachineLearning/exposition@<tag>`). The action
keeps the two pieces of state that must persist between runs: the previous `data.json`, used for
revision diffs, and a provenance ledger on its own branch.

**What the site shows.**

- **Declaration cards.** Each has a "statement anatomy" (the objects, their structure, the
  hypotheses, the claim; for definitions, the result type and body), the source, the proof, and a
  per-declaration dependency graph laid out by depth. There is deliberately no whole-project graph.
- **Theorems page.** It lists what the author wrote with `theorem` rather than `lemma`, which is
  the one piece of editorial intent the tool relies on.
- **Claims page.** It lists `status.main_results` from `formalization.yaml` (the Palomar registry
  metadata) or from a [Comparator](https://github.com/leanprover/comparator) setup. It is the only
  list on the site that is *asserted* rather than measured. It also shows literature dependencies
  and `status.scope`.
- **Sorries and assumptions.** `sorry` chains, extra axioms and unaudited upstream packages. It
  reports findings only and never a progress percentage.
- **Upstream trust (`--trust PKG`).** Trusting a package also trusts everything it depends on. The
  toolchain is always trusted and the default is to trust nothing. The dependency that matters is
  an upstream *definition a statement is about* (`meaningDeps`), not something a proof merely
  calls. Small upstream packages are expanded into their internal structure within a budget of 500
  constants. Mathlib is drawn as one flat band.
- **Specifications and characterizations**, read from the separate
  [Characterization](https://github.com/RemyDegenne/characterization) package.
  `@[specifies d "note"]` marks a theorem as evidence that `d` is the intended definition.
  `@[characterization property/existence/uniqueness]` records that a property pins `d` down up to
  a stated relation, and the attribute *checks* the shapes of the three declarations. The site
  lists definitions that have no specification, ranked by how much uses them. A characterized
  definition's graph can switch from "Construction" to "via its property".
- **Audit state.** Per declaration, a verdict: *unread*, *accepted* or *query* (with a note).
  **Coverage** is derived from it: a declaration is covered when it is accepted *and* everything in
  its statement closure is accepted. That exposes the state "accepted but not covered". Each verdict
  records the proof-irrelevant semantic hash at the time it was set, so a later build can show
  "accepted, then changed". The page states its limits itself: *nothing is authenticated, nothing
  is verified, and the browser is not storage*. Export and import go through JSON, and "Generate
  report" writes a Markdown referee report.
- **Revisions (`--baseline`).** It classifies changes as statement changes, body changes (which
  matter only for definitions), proof-only changes (no re-reading needed) and **indirect
  invalidation**. Indirect invalidation is a statement that reads the same byte for byte but whose
  meaning moved because a definition below it changed. With semantic hashes it adds a fourth
  class, "meaning changed underneath", for changes upstream or in unexposed code.
- **Provenance.** It keeps two facts apart: when the *file* was last edited (git blame) and when the
  *meaning* last changed (from the hash ledger). For example: *"meaning unchanged since v0.1; file
  edited 2026-07-29 without changing what it means."*
- **Standalone files.** One per declaration, with dependencies inlined and proofs replaced by
  `sorry`, linked to live.lean-lang.org. There are two tiers: readable (99.6% compile) and flat,
  rendered from `ConstantInfo` (100% compile). The extraction lives in
  [ChallengeGen](https://github.com/RemyDegenne/challenge-gen). The two tiers get their
  dependencies differently. The readable tier inlines MeaningGraph's closure. The flat tier ignores
  MeaningGraph's dependencies and builds its own list from the constants its printer writes out
  (section 6.2).
- **Scoped builds.** `--claims-only` and `--only DECL` restrict the build to the claims and their
  statement closure, which is about 2% of a library: 1901 declarations shrink to 37.
- **JunkValues.** A Lean-core-only linter for definitions that silently rely on junk values
  (`∫` of a non-integrable function is 0, and so on). Each rule is an existing kernel-checked
  theorem tagged `@[junk_value]`, and `fun_prop` or `norm_num` act as dischargers.

**Separate packages** (each depends only on Lean core, so a project can use one without Verso):
MeaningGraph, Characterization, ChallengeGen, JunkValues.

**Self-described gaps** ([`TRUST-GAPS.md`](https://github.com/LeanMachineLearning/exposition/blob/main/docs/design/TRUST-GAPS.md)):
it has one epistemic channel, *"a human reads statements, statically, from one build of one
project"*. It supports **no second reader**: merging audit files and reporting disagreements is
proposed but not built. It has **no CI policy gate** (`referee check --policy` is proposed). Its
extraction fidelity rests on the tool itself, and a Comparator run against the extracted files is
proposed as the fix.

---

## 3. trust (`chrisflav/trust` and companions)

**What it is.** In the README's words, a tool for "estimating the trust debt of a Lean statement:
what it definitionally rests on, and what rests on it." The project is split across five
repositories, each versioned by a different thing:

| repo | role | language |
|---|---|---|
| `trust` | exporter (`trust export/deps/rdeps/decl`), index format, marks, certificate format, federation rules as code, conformance vectors | Lean |
| `trust-web` | React app: dependency graph, rendered code, marks and certificates | TypeScript (deliberately a second implementation, so the browser checks signatures itself) |
| `trust-cli` | `trust-cert issue/sign/verify/publish/revoke/fetch/who-trusts/verify-bundle/import` | Lean |
| `trust-server` | certificate node: SQLite store, GitHub sessions, federation | Lean (`Std.Http`, `leansqlite`) |
| `trust-action` | GitHub Action that exports an index in a library's CI, as an artifact, a branch or a release | YAML |

**Dependency model.** In the README's words, trust builds on aftk, but it computes forward
dependencies itself (`Trust/Deps.lean`), adding three things aftk doesn't have:

- *edges* rather than a flat set;
- edges of the *statement* (the type) kept separate from edges of the *body*;
- a *data-carrying* test: traversal descends into a node whose type is not a `Prop` and stops at
  proofs.

aftk supplies module selection and the reverse dependencies of `trust rdeps`, which deliberately
cover types and values, proofs included (`Trust/Reverse.lean`).

Proof edges are left out by default, since they make up 89% of body edges in Lean core, and
`--with-proofs` adds them back. Inductive types use their constructors' types as a body. The
exporter can index any library, from Lean core (about 30 s, about 40 MB) to Mathlib (about 25 min).
Edges are stored as binary `Int32` pairs so the browser loads them without parsing.

**Frontend.** An interactive graph in both directions (dependencies and reverse dependencies), with
depth, direction and repository filters and a full-screen view. Hovering a node shows a card with
its docstring, signature and optionally its body. Declarations are rendered with clickable constant
ranges taken from the delaborator's info map. A **"trusted mode"** cuts the tree at trusted
declarations and *replaces* a characterized definition's dependencies with those of its
characterizing theorems. Indexes can be read from any GitHub repository (`?gh=owner/repo`) or from a
release.

**Marks (local, in `trust-marks.json`, each recording its commit):**

- `trusted`: someone vouched for this declaration at this commit;
- `characterize DEF THM…`: these theorems pin down this definition (an unchecked JSON claim);
- `protect`: store a hash snapshot of the declaration. `trust check` **exits non-zero** when a
  protected declaration has changed, so it can gate CI.

**Certificates and federation** (spec in
[`FEDERATION.md`](https://github.com/chrisflav/trust/blob/master/FEDERATION.md), protocol `trust/1`):

- A certificate is one key-holder's claim about one declaration, keyed by **semantic hash** (the
  pinned `semantic_hash` commit, proof-relevant `runFor`). The hash covers the whole definitional
  closure, so *vouching for a hash vouches for the whole subtree*, and any change underneath voids
  the certificate. The same certificate is valid in any repository where the hash still matches.
- The signature covers canonical bytes: eight fields in alphabetical order. Since the TypeScript
  server was retired, two implementations remain: Lean (core, CLI, server) and the browser's
  TypeScript. Generated conformance vectors pin them to agree byte for byte.
- Only **signed** entries federate. "Attested" entries (a signed-in account) stay on the node that
  made them. Identity across nodes is a key fingerprint, and usernames travel only as unverified
  hints. **Trust is not transitive**: following someone counts their certificates and nobody
  else's. The spec also covers revocation (signed by the same key), delegated queries with depth
  and budget limits, and peer discovery with an address policy.

**Self-described caveats.** It is LLM-generated and unreviewed, and all performance claims were
measured by the same process. Certificates are currently proof-relevant, so re-proving a lemma
voids them, which the code notes is "stricter than it needs to be".

---

## 4. Reviewed-by for Tau Ceti (`CBirkbeck/tauceti-reviewed-by-test`)

**Context.** [Tau Ceti](https://github.com/TauCetiProject/TauCeti) is an "AIs-welcome" library
downstream of Mathlib, incubated by the Lean FRO and the Mathlib Initiative. Humans steer it
through [TauCetiRoadmap](https://github.com/TauCetiProject/TauCetiRoadmap). Before merge, PRs are
reviewed by AI agents under per-angle rubrics in
[TauCetiReview](https://github.com/TauCetiProject/TauCetiReview). This test adds what that setup
lacks: **declaration-level review after merge**, modelled on the Linux kernel's `Reviewed-by:`
trailers.

**How it works.**

- `fetch_declarations.py` parses every Tau Ceti module with regexes (about 75k declarations across
  about 7k modules in the current index). For each declaration it keeps the name, kind, docstring
  and "the source a reviewer signs off": the whole of a definition, or a theorem up to its `:=`.
  It also records a **SHA-256 of that text with whitespace normalized**. The docstring notes that
  *"a deployment inside Tau Ceti would hash the elaborated terms instead."*
- A reviewer can **submit an issue form** ("Review this" pre-fills the declaration and version) or
  write lines like `Reviewed-by: <decl> — <evidence>` in a comment on issue #1. A workflow checks
  that the declaration exists, appends the record to `reviews/records.jsonl` (committed directly,
  no PR), replies, closes the issue and rebuilds the Pages site. GitHub authenticates the author.
- **AI agents are first-class but kept apart**: a hidden marker names the agent, model and session;
  an agent *must* give evidence while a person need not; counts are shown as "Reviewed-by · 12
  people · 5 AI".
- **Staleness**: a mark stays attached, but is greyed, once the hash of the declaration's text
  changes.
- **Tests** are not marks. Three kinds are listed:
  - *unit tests*: the `example`s whose statement names the declaration, found automatically;
  - *key results*: lemmas that pin the declaration down, listed through `Test:` comment lines,
    mostly by agents;
  - *suggested tests*: an issue that stays open until someone writes the test.

  A test "passes" while it is present at the pinned commit without `sorry`. The README notes that
  tests, unlike marks, never go stale: Lean re-checks them at every commit.
- **Problem reports**: a form requires *what* is wrong (false as stated, not the intended notion,
  or a misleading name or docstring) and *why* (a counterexample, a source, a failing step). The
  issue stays open until it is closed as fixed or as not planned, and the page flags the
  declaration with `!` in the meantime.
- **Named results**: taken from the roadmaps' generated `STATUS.md` files ("Named results", "Notable
  definitions") and from announcements by the Voyager Zulip bot. They get their own tab so readers
  see first what matters most.
- **Written back into code**: `apply.py` edits a fork (`CBirkbeck/TauCeti`, PR #1, merged into the
  fork). It adds a link to the review page under each module docstring and a line such as
  `Reviewed-by: 1 person and 1 AI agent / Tested by: 4 key results` to each reviewed declaration's
  docstring. The commit carries git trailers, and a daily workflow resynchronizes it.
- A daily cron moves the pinned Tau Ceti commit to `main` and rebuilds the site.

**Deliberately absent:** a dependency graph, closures, compiled data, and any cryptography beyond
GitHub's own authentication.

---

## 5. What they have in common

1. **The same gap.** Each tool says in its own words that the kernel settles proofs but not
   meaning. What is left is a human (or agent) judgement that a definition or statement is the
   intended one, recorded per declaration.
2. **Statement versus proof.** All three draw the line in the same place:
   - Referee's `meaningDeps` keeps a theorem's statement and a definition's data-carrying body.
   - trust's data-carrying traversal stops at proofs.
   - Reviewed-by hashes a theorem by its statement and a definition whole.

   All three reason that the kernel has already checked a proof, so a proof change needs no
   re-review.
3. **Judgements are tied to a version and can go stale.** Referee records a hash in each verdict
   and reports "accepted, then changed". trust records a commit in each mark and a semantic hash in
   each certificate. Reviewed-by records a text hash in each mark and greys the mark out. None of
   the three silently carries a judgement over to a changed declaration.
4. **"These theorems say what the definition means" as a first-class idea.** Referee has
   `@[specifies]`/`@[characterization]`, trust has `characterize` marks and the characterized cut
   in trusted mode, and Reviewed-by has key-result tests. Section 7 compares them.
5. **Derived data kept apart from human judgement.** All three regenerate derived data (the graph,
   the index, the declaration list) from the library, while human judgements go in a separate,
   append-only or diffable file: audit JSON, `trust-marks.json`, `reviews/*.jsonl`.
6. **A static site plus a CI action.** Each tool is meant to run in the library's own CI and
   publish static files: Referee's composite action produces a Verso site, `trust-action` produces
   an index that `trust-web` reads, and GitHub Actions publish Reviewed-by to Pages.
7. **`semantic_hash`** is used, or named as the right thing to use, by all three. Referee uses it
   optionally (both variants), trust by default (pinned), and Reviewed-by names it as what a real
   deployment should use.
8. **Honest about limits.** Each tool writes its own limits into the product: Referee's three limits
   appear on the page itself, trust carries its LLM-generated banner and non-transitivity rule, and
   Reviewed-by labels itself "a test".

---

## 6. Where they differ

### 6.1 Who the judgement is for

| | Referee | trust | Reviewed-by |
|---|---|---|---|
| Number of judges | one reader | many, each reader choosing whom to count | many, all shown together |
| What a judgement means | "I read this, and it says what its name claims" (private working aid) | "this key vouches for this hash", covering the **whole subtree** below it | "this is the intended mathematical notion" (public, attributed) |
| Negative judgements | *query* with a note (private) | only by revoking your own certificate | **problem report**, a public open issue with a required reason |
| Aggregation | none; merging is a proposed gap | per reader: your trust list plus "who else vouched" | counts of people and AI agents, with the full list one click away |
| AI agents | not modelled | not modelled | first-class, shown separately, evidence required |

The most important conceptual difference is **how far a judgement reaches**:

- In **Referee**, accepting a declaration covers *that declaration only*. Coverage is *derived*
  from the acceptances of its closure, so "accepted but not covered" is a visible state.
- In **trust**, a certificate is on a hash of the whole definitional cone, so it *is* a claim about
  the subtree. Nothing records whether the signer actually read the subtree. The "trusted mode" cut
  then treats a trusted node as a leaf.
- In **Reviewed-by**, a mark covers one declaration's text. There is no dependency information, so
  neither coverage nor subtree semantics can be expressed.

### 6.2 What counts as a dependency

| | Referee | trust | Reviewed-by |
|---|---|---|---|
| Engine | MeaningGraph: `getUsedConstants`, plus recovery of compiler helpers, `Expr.proj` structure names, notation expansions and coercion instances. The flat extraction uses its own list (below) | `Trust.Deps`: `getUsedConstants` on type and value, with constructor types standing in for an inductive's body. aftk for reverse dependencies | — |
| Edge kinds | `typeDeps` / `meaningDeps` / `deps` (with proofs) | statement edges / body edges / optional proof edges | — |
| Direction | mostly downward ("what this rests on"). A reverse "blast radius" view is proposed | **both directions are first-class** (`rdeps`, "used by" in the UI) | — |
| Upstream | package-level, with `--trust PKG`. Small packages are expanded and Mathlib stays a flat band | every declaration of the indexed import closure is a node (core and Mathlib included) | Tau Ceti only |
| Whole-library graph | rejected on purpose | full-screen graph of any closure | — |

#### Five ways of computing a dependency list

The tools and the libraries they rely on contain five separate implementations over the compiled
environment. Each is a separate body of code, and each makes its own choices:

| mechanism | code | used by | direct dependencies from | proofs | structure in `Expr.proj` | generated constants | scope |
|---|---|---|---|---|---|---|---|
| **aftk** | `AFTK/Dependency.lean` (`directDependencies`) | `aftk deps` / `rdeps`, `trust rdeps` | Lean core's `ConstantInfo.getUsedConstantsAsSet`: type and value together, with an inductive type pointing to its constructors | followed | missed before Lean 4.34 | kept as nodes | everything imported |
| **trust** | `Trust/Deps.lean` | `trust deps`, `trust export`, trust-web | `getUsedConstants` on the type (statement edges) and on the value (body edges), with constructor types for inductives | a proof is kept as a node but not entered. Proof edges only with `--with-proofs` | missed before Lean 4.34 | kept as nodes | everything imported |
| **MeaningGraph** | [meaning-graph](https://github.com/RemyDegenne/meaning-graph) | Referee's site, and ChallengeGen's readable tier | `getUsedConstants`, plus projection structures, notation expansions and coercion instances | separate closures without proofs (`meaningDeps`) and with them (`deps`) | recovered | expanded through to declarations a human wrote | project |
| **semantic_hash** | `SemanticHash/Hashing/Expr.lean` | the hash of every declaration, which must cover everything below it | its own walk over each expression | followed in the proof-relevant variant. The proof-irrelevant variant skips theorem bodies (including `_proof_n` theorems) but still follows proofs written inline | recovered, after a bug fix | constructors and recursors are hashed together with their inductive type | everything imported |
| **ChallengeGen flat tier** | `ChallengeGen/Flat.lean` | `referee extract-flat` | the constants its printer writes out, in fully explicit form | every proof subterm is printed as `sorry` and not entered (`isProof`) | missed as a direct dependency, but reached through the projected term | redirected to the declaration that owns them | project; everything else comes in through whole-module imports |

All five start from the same thing: the constants in an elaborated term. They differ in their
choices about proofs, projections, generated constants, and whether definition bodies count.
ChallengeGen's readable tier is not a sixth mechanism. It takes MeaningGraph's closure with proofs
and widens it to cover the notation commands and sibling declarations its source text needs.

Reviewed-by has a sixth mechanism, over source text rather than the compiled environment. For each
`example`, `fetch_declarations.py` resolves the identifiers written in its statement to Tau Ceti
declarations, through the surrounding namespaces and the file's `open`s, and counts the example as
a unit test of each. It computes direct links only, with no closure. Its review marks hash only the
declaration's own text, so nothing below a declaration is covered.

Only the flat tier's list is checked systematically from outside: a constant is on it because the
printer had to write it, so compiling the file (100% of 3164 files) shows the list is sufficient.
The readable tier's 99.6% compile rate shows only that MeaningGraph's closure *with* proofs is
sufficient. trust's integration CI checks a few hand-picked facts about `Nat.gcd` in Lean core.
None of the other statement-only closures, which are the minimal ones, has an external check.
[dependency-testing.md](dependency-testing.md) reviews how each mechanism is tested, its
shortcomings, and recommendations.

### 6.3 Change over time

| | Referee | trust | Reviewed-by |
|---|---|---|---|
| Measure | proof-irrelevant plus proof-relevant semantic hash, falling back to text | `semantic-v1` (proof-relevant, pinned `semantic_hash` commit) or `structural-v1`. The hasher name is recorded, and hashes from different hashers are never compared | SHA-256 of whitespace-normalized source text |
| Re-proving a theorem | no effect on verdicts | voids certificates and changes protected snapshots | no effect (only the statement text is hashed) |
| Renaming a binder, or reformatting | no effect with hashes | no effect | a binder rename marks it stale; reformatting does not |
| A definition below changes | **indirect invalidation** is detected and reported | the certificate is voided automatically (deep hash) | **not detected**: the statement text is unchanged |
| History | provenance ledger: when the meaning last changed, kept apart from git blame | snapshots per commit for protected declarations | the ledger keeps the version each mark was made at |
| CI gate | none yet (proposed) | `trust check` exits non-zero | none. The fork's CI checks that the docstring lines build |

### 6.4 Features only one of them has

- **Referee only:** the claims page (formalization.yaml and Comparator); sorry and axiom chains;
  statement anatomy; standalone extracted files and the web editor; revision diff with indirect
  invalidation; provenance; derived coverage; generated referee report; claims-only builds;
  JunkValues linter; *checked* characterization attributes; a specification-gap ranking.
- **trust only:** signed and portable certificates; the federation protocol, revocation and
  conformance vectors; reverse dependencies; an index of arbitrary libraries including Mathlib and
  core; protect/check; code rendered with UTF-16 constant ranges; marks editable live through
  `serve-marks`.
- **Reviewed-by only:** public problem reports tied to a fix workflow; AI and human attribution;
  tests as evidence (automatic unit tests, key results, suggestions); named results pulled from
  roadmaps and a Zulip bot; review status **written back into source docstrings**; zero toolchain
  coupling; setup that is trivially cheap (Python and GitHub).

### 6.5 Engineering trade-offs

- **Compiled environment versus source text.** Referee and trust must run a binary built for the
  target's exact toolchain: Referee checks the toolchain in its action, and trust cuts a release
  tag per Lean version. Reviewed-by avoids that entirely by reading text. The price is a regex
  parser, statement hashes that miss meaning changes underneath, and test discovery that
  approximates Lean's name resolution.
- **Where state lives.** Referee: in the reader's browser and an exported file ("the browser is
  not storage"). trust: in local JSON plus servers that relay other people's signed claims without
  becoming a trusted party themselves. Reviewed-by: in git, with GitHub as the identity provider
  and the write API.
- **Scale.** Referee has measured a whole-Mathlib build (93k declarations in 43 min and 10.9 GB for
  `build-site`) and relies on claims-only builds to stay readable. trust exports Mathlib in about
  25 min into a browser-loadable index. Reviewed-by covers about 75k Tau Ceti declarations with a
  search index plus per-module JSON loaded lazily.

---

## 7. One idea, three encodings

| idea | Referee | trust | Reviewed-by |
|---|---|---|---|
| "This declaration is right" | verdict *accepted* (private, hash-stamped) | `trusted` mark (commit) / signed certificate (semantic hash) | `Reviewed-by:` mark (text hash, GitHub user or agent) |
| "Something is off here" | verdict *query* plus a note | — | problem report (a public issue with a required reason) |
| "These theorems pin down this definition" | `@[specifies]` and `@[characterization]` **in the Lean source**, shapes checked by the attribute | `characterize DEF THM…` in `trust-marks.json`, unchecked | `Test:` key results in a JSONL file, plus `example`s that name the declaration |
| Graph using the characterization instead of the construction | "via property" view for complete characterizations | trusted mode replaces the dependencies with those of the characterizing theorems | — |
| "These are the main results" | `theorem` versus `lemma`, and `formalization.yaml` `main_results` / Comparator | — | named results from roadmap `STATUS.md` files and Voyager |
| "I accept this upstream code as-is" | `--trust PKG` (per package, transitive) | trusted declarations become leaves (per declaration) | — |
| "Tell me when this changes" | stale verdicts, `--baseline`, provenance | `protect` / `check` | greyed marks |

---

## 8. Observations on combining them

These are not recommendations to merge anything. They are the places where the tools already touch.

1. **Hash compatibility is the first obstacle.** Referee's verdicts use `semantic_hash`'s
   *proof-irrelevant* hash, from whichever revision the user built. trust's certificates use the
   *proof-relevant* on-demand hash at a pinned commit. So the same judgement produces different
   keys, and a Referee acceptance cannot be read as a trust certificate (or the reverse) until both
   sides agree on the revision and the variant. trust's own code already calls proof-relevance
   "stricter than it needs to be" for certificates, so proof-irrelevant is the natural common
   choice.
2. **Characterization has three encodings.** Referee's version is the only checked one: an
   attribute in the source that verifies the existence and uniqueness shapes. trust's trusted-mode
   cut is the most developed *use* of the idea in a graph. Reviewed-by's key results are the
   lowest-effort way to *collect* candidates from agents. A pipeline could run from Reviewed-by-style
   "Test:" lines, to a PR adding `@[specifies]`, to Referee and trust both reading the attribute.
3. **Multi-reader support, Referee's self-declared gap (TRUST-GAPS §5), is what the other two
   provide.** trust adds cryptographic identity and federation. Reviewed-by adds GitHub identity,
   AI attribution and public disagreement through problem reports. Referee's export already holds
   hash-stamped per-declaration verdicts, so it is close to an unsigned trust claim or a
   Reviewed-by ledger line.
4. **Coverage exists only in Referee.** Neither trust (which vouches for a subtree) nor Reviewed-by
   (which has no graph) can say "reviewed, but rests on definitions nobody reviewed". In an
   AI-written library such as Tau Ceti that is arguably the most useful signal, and it needs only a
   statement closure (from trust's index or Referee's `data.json`) joined with the ledger.
5. **Reviewed-by's text hash is its weakest point, and it says so.** A mark on a theorem survives a
   change to a definition it mentions. Either tool's compiled data would fix this: trust's
   `export --with-hashes` output, or Referee's `collect --hashes` output.
6. **CI gating** is built in trust (`check`), proposed in Referee (`check --policy`), and absent in
   Reviewed-by. Comparator's `permitted_axioms` exit code, discussed in Referee's TRUST-GAPS §8, is
   a third route.
7. **Only Reviewed-by models AI reviewers.** Tau Ceti already runs thousands of AI PR reviews
   through TauCetiReview. Declaration-level agent marks with required evidence, shown apart from
   human marks, is a distinction neither Referee nor trust can currently make.

---

## 9. What is built, sorted by type of tool

This section sorts what already exists into four types of tool:

1. **Surface or inject data about the code**: annotations and metadata saying what a declaration
   is, what it means, or where it comes from.
2. **Gather and compute data** from the code and from type-1 annotations: graphs, closures,
   hashes, findings.
3. **Visualize and interact**: views for a Lean expert reviewing, or for a non-expert exploring.
4. **Store, share and act on** what comes out of the interaction: reviews, reports, challenges.

Most of the projects above span several types, so they are split into components here. A
component appears under the type it mainly serves.

### 9.1 Type 1: surface or inject data

| component | project | what it records | where it lives | what Lean checks |
|---|---|---|---|---|
| `@[characterization property / existence / uniqueness]` | Characterization | the definition is the unique object with a property, up to a relation read off the uniqueness theorem | in the source | the shapes of the three parts, by `isDefEq`. Not whether the property says anything |
| `@[specifies d "note"]` | Characterization | this theorem is part of the specification of `d` | in the source | the target resolves and is a definition, and the tagged declaration is a proposition. It warns if the statement never mentions the target |
| `@[junk_value (generalizing i) "note"]` | JunkValues | when a total function collapses to a default value outside its intended domain, stated as a theorem `guards → lhs = rhs` | in the source, on the theorem that proves the collapse | the shape is checked when the attribute is written, and the rule itself is a kernel-checked theorem |
| `JunkValues.Extra` (Catalogue, Arithmetic) | JunkValues | the same kind of rule for Mathlib: integrals, conditional expectations, derivatives, sums, division, truncated subtraction, `⊤` coerced to 0 | a separate import, for libraries you cannot annotate | the named theorems must resolve. `#junk_rules` lists those that do not |
| `@[stacks]`, `@[kerodon]`, `@[wikidata]`, `@[lmfdb]`, `@[pibase]`, `@[dlmf]` | Mathlib (`CrossRefAttribute`, which replaced the Stacks-only attribute in 2026-05) | a link to an entry in an external mathematical database, stored in an environment extension and appended to the docstring | in the source | the tag is parsed. Whether it matches the declaration is not checked |
| `theorem` versus `lemma` | Lean convention, which Referee reads | results versus intermediate steps | in the source | nothing: Lean records both as the same kind |
| `formalization.yaml`: `status.main_results`, `literature_dependencies`, `status.scope` | Palomar registry format | the main results, results assumed from the literature, and the scope | a file beside the code | nothing. Referee warns when a named result does not exist |
| Comparator config | Comparator | which theorems a challenge module states, and which axioms are allowed | a file beside the code | read by Comparator itself (type 2) |
| `trust characterize`, `trust trusted`, `trust protect` | trust | a characterization, a vouch, a watched declaration, each with its commit | `trust-marks.json`, outside the code | nothing for the first two. `protect` stores a hash |
| roadmap `STATUS.md` "Named results", Voyager `Named:` lines | TauCetiRoadmap, Reviewed-by | which declarations are the named results and notable definitions | outside the code | only that the declaration exists |
| `Test:` key results | Reviewed-by | lemmas that pin a declaration down, with what each one checks | JSONL, outside the code | only that the declaration exists |
| review lines in docstrings (`apply.py`) | Reviewed-by, fork of Tau Ceti | review and test counts, and a link to the review page | in the source, written back from type 4 | the fork's CI builds and lints them |

The closest built tool to an attribute declaring a function's "true" domain is `@[junk_value]`. It
records the complement of that domain: the condition under which the function collapses, through
the theorem that proves the collapse.

### 9.2 Type 2: gather and compute

| component | project | what it computes | from | how it is verified |
|---|---|---|---|---|
| MeaningGraph | Referee's dependency | per declaration, the constants of the statement and of statement plus body, recovering compiler helpers, `Expr.proj` structures, notation expansions and coercion instances. Also reverse edges and topological closure | the compiled environment | proofs in `MeaningGraph/Proofs.lean`: the project boundary (`hasPrefixName`, `isInternalName`), `topologicalClosure` has no duplicates and is closed under dependencies, `projStructureNames` is complete. Nothing checks its dependencies from outside: the readable extraction's compile rate covers only its closure *with* proofs, and the flat extraction does not use its dependencies (section 6.2) |
| `Trust.Deps`, `trust export` | trust | statement edges, body edges and optional proof edges, following only data-carrying nodes and stopping at proofs. The web app walks the same edges in reverse. `trust rdeps` instead takes reverse dependencies from aftk, over types and values | the compiled environment. aftk selects modules and supplies `rdeps` | `lake test`, which doesn't check dependencies. The integration CI checks a few facts about `Nat.gcd` in an export of Lean core |
| `aftk deps` / `rdeps` | mathlib-initiative | flat sets of transitive dependencies and reverse dependencies, from Lean core's `getUsedConstantsAsSet` | the compiled environment | `tests/dependency.sh`, on a toy project. It tests query scoping and name resolution, not whether the dependencies are right |
| `aftk tech-debt` | mathlib-initiative | technical-debt markers with their locations (`sorry`, `axiom`, `maxHeartbeats`, `erw`, deprecated, …) | elaborated info trees | — |
| semantic_hash | mathlib-initiative | a rename-invariant structural hash, in proof-relevant and proof-irrelevant variants. To make a hash cover everything below a declaration, it walks the dependencies itself (section 6.2) | the compiled environment | the executable specification `HashingTests.lean` (about 230 checks on small fixtures). trust's `hash-invariants` re-checks the invariants on real declarations. Nothing checks its dependency walk on real code |
| ChallengeGen readable tier (Referee `extract`) | Referee | one standalone file per declaration, copying source text: MeaningGraph's closure with proofs inlined, widened to the notation commands and sibling declarations the source needs | the environment, the source, and the closure the caller passes in | compile checks (`highlight-extracted`, `check-extracted-compile.sh`): 99.6%, which shows MeaningGraph's closure with proofs is sufficient |
| ChallengeGen flat tier (Referee `extract-flat`) | Referee | one standalone file per declaration, printed from `ConstantInfo`, with proofs replaced by `sorry`. It computes its own dependency list from the constants its printer writes out (section 6.2) | the environment | compile check: 100%, which shows its own list is sufficient. For both tiers, nothing checks that the file states the same thing as the original (a Comparator run is proposed for this) |
| Referee `collect` | Referee | `sorry` chains and axioms, the upstream package surface expanded inside small packages, the specification pull, the claims scope (statement closure) | the environment, the source and `formalization.yaml` | the `Test` and `Proofs` libraries |
| "via property" graph | Referee | an alternative graph for a characterized definition: its property, its relation, and what those two mean | `@[characterization]` and MeaningGraph | built for complete characterizations only |
| trusted-mode cut | trust-web | a graph where trusted nodes are leaves and a characterized definition's dependencies are replaced by those of its characterizing theorems | marks, certificates and the index | `trustedMode.test.ts` |
| revision diff, provenance ledger | Referee | changes classified as statement, body, proof-only, indirectly invalidated, or changed underneath, plus when meaning last changed | two `data.json` files, semantic hashes, git | proofs of the `ChangeKind` invariants and the ledger (`Proofs/Diff.lean`, `Proofs/Provenance.lean`) |
| `trust check` | trust | protected declarations that changed or disappeared, with a non-zero exit code | marks and the environment | — |
| JunkValues linter, `#junk_check`, `scanProject`, Discovery | JunkValues | junk values used where nothing in scope rules them out (guarded, unguarded or triggered), split into statement findings (a vacuity risk) and body findings (a meaning risk). Discovery proposes candidate rules: 1417 over Mathlib | the environment, the rules, and a discharger (`fun_prop`, `norm_num`) | unit tests, and integration tests on real Mathlib code |
| Comparator | leanprover | certifies that a solution proves the same statements as a challenge, within the allowed axioms, through kernel replay (optionally also nanoda), inside a sandbox | challenge and solution modules | it is itself the verifier |
| `fetch_declarations.py` | Reviewed-by | declarations, text hashes, and `example`s resolved as unit tests | source text, with regexes | Python unit tests. Name resolution is approximated |

### 9.3 Type 3: visualize and interact

| component | project | for whom | what it shows |
|---|---|---|---|
| Referee site | Referee | an expert reviewer or referee | declaration cards with statement anatomy, per-declaration graphs by depth with upstream blocks, the Theorems, Claims, Sorries and assumptions, Specifications, Changes and Browse pages, verdict controls on graph nodes, and extracted files that open in live.lean-lang.org |
| trust-web | trust | anyone exploring a library, core and Mathlib included | a full-screen graph in both directions, hover cards, rendered code with clickable constants, trusted mode, who vouches for each node |
| Reviewed-by page | Reviewed-by | the Tau Ceti community | search across all declarations, a Named tab, marks, tests and problems, and "Review this", "Report a problem" and "Suggest a test" buttons |
| LeanExposition | Matthew Ballard | mathematician-facing exposition | a Verso Manual site: cards, Uses and Used by, a whole-project D3 graph with a chapter filter, Comparator tags for the trusted formalization base |
| linter warnings, `#junk_check`, `#junk_rules` | JunkValues | the author, in the editor | findings on the declaration being written |
| `trust deps` / `rdeps` / `decl`, `trust-cert show` / `who-trusts` | trust | the command line | JSON graphs, rendered code with constant ranges, and who signed what |
| docstring review lines | Reviewed-by | anyone reading the source or the generated docs | review and test counts, and a link to the review page |

None of these was designed specifically as an exploration view for non-experts. LeanExposition's
mathematician-facing site and Reviewed-by's search with its Named tab come closest.

### 9.4 Type 4: store, share and act on interaction data

| component | project | what it stores | how it is shared | identity | challenges |
|---|---|---|---|---|---|
| audit export and import | Referee | verdicts (accepted or query, with a note), each with its proof-irrelevant hash | a file the reader chooses to pass on, and a generated Markdown report | none | a *query* is a private question |
| `trust-marks.json`, `serve-marks`, `sync-marks` | trust | trusted, characterized and protected marks, with commits and hash snapshots | committing the file, or exporting it into the index as `marks.json` | none | — |
| certificates: `trust-cli`, `trust-server`, `FEDERATION.md` | trust | signed claims keyed by semantic hash, and revocations | federated nodes with delegated queries | OpenPGP key. GitHub sign-in for "attested" entries, which stay on one node | — |
| ledgers and workflows | Reviewed-by | Reviewed-by marks, key-result tests, suggested tests, problem reports, named results | the git repository, `reviews.json`, docstring lines | GitHub account, and a marker for AI agents | **suggested test**: an issue that stays open until the property is proved in Tau Ceti. **Problem report**: an issue that stays open until the declaration is fixed |
| TauCetiReview, TauCetiData | Tau Ceti | AI review verdicts for each PR and rubric, and meta-review A/B judgments | a public archive | agents, with human meta-reviewers | `block` and `request_changes`, at PR level rather than per declaration |
| ChallengeGen with Comparator | Referee, leanprover | nothing (no storage) | challenge files and certifications | — | the machinery for *formal* challenges: ChallengeGen writes the challenge and Comparator certifies a solution |

The only challenge mechanism in use is Reviewed-by's "Suggest a test". It asks contributors to prove
that a definition has an expected property, but the property is written as free text in an issue.
The pieces for a formal version exist but are not connected to any review tool: a challenge would
be a Lean statement about the definition with `sorry`, ChallengeGen could produce its standalone
file, and Comparator could certify the library's proof.

### 9.5 What the sorting shows

- **Type 1 in the source is thin.** Only Characterization, JunkValues and Mathlib's cross-reference
  attributes put information into the code. The rest (characterize marks, key results, named
  results, main results) lives outside the code, in several formats, and nothing checks it.
- **Type 2 is duplicated.** There are six ways of computing a dependency list (section 6.2). Five
  work over the compiled environment: aftk, trust's `Deps`, MeaningGraph, semantic_hash's own walk,
  and ChallengeGen's flat printer. The sixth is Reviewed-by's name resolution over source text.
  There are also three change detectors: Referee's diff, `trust check`, and Reviewed-by's text
  hash. Proofs cover MeaningGraph's closure algorithm and project boundary, but not its
  dependencies. Only the flat printer's list is checked systematically from outside, by
  compiling (see [dependency-testing.md](dependency-testing.md)).
- **Type 3 has three frontends**, and each one does its own gathering underneath. Nothing lets one
  frontend reuse another's type-2 output, so a new view cannot yet be "cheap" in the sense of reading
  data that already exists.
- **Type 4 has three stores that cannot read each other**, with different identity models and
  different hash keys (point 1 of section 8). Formal challenges are not built.
