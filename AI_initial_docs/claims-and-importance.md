# Claims and importance: guiding readers to what matters

Snapshot of 2026-09-25. Which declarations of a formalization should a reader look at first, and how
can tools find them and steer readers there, in different contexts: a single formalization
registered with Palomar, a community library like Tau Ceti, a foundational library like Mathlib?
This note lists the signals available, what each context needs, the ways to guide a reader, and the
indicators worth adding.

It is a companion to [suite-design.md](suite-design.md), [reviews.md](reviews.md),
[interfaces-by-audience.md](interfaces-by-audience.md) (personas P1 to P9) and
[trusting-definitions.md](trusting-definitions.md) (failure modes F1 to F9). The Mathlib figures
were measured at Mathlib commit `217ba069a5` (2026-09-10).

---

## 1. Three kinds of importance, and attention

Three different things get called "important":

1. **Claims:** the results a project puts forward as its point. Someone declares them, and there are
   few of them.
2. **Load-bearing definitions:** what the claims' statements rest on. These can be derived from the
   claims, as their statement closure.
3. **Landmarks:** declarations that correspond to known mathematics, such as named theorems or
   textbook notions. Their importance comes from outside the project.

A fourth question, **where attention is needed**, is about risk rather than importance. Guidance
combines all four, weighted differently in each context.

### Don't derive claims from the shape of the graph

Referee's `WEBSITE-DESIGN.md` records trying "the claims are the declarations nothing else uses".
The rule was wrong in both directions:
- a headline theorem reused by one corollary stopped counting;
- a lemma proved during an API build-out and never used again started counting.

On LeanMachineLearning, whose authors use `theorem` for results and `lemma` for steps, the rule
dropped 6 of the 11 regret-bound theorems that are the point of the library. Derived measures work
for **definitions**, as what claims rest on. **Claims have to be declared or curated.**

---

## 2. Available signals

| source | signal | exists today |
|---|---|---|
| **declared in the code** | `theorem` versus `lemma` | Referee's Theorems page. It works where a project uses the keywords deliberately: 11 of 698 declarations in LeanMachineLearning are theorems |
| | "Main definitions" and "Main results" sections in module docstrings | a Mathlib convention: **1,466** of Mathlib's 8,516 modules have a "Main definitions" section, and **1,296** a "Main results" (or "statements", or "theorems") section. Parseable, and read by no tool |
| | cross-reference tags (`@[stacks]`, `@[kerodon]`, `@[wikidata]`, `@[lmfdb]`, `@[pibase]`, `@[dlmf]`) | Mathlib: marks a declaration as matching a literature item |
| **declared beside the code** | `formalization.yaml`: `main_results`, literature dependencies, scope | Palomar registry; Referee's Claims page |
| | Comparator configs | certified claims |
| | blueprints: top-level nodes and their `\lean{}` links | projects using leanblueprint |
| | roadmap target signatures (`Suggested.lean`), and `STATUS.md` named results and notable definitions | Tau Ceti. **Humans declare the targets before the code exists** |
| | bot announcements | Voyager on Zulip, collected by Reviewed-by |
| **curated lists** | Mathlib's `docs/100.yaml` (Freek Wiedijk's 100 theorems; 78 with a declaration), `docs/1000.yaml` (the 1000+ theorems project, keyed by Wikidata; 216 with a declaration), `docs/undergrad.yaml` (about 560 undergraduate topics mapped to declarations), `docs/overview.yaml` (about 510) | shown on the Mathlib website only |
| **derived from the graph** | membership in claims' statement closures; how many claims rest on a definition; use in statements rather than proofs; dependency footprint | Referee: claims-only builds (about 2% of a library), ranking theorems by footprint, ranking unspecified definitions by use |
| **use across the ecosystem** | use by downstream packages; mentions in papers; documentation page views | nothing |
| **risk** (attention, not importance) | unreviewed or stale; written by AI; recently changed; junk-value findings; no examples; disagreements; problems or questions left open | parts in Referee, trust and Reviewed-by |

---

## 3. By context

| context | where claims come from | what is load-bearing | main reader | how to guide them |
|---|---|---|---|---|
| **single formalization** (a paper, registered with Palomar) | `main_results`, Comparator configs, the paper's theorem numbers, the blueprint's top nodes. Few and explicit | the statement closure of the claims: about 2% of the library | a referee (P6) | a claims-only site; each claim next to the paper's statement, with the gaps in scope and generality; the closure in reading order; literature dependencies made visible |
| **community or AI-written library** (Tau Ceti) | roadmap targets, named results, notable definitions. The code is written by AI, so **human declarations of importance are the anchor** | the closures of the targets | domain experts (P1), agents, maintainers | a roadmap view (target → the declaration that realizes it → its review status); a feed of newly named results; the queue weighted by roadmap |
| **foundational library** (Mathlib) | no claims at library level: importance is **relative to the reader** | from the point of view of a downstream project, what its claims rest on | Mathlib reviewers; downstream users | landmarks (curated lists, cross-references, "Main" sections) as a table of contents; "the Mathlib definitions *your* claims rest on"; for Mathlib reviewers, definitions ranked by use in statements across Mathlib and downstream |
| **downstream project** (built on Mathlib or LeanMachineLearning) | its own claims | its own closure, plus the part of the upstream closure it touches | its referee, its users | trust in upstream is graded: reviews of the upstream definitions it touches, reused by meaning hash (reviews.md §5) |
| **collections of statements** (open-conjecture repositories, benchmarks) | every statement is a claim, and there are no proofs | the statements' closures | anyone using the statements | reviewing *is* the product: reports of mis-formalized statements, organized by category |

---

## 4. Ways to guide a reader

- **Entry points:** a claims-first landing page, a named-results tab, a roadmap view, and a landmark
  index for explorers.
- **Reading paths:** from a claim down its statement closure, in dependency order, like Referee's
  "everything it rests on" stack. Definitions come first for readers who want to build up, and the
  claim first for readers who want to drill down.
- **Progressive disclosure:** lemmas, instances and internal declarations are hidden by default.
- **Importance relative to the reader:** "for your claims", or "for this roadmap".
- **The review queue:** importance × risk × missing evidence (reviews.md §5).
- **Feeds:** newly claimed or named results, and reviews that just went stale.

---

## 5. Indicators and mechanisms to add

1. **An in-code `@[claim]` attribute,** built on the generic extension (suite-design.md §3.2),
   carrying a reference: paper and theorem number, or a Wikidata item.
   - Unlike `formalization.yaml`, the name can't drift: a rename that breaks it is a build error.
   - It is exported as a facet, and `formalization.yaml` can be generated from it.
   - A companion **`@[landmark]`** attribute would do the same for definitions, unless the Wikidata
     tag already covers it.
2. **Read the "Main definitions" and "Main results" sections of module docstrings** into a facet.
   About 1,300 to 1,500 Mathlib modules already provide them, and no tool reads them.
3. **Import Mathlib's curated lists as facets,** and treat each mapping from a topic to a
   declaration as a **correspondence to review**. "Is `Submodule.completeLattice` really 'sum of
   subspaces'?" is exactly the kind of question reviews.md §1 lists among the subjects of reviews.
4. **Derived importance facets for definitions:** how many claims' statement closures contain it;
   its use in statements across Mathlib and downstream packages; whether it is a landmark.
5. **Nominations as S3 records.** Reviewed-by's `Named:` lines are already this. A community can
   propose something as important, and where that disagrees with the author's list, the
   disagreement is worth showing.
6. **The gap for each claim:** its informal counterpart, and the difference in scope and generality
   (F9 at the level of claims). `status.scope` in `formalization.yaml` is the free-text version
   today.

**Since this snapshot** (2026-09-26; see [status.md](status.md)):
- **`@[claim]`** exists, with an optional reference
  ([TrustAnnotations](https://github.com/LeanTrustBuilders/annotations)), and is exported as a
  facet.
- **Claims are read** from `@[claim]`, from `formalization.yaml` and from Comparator configs, by the
  Referee-style site. It builds claims-only sites (53 declarations of 815 for one paper
  formalization) and a page per claim.
- **Nominations** are S3 `named` records in the Tau Ceti pilot.
- **Not built:** `@[landmark]`, reading "Main" sections, Mathlib's curated lists, and derived
  importance facets.
