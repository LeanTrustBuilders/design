# A shared suite of trust tools: proposal

Snapshot of 2026-09-25. A proposal for discussion among the authors of Referee, trust, Reviewed-by,
MeaningGraph, ChallengeGen, Characterization, JunkValues, semantic_hash and Comparator: what a
single suite built from these tools could look like, its independent pieces, how data flows through
it, who uses which piece, and in what order to build it.

It builds on the other notes in this folder:
- [review-tools-comparison.md](review-tools-comparison.md): what each tool does today;
- [dependency-testing.md](dependency-testing.md): how dependency lists are computed and tested;
- [trusting-definitions.md](trusting-definitions.md): the kinds of evidence for a definition;
- [interfaces-by-audience.md](interfaces-by-audience.md): who needs which interface;
- [data-formats.md](data-formats.md): the current data formats.

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
2. **Only the extractor touches `.olean` files.** It is released per Lean toolchain. Views, stores
   and analyses of the data run without Lean.
3. **Checked facts go in the code; judgements go in the evidence store.** What the kernel can
   check lives in the library as annotations. What people and agents assert lives in versioned
   records.
4. **Every record says how it is backed:** checked by the kernel, computed by a tool, asserted by a
   person, or asserted by an AI.
5. **Accountability is chosen per record,** not per tool: anonymous, GitHub identity, or signed.
6. **Nothing becomes stale silently.** Records are keyed by meaning, so a change underneath shows as
   "stale".

---

## 3. The three specifications

| spec | contents | built from |
|---|---|---|
| **S1: declaration key** | a name, module and package at a commit and toolchain, plus a **meaning hash** (the proof-irrelevant semantic hash) and a **content hash** (proof-relevant). The hasher is identified by semantic_hash revision and variant | Referee's verdict hashes; trust's hasher field |
| **S2: dataset** | a directory with a `meta.json` header saying which tool and version produced it, the toolchain, the commit, the hasher, the scope, and which notions of dependency are present. Then declarations as JSONL (kind, hashes, `sorry`, axioms, source range, signature, docstring); **one edge file per notion of dependency** (statement, statement plus data, full term, source), in binary; the in-code annotations as JSONL; analysis findings as JSONL; code shards | trust's index layout, plus the per-declaration fields of Referee's `data.json` |
| **S3: evidence record** | JSONL, each record with a schema tag, the S1 key of its subject, a kind (review, problem, challenge, answer, test link, named result, certificate, …), its backing, identity (none, GitHub or a key), **what was checked** (a checklist of the failure modes in trusting-definitions.md §2), payload, origin (web form, issue, CLI, agent), a timestamp, and an optional signature over canonical bytes | Reviewed-by's ledger records; trust's canonical claims |

Each specification has a version and conformance vectors, as trust already does for its federation
protocol. The notions of dependency named in S2 are those of dependency-testing.md §2.

---

## 4. The pieces

In the order data flows through them. The "type" column refers to the kinds of tool discussed in
review-tools-comparison.md §9, plus a fifth kind: tools that generate evidence by trying to break
the code.

| # | piece | type | what it does | built from |
|---|---|---|---|---|
| 1 | **annotation packages** (Lean, no dependencies) | 1 | `@[specifies]` with kinds (property, example, non-example, value, agreement, known result); `@[characterization]`; `@[junk_value]` and domain annotations. Mathlib's cross-reference tags are read as they are | Characterization, JunkValues |
| 2 | **extractor** (Lean, one release per toolchain) | 2 | runs under the target's `lake env` and writes S2 in one pass: dependencies by notion, hashes, annotations, rendered code, `sorry` and axioms | MeaningGraph, semantic_hash, trust's code renderer, Referee's `collect` |
| 3 | **analyzers** (Lean, plugins to the extractor) | 2 | junk-value scan; choice, instance and generality reports; inhabitation and consistency checks. Findings go into S2 | JunkValues; the proposals in trusting-definitions.md |
| 4 | **standalone files and certification** | 2 | a self-contained file per declaration (readable and flat); certification of answers to challenges | ChallengeGen, Comparator |
| 5 | **self-checks** | 2 | compares each dependency notion against the flat printer's list, and checks closures by kernel replay, in the extractor's own CI (dependency-testing.md §7) | new |
| 6 | **evidence core**, a library in TypeScript or Rust as well as Lean | 2 and 4 | pure functions over S2 and S3: staleness, carrying reviews across renames by hash, coverage over closures, review queue ranking, revision diff with indirect invalidation, provenance | Referee's diff and provenance logic, extracted as a library |
| 7 | **evidence store** | 4 | S3 records in a git repository by default (append-only, one writer); intake from GitHub issue forms and comments, a web form, the CLI and agents | Reviewed-by's ledger and workflows |
| 8 | **signing and federation** (optional) | 4 | signs S3 records; nodes exchange signed records keyed by meaning hash | trust-cli, trust-server and `FEDERATION.md`, generalized from certificates to every record kind |
| 9 | **evidence generators** (agents and tools) | 5 | proposing and checking examples and non-examples, disproof attempts, blind re-definition, mutation of specifications, value checks against LMFDB, DLMF and OEIS. They write S3 records, and **pull requests** that add examples or `@[specifies]` to the library | `plausible`; TauCetiReview-style agents; mostly new |
| 10 | **views**, which only read S2 and S3 | 3 | static site (claims, evidence cards, a math-language layer with a conventions panel); graph explorer across libraries; review workspace; editor extension; pull-request bot and policy gate; machine interface (CLI and MCP) for agents; dashboard | Referee's site, trust-web, the Reviewed-by page |

Pieces 1 and 2 are the only Lean code that has to match a toolchain. Everything from 6 onwards can
be written in any language and upgraded independently.

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
| MeaningGraph | the dependency engine inside the extractor, with its notions of dependency made explicit |
| semantic_hash | the hashing, with a header on its output and one pinned variant used for keys |
| trust's export and index | the basis of the S2 format and of the extractor's writer |
| Referee `collect` | extractor fields. Its `build-site` becomes the static site; its diff and provenance logic become the evidence core |
| ChallengeGen | standalone files, challenges, and the independent list that the self-checks compare against |
| Characterization, JunkValues | the annotation packages, with new kinds |
| trust-web | the explorer |
| trust-cli, trust-server | signing and federation, for every kind of record |
| Reviewed-by | the GitHub intake, the ledger conventions of S3, and write-back into docstrings as one more view |
| Comparator | certification of challenges and claims |
| aftk | stays a separate query and diagnostics tool on live environments. The dataset replaces its role as a source of dependencies |
| TauCetiReview | one agent that uses the machine interface |

---

## 9. Open questions to settle early

1. **One dependency engine, or several?** Suggested: one engine for the dataset, with the flat
   printer kept as an independent check (dependency-testing.md).
2. **Hash migrations.** When semantic_hash changes, every key changes. Records need both old and new
   hashes during a transition, and a migration that maps them through names at a commit.
3. **Extraction at Mathlib scale.** Extract incrementally, per module, caching by `.olean` hash.
4. **Write policy for evidence.** Who may write, whether AI output is rate-limited, how
   disagreements are shown. Suggested: decided per project.
5. **The math-language layer.** Generate it from structured data where possible; label AI
   paraphrase, and always show the formal text beside it.
6. **Where state lives.** Static wherever possible; a service only for the workspace, identity and
   federation.

---

## 10. Suggested phases

1. **The specifications, and extractor v1.** Write S1 to S3 with conformance vectors. Merge Referee's
   `collect`, trust's export and the hashing into one extractor with self-checks. Port the static
   site and the explorer to read S2.
2. **Evidence.** The store and its intake (GitHub and CLI), the evidence core, and the pull-request
   bot. Migrate Reviewed-by's ledgers, Referee's audit exports and trust's marks into S3.
3. **Richer evidence.** The new `@[specifies]` kinds and domain annotations, the analyzers, evidence
   cards in the views, and the review workspace.
4. **Scale and automation.** Generators and agents, challenges certified by Comparator, signing and
   federation, the editor extension, and the dashboard.
