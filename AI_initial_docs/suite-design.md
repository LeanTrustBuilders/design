# A shared suite of trust tools: proposal

Snapshot of 2026-09-25, with the decisions taken while building it added on 2026-09-26. A proposal
for discussion among the authors of Referee, trust, Reviewed-by, MeaningGraph, ChallengeGen,
Characterization, JunkValues, semantic_hash and Comparator: what a single suite built from these
tools could look like, its independent pieces, how data flows through it, who uses which piece, and
in what order to build it.

**What has been built** since, in the [LeanTrustBuilders](https://github.com/LeanTrustBuilders)
organization, and what has not: [status.md](status.md).

It builds on the other notes in this folder:
- [review-tools-comparison.md](review-tools-comparison.md): what each tool does today;
- [dependency-testing.md](dependency-testing.md): how dependency lists are computed and tested;
- [trusting-definitions.md](trusting-definitions.md): the kinds of evidence for a definition;
- [interfaces-by-audience.md](interfaces-by-audience.md): who needs which interface;
- [data-formats.md](data-formats.md): the current data formats;
- [reviews.md](reviews.md): what reviews contain and how they are used (the S3 review fields);
- [claims-and-importance.md](claims-and-importance.md): how to find what matters and guide readers
  to it;
- [status.md](status.md): where the suite stands, repository by repository.

---

## 1. Summary

The suite is a set of small pieces that agree on **three shared specifications**. Only one piece
depends on the Lean toolchain; everything else reads files and runs anywhere.

It combines the best choices already made by the existing projects:
- **Referee:** whatever needs a Lean environment produces data; everything else only reads data.
- **Characterization, JunkValues, MeaningGraph:** Lean packages that depend on nothing but Lean
  core, so a library can adopt one without the rest.
- **trust:** an index format built for scale, canonical signed records, and releases tagged per
  toolchain.
- **Reviewed-by:** low-friction intake from a browser, and append-only ledgers in git.

---

## 2. Principles

1. **The three specifications come first:** how to identify a declaration, the dataset format, and
   the evidence record format. Every piece reads or writes these, so any piece can be replaced.
2. **Only the Lean-side producers touch `.olean` files:** the extractor and the analyzers, released
   per Lean toolchain. Views, stores, and computations over the data run without Lean.
3. **Checked facts go in the code; judgements go in the evidence store.** What the kernel can
   check lives in the library as annotations. What people and agents assert lives in versioned
   records.
4. **Every record says how it is backed:** checked by the kernel, computed by a tool, asserted by a
   person, or asserted by an AI.
5. **Every record is accountable.** It names the GitHub account it came from, or is labelled as an
   AI agent's, or both. There are no anonymous records: a reader's private judgements stay in their
   browser until they publish them under their account. Signatures come later, on top of these
   identities. (Decided on 2026-09-26; the first version of this note proposed a choice per record
   between anonymous, GitHub identity and signed.)
6. **Nothing becomes stale silently.** Records are keyed by meaning, so a change underneath shows as
   "stale".
7. **New data arrives without new releases.** A new attribute or a new analysis adds a facet to the
   dataset (section 3.1). No existing reader breaks, and the extractor needs no new release
   (section 3.2).

---

## 3. The three specifications

| spec | contents | built from |
|---|---|---|
| **S1: declaration key** | a name, module and package at a commit and toolchain, plus a **meaning hash** (the proof-irrelevant semantic hash, which is deep: it covers everything below), a **content hash** (proof-relevant), and a **local hash** (the declaration's own statement and data, with references by name). The local hash lets a review tell "changed itself" from "changed underneath" (reviews.md §1). The hashers are identified by semantic_hash revision and variant | Referee's verdict hashes; trust's hasher field and `structural-v1` hasher |
| **S2: dataset** | a directory with a `meta.json` header saying which tool and version produced it, the toolchain, the commit, the hasher, the scope, and which facets and edge files are present. Then a **minimal** `decls.jsonl` (identity, kind, hashes); **one edge file per notion of dependency** (statement, statement plus data, full term, source), in binary; **one file per facet** for everything else about declarations (section 3.1); code shards | trust's index layout, plus the per-declaration fields of Referee's `data.json`, split into facets |
| **S3: evidence record** | JSONL, each record with a schema tag, the S1 key of its subject, a kind (review, comment, status, test link, named result; later challenge, certificate, …), its backing, who made it (a GitHub account, an AI agent, or both; never anonymous), origin (web form, issue, CLI, agent), a timestamp, links to other records (supersedes, replies to), and an optional signature over canonical bytes. **Reviews** add: the kind of subject, a verdict (`accept`, `problem` with a category, or `question`), the reference compared against, **what was checked** (a checklist of failure modes), caveats, a rationale, and the reviewer's involvement. [reviews.md](reviews.md) specifies them. Records live in **evidence stores**: a directory of a git repository, append-only | Reviewed-by's ledger records; trust's canonical claims; Referee's verdicts |

Each specification has a version and conformance vectors, as trust already does for its federation
protocol. The notions of dependency named in S2 are those of dependency-testing.md §2.

**As built** ([LeanTrustBuilders/specs](https://github.com/LeanTrustBuilders/specs), version 0 of
each): S2 has three edge notions, `statement`, `meaning` and `term`, with the source dependencies
(notation, coercion instances) folded into all three; and, instead of code shards, facets giving
statements taken apart and signatures with the constant each identifier names. S3 has the kinds
`review`, `comment`, `status`, `test` and `named`, and defines evidence stores.

### 3.1 Extending the dataset: facets

Everything S2 says about a declaration, beyond its identity, kind and hashes, is a **facet**:
docstrings, source ranges, `sorry` and axioms, `@[specifies]` and characterization links, junk-value
findings, and any attribute or analysis defined later. Referee's 30-field declaration record becomes
about a dozen facets.

- **One file per facet:** `facets/<name>.jsonl`, one line per declaration it applies to,
  `{"decl": <key>, …payload}`.
- **Relations between declarations** that are not dependencies get their own edge file, declared the
  same way.
- **`meta.json` lists the facets and edge files present:** for each, its name, schema version,
  producing tool and version, and a one-line meaning.
- **Readers ignore facets they don't know,** so adding one never breaks an existing view. A view
  that needs a facet checks its schema version.
- **A facet can be added to an existing dataset later,** by a separate tool, as long as it is keyed
  to the same commit. For example, an analysis written the following week adds its file without
  re-running the extractor.
- **A registry of facet names and schemas** lives with the specifications, so two tools don't
  define the same name differently.
- **What belongs in S2 and what in S3:** a facet is a deterministic function of the code and of the
  producing tool's version. Anything asserted by a person or an agent is an S3 evidence record, not
  a facet.

### 3.2 Reading attributes out of Lean

A new attribute stores its data in an environment extension inside the `.olean`. To read it, the
reading process needs the extension's code:

- **Today's readers link the package.** Characterization's README and Referee's lakefile both say
  so: extension entries are matched to registered extensions by name, and are silently dropped
  otherwise.
- **Registration alone isn't enough.** Lean core registers extensions declared in imported modules
  when initializers are enabled (`importModules (loadExts := true)` runs their `[init]`
  declarations in `finalizePersistentExtensions`). But the reader's compiled code still has no typed
  handle with which to decode the entries.

If the extractor linked every annotation package, each new attribute would need a new extractor
release for every toolchain, and the extractor would depend on every package: the opposite of
independent pieces. The options:

| option | how it works | cost |
|---|---|---|
| **a. One generic extension** | a tiny core package, depending on Lean core only, provides one extension holding `(attribute, declaration, payload as JSON)` entries, plus a helper to define an attribute on top of it with its own checks. The extractor links only this package and exports each attribute as a facet named after it | annotation packages must depend on the core package, which is cheap. Existing attributes, such as Mathlib's cross-references, need a dedicated reader, or a change so that they also write to the generic extension |
| **b. Exporter convention** | each annotation package includes an exporter function, found through an attribute, that turns its extension into JSON. The extractor imports with initializers enabled and runs the exporters in the interpreter | no linking, and each package keeps its own types. But code from the target's dependencies runs inside the extractor, which needs a sandbox like Comparator's `landrun`. The exporter's signature becomes one more specification to version |
| **c. Link each package** (today's approach) | the extractor depends on each annotation package | fine for a few core attributes; poor for an open-ended set |

**Proposed, and built** ([TrustAnnotations](https://github.com/LeanTrustBuilders/annotations), with
JSON payloads): option **a** for every attribute designed for the suite: the new `@[specifies]`
kinds, domain annotations, and later ones. Dedicated readers, or option **b**, for attributes that
already exist elsewhere, such as Mathlib's. Defining a new attribute on the core package is then all
it takes: its data appears as a new facet at the next extraction.

Analyses that are not attributes work the same way. An analyzer is a separate executable that runs
under `lake env`, reads `decls.jsonl` to line up keys, and writes its facet into the dataset. Only
its output has to follow the specification.

---

## 4. The pieces

In the order data flows through them. The "type" column refers to the kinds of tool discussed in
review-tools-comparison.md §9, plus a fifth kind: tools that generate evidence by trying to break
the code.

| # | piece | type | what it does | built from |
|---|---|---|---|---|
| 1 | **annotation packages** (Lean, no dependencies beyond the core package) | 1 | a **core package** with one generic extension that new attributes are built on (section 3.2); on top of it, `@[specifies]` with kinds (property, example, non-example, value, agreement, known result), `@[characterization]`, `@[junk_value]` and domain annotations. Mathlib's cross-reference tags are read by a dedicated reader | Characterization, JunkValues |
| 2 | **extractor** (Lean, one release per toolchain) | 2 | runs under the target's `lake env` and writes S2 in one pass: dependencies by notion, hashes, rendered code, `sorry` and axioms, and a facet for every attribute built on the core package. It links only the core package, so a new attribute needs no new release | MeaningGraph, semantic_hash, trust's code renderer, Referee's `collect` |
| 3 | **analyzers** (Lean, separate executables) | 2 | junk-value scan; choice, instance and generality reports; inhabitation and consistency checks. Each runs under `lake env` and adds its own facet to the dataset, possibly after the extractor | JunkValues; the proposals in trusting-definitions.md |
| 4 | **standalone files and certification** | 2 | a self-contained file per declaration (readable and flat); certification of answers to challenges | ChallengeGen, Comparator |
| 5 | **self-checks** | 2 | compares each dependency notion against the flat printer's list, and checks closures by kernel replay, in the extractor's own CI (dependency-testing.md §7) | new |
| 6 | **evidence core**, a library in TypeScript or Rust as well as Lean | 2 and 4 | pure functions over S2 and S3: staleness, carrying reviews across renames by hash, coverage over closures, review queue ranking, revision diff with indirect invalidation, provenance | Referee's diff and provenance logic, extracted as a library |
| 7 | **evidence store** | 4 | S3 records in a git repository by default (append-only, one writer); intake from GitHub issue forms and comments, a web form, the CLI and agents | Reviewed-by's ledger and workflows |
| 8 | **signing and federation** (optional) | 4 | signs S3 records; nodes exchange signed records keyed by meaning hash | trust-cli, trust-server and `FEDERATION.md`, generalized from certificates to every record kind |
| 9 | **evidence generators** (agents and tools) | 5 | proposing and checking examples and non-examples, disproof attempts, blind re-definition, mutation of specifications, value checks against LMFDB, DLMF and OEIS. They write S3 records, and **pull requests** that add examples or `@[specifies]` to the library | `plausible`; TauCetiReview-style agents; mostly new |
| 10 | **views**, which only read S2 and S3 | 3 | static site (claims, evidence cards, a math-language layer with a conventions panel); graph explorer across libraries; review workspace; editor extension; pull-request bot and policy gate; machine interface (CLI and MCP) for agents; dashboard | Referee's site, trust-web, the Reviewed-by page |

Pieces 1 to 5 are Lean code tied to a toolchain. Adding an attribute (on the core package) or an
analysis (as a separate executable) doesn't require a new extractor release. Everything from 6
onwards can be written in any language and upgraded independently.

---

## 5. How data flows

```
 library source + annotations (1)                     informal sources: papers, Stacks, LMFDB…
          │ lake build                                            │ linked by tags
          ▼                                                       ▼
       .olean ──► extractor (2) + analyzers (3) ──► dataset S2 ◄──┘
                      │                                  │
                      ▼                                  ▼
             standalone files (4) ──► generators (9) ──► evidence S3 (7) ◄── people and agents,
                      │                   │                  ▲   │             through the views (10)
                      │                   │ pull requests:    │   └──► federation (8)
                      ▼                   ▼ examples, specs   │
                 Comparator ──► S3    library source ◄────────┴── fixes for problems and challenges

 S2 + S3 ──► evidence core (6) ──► staleness · coverage · queue · diff · provenance ──► views (10)
```

### The review loop

1. A reviewer takes an item from the queue and reads its evidence card.
2. They record a verdict, a problem or a challenge. The staleness engine marks it current.
3. When the code changes, the extractor produces a new dataset, and the core recomputes which
   records went stale.
4. The queue shows what needs looking at again, ranked by what the claims rest on.

### The strengthening loop

Generators and challenges turn questions into Lean code: an example, a non-example, an agreement
theorem, a `@[specifies]` link. It lands in the library by pull request, where the kernel checks it,
and keeps checking it at every build. **Asserted evidence becomes checked evidence wherever it
can**, which moves trust from people's judgements to the kernel.

### In CI, on each pull request

The action runs the extractor on the base and on the head. The core compares the two, and the bot
comments with:
- the definitions whose meaning changed, including indirectly;
- the reviews the change made stale;
- the claims affected.

A policy gate can fail the build, for example when a claim's closure gains a `sorry` or an
unreviewed definition.

---

## 6. Who uses which piece

The personas are those of interfaces-by-audience.md §2.

| persona | pieces they touch |
|---|---|
| P1 domain expert without Lean | static site or workspace, in the math-language layer; a web form or an issue to give a verdict or report a problem |
| P2 Lean expert | explorer, formal layer, standalone files, editor extension |
| P3 author or maintainer | annotation packages and their linters, pull-request bot and policy gate, reports of missing evidence, dashboard |
| P4 returning reviewer | review workspace: queue, checklists, stale items |
| P5 user of a result | claim page with its evidence card and trust surface, or a badge |
| P6 referee of a paper | a site built for the claims only, the paper map, an exported report |
| P7 explorer or learner | static site and explorer, with examples |
| P8 manager | dashboard |
| P9 agent | machine interface to query and submit; generators; answers to challenges |

---

## 7. Three ways to deploy it

| setting | what runs | server? |
|---|---|---|
| **one formalization** (a paper, a thesis) | the CI action: extractor, then static site; evidence as JSONL in the repository; private browser audit | no |
| **community library** (Tau Ceti style) | the above, plus GitHub intake, review workspace, agents, and the pull-request bot | GitHub Actions; a small service for the workspace |
| **ecosystem** (Mathlib and the libraries above it) | datasets published per release; federation nodes exchanging signed evidence keyed by hash; explorer across libraries | federation nodes |

---

## 8. What happens to the existing code

| existing | becomes |
|---|---|
| MeaningGraph | the dependency engine inside the extractor, with its notions of dependency made explicit. *Done: moved to LeanTrustBuilders, made fast, and given options for trust's choices of graph* |
| semantic_hash | the hashing, with a header on its output and one pinned variant used for keys |
| trust's export and index | the basis of the S2 format and of the extractor's writer |
| Referee `collect` | extractor fields. Its `build-site` becomes the static site; its diff and provenance logic become the evidence core. *Done, as trust-extract, referee-site and evidence-core* |
| ChallengeGen | standalone files, challenges, and the independent list that the self-checks compare against |
| Characterization, JunkValues | the annotation packages, with new kinds, rebuilt on the core package's generic extension. *Done for Characterization; not yet for JunkValues* |
| trust-web | the explorer. *Done as a fork that reads indexes written from our datasets* |
| trust-cli, trust-server | signing and federation, for every kind of record |
| Reviewed-by | the GitHub intake, the ledger conventions of S3, and write-back into docstrings as one more view. *evidence-store implements its model of intake (issue forms, comments, a bot keeping the ledger) for any repository; the Tau Ceti pilot keeps its records in an S3 store but still uses its own intake* |
| Comparator | certification of challenges and claims |
| aftk | stays a separate query and diagnostics tool on live environments. The dataset replaces its role as a source of dependencies |
| TauCetiReview | one agent that uses the machine interface |

---

## 9. Open questions to settle early

1. **One dependency engine, or several?** Suggested: one engine for the dataset, with the flat
   printer kept as an independent check (dependency-testing.md). *Settled: MeaningGraph, with
   options for other tools' choices; the independent check is not built yet.*
2. **Hash migrations.** When semantic_hash changes, every key changes. Records need both old and new
   hashes during a transition, and a migration that maps them through names at a commit. *Now
   pressing: [meaning-hash.md](meaning-hash.md) proposes a meaning hash of our own, derived from the
   meaning graph's rule, which would be the first such migration.*
3. **Extraction at Mathlib scale.** Extract incrementally, per module, caching by `.olean` hash.
   *Open: libraries on Mathlib are extracted in parts (Tau Ceti in 80 seconds), Mathlib itself not
   yet.*
4. **Write policy for evidence.** Who may write, whether AI output is rate-limited, how
   disagreements are shown. Suggested: decided per project. *Partly settled: anyone with a GitHub
   account may write under it, agents say so, statuses belong to a record's author and the store's
   maintainers, and disagreements are shown side by side. Rate limits are open.*
5. **The math-language layer.** Generate it from structured data where possible; label AI
   paraphrase, and always show the formal text beside it.
6. **Where state lives.** Static wherever possible; a service only for the workspace, identity and
   federation. *So far everything is static: identity is GitHub's, and changes go through issue
   forms a page prefills.*
7. **The generic extension's payload.** Arbitrary JSON is the most open choice. A small typed
   vocabulary (declaration names, strings, numbers, lists) would let the core package check more
   when an attribute is written, at the cost of flexibility. The choice fixes what "defining an
   attribute on the core package" means. *Settled: JSON.*
8. **Attributes that already exist elsewhere,** such as Mathlib's cross-references: a dedicated
   reader in the extractor, the exporter convention of section 3.2, or a change upstream so they
   also write to the generic extension.

---

## 10. Suggested phases

Progress as of 2026-09-26, in italics; details in [status.md](status.md).

1. **The specifications, and extractor v1.** Write S1 to S3 with conformance vectors, and start the
   facet registry. Write the core annotation package with its generic extension. Merge Referee's
   `collect`, trust's export and the hashing into one extractor with self-checks. Port the static
   site and the explorer to read S2. *Done, except the self-checks.*
2. **Evidence.** The store and its intake (GitHub and CLI), the evidence core, and the pull-request
   bot. Migrate Reviewed-by's ledgers, Referee's audit exports and trust's marks into S3. *Done,
   except the pull-request bot.*
3. **Richer evidence.** The new `@[specifies]` kinds and domain annotations, built on the core
   package; the analyzers, as separate executables adding facets; evidence cards in the views; and
   the review workspace. *Begun: example and non-example attributes, and a claim's page with each
   declaration's evidence and reviews.*
4. **Scale and automation.** Generators and agents, challenges certified by Comparator, signing and
   federation, the editor extension, and the dashboard. *Not started, though AI agents already
   review through the store.*
