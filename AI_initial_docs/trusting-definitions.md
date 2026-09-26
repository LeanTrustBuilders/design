# Gaining trust in a definition

Snapshot of 2026-09-25. What can make us believe that a Lean definition means what it is meant to
mean? This note lists the kinds of evidence available, what each can and cannot show, which tools
exist for each, and which could be built. It is a companion to
[review-tools-comparison.md](review-tools-comparison.md) and
[dependency-testing.md](dependency-testing.md). The Mathlib facts quoted were checked at Mathlib
commit `217ba069a5` (2026-09-10).

---

## 1. The problem

The kernel checks proofs. Nothing checks definitions. A theorem means exactly what the definitions
in its statement mean, so trusting a formalized result comes down to trusting those definitions and
everything they rest on (their statement closure; see dependency-testing.md). Comparator's README
says the same about the definitions a challenge leaves open: they "can be gamed without additional
oversight".

**Trusting a definition** means believing that it denotes the intended mathematical object, up to
the intended notion of sameness, in every case where the statements using it apply. The intended
object is almost always given informally: a textbook, a paper, a database entry. So every kind of
evidence does one of two things:

- it **compares** the definition with that intent: reading it, checking values, recovering known
  results; or
- it **constrains** the definition by theorems, so that fewer wrong definitions remain possible:
  characterizations, specifications, examples.

---

## 2. How a definition can be wrong

Each kind of evidence in section 3 catches some of these failures and misses others. The matrix in
section 4 summarizes which.

| | failure | what goes wrong | example |
|---|---|---|---|
| F1 | **different object** | the formula or construction is simply not the intended one | ε and δ quantified in the wrong order, which defines a different notion than continuity |
| F2 | **different convention** | a close relative of the intended object: another normalization, indexing, base or sign | a Fourier transform with `e^{-ixξ}` instead of `e^{-2πixξ}`; `Finset.range n` is `{0, …, n-1}`; in Mathlib the norm on `E × F` is the max norm (`Prod.norm_def : ‖x‖ = max ‖x.1‖ ‖x.2‖`), so a definition using the norm on `ℝ × ℝ` is not the Euclidean distance |
| F3 | **different edge cases** | textbooks exclude or special-case degenerate inputs; a formal definition has to decide | Mathlib's `IsConnected s` requires `s.Nonempty`; a field is nontrivial (`DivisionRing` extends `Nontrivial`); `Nat.Prime 1` is false. A definition that decides otherwise than the source changes every theorem at the boundary |
| F4 | **junk values** | a total function returns a default value outside its intended domain, and a definition built on it silently means something else | `x / 0 = 0`; `√x = 0` for `x ≤ 0` (`Real.sqrt_eq_zero_of_nonpos`); `Real.log 0 = 0`; `∫ f = 0` when `f` is not integrable (`integral_undef`); `deriv f x = 0` when `f` is not differentiable at `x` |
| F5 | **vacuous or trivial** | a predicate no object satisfies, or every object satisfies; a structure or class with contradictory fields | every theorem assuming such a predicate is vacuous, or says nothing |
| F6 | **arbitrary choice** | the definition picks a witness with `Classical.choose`, while the intended object is canonical | a quantity defined through a chosen basis, with no proof that it doesn't depend on the basis |
| F7 | **wrong thing underneath** | a definition in the closure, or an instance found by typeclass search, is not the intended one | F1 to F6 one level down; or the norm example of F2, where the instance is found silently |
| F8 | **drift** | the definition was right when checked, and it or something below it has changed since | Referee's "indirect invalidation": the statement reads the same, but a definition it uses has moved |
| F9 | **insufficient generality** | the right definition, but in a narrower setting than the source: concrete types where the source has a class, or assumptions the notion doesn't need | defined on `ℝⁿ` (`EuclideanSpace ℝ (Fin n)`) where the source defines it on any Banach space; `ℝ`-valued where it should be over any `RCLike 𝕜`; a `[Fintype ι]` or finite-dimensionality assumption the notion doesn't need |

F9 differs in kind from F1 to F8: the definition is right wherever it is defined. What goes wrong
is what it can be applied to.
- **Theorems about it are weaker than the source's.** A result proved for `ℝⁿ` doesn't cover the
  Banach spaces the paper claims.
- **Its proofs may lean on the special setting,** such as coordinates, an inner product, or compact
  closed balls. Generalizing later can then be more than a change of signature.
- **Whoever needs the general version writes a second definition,** and the two must then be
  reconciled (section 3.7).

The same failure exists for statements: a theorem proved for `ℝⁿ` that the source states for
Banach spaces. Referee's Claims page shows `status.scope` from `formalization.yaml`, which is where
an author can declare such a restriction.

---

## 3. Kinds of evidence

Each kind below gives: what it is, which failures it catches, what it cannot show, what exists
today, and what could be built. Families A and C are judgements and analyses; family B is theorems,
which the kernel checks; family D is context.

### A. Judgement

#### 3.1 Reading the definition and what it rests on

**What it is.** A person reads the definition, compares it with the source, and reads what it rests
on.

**Catches.** F1 reliably. F2 and F3 when the reader knows the source's conventions. F7 if the
closure is read, not just the definition. F9 when the reader compares the signature with the
source's setting; generality is visible in the signature.

**Cannot show.** Subtle junk values and vacuity (F4, F5) are easy to miss. It doesn't scale, and it
is only as good as the reader.

**Exists.**
- Referee: statement anatomy with hovers, per-declaration dependency graphs, coverage over the
  closure, and self-contained extracted files that open in the web editor.
- trust-web: the graph in both directions, with definition bodies shown on hover.

**Could be built.**
- **Statement unfolding:** show a definition expanded down to library basics, to a chosen depth
  (Referee's `PROPOSED-TOOLS.md` §6).
- **Resolved instances:** show which instances a definition actually uses, such as which norm,
  topology or order. The flat printer already writes them out explicitly.
- **Informal paraphrase:** an AI-generated paraphrase shown next to the source's own definition,
  labelled as machine-generated. It is weak evidence, but cheap, and mismatches are worth a look.

#### 3.2 Reviews by others

**What it is.** Recorded judgements from other readers, human or AI.

**What makes a review worth more:** it was made independently, by someone with the relevant
expertise; it says what was checked; it covers the closure; and it is still current.

**Catches.** The same failures as reading, spread over more readers. A review tied to a hash also
detects F8: it goes stale when the definition changes.

**Exists.**
- Referee: private verdicts, stamped with the proof-irrelevant hash.
- trust: marks, and signed certificates that federate between servers.
- Reviewed-by: marks with GitHub identity, AI marks kept separate and required to give evidence,
  and public problem reports.
- TauCetiReview: verdicts from AI agents, per pull request.

**Could be built.**
- **Structured reviews:** a review records which failure modes of section 2 were checked. It can
  then say "conventions and edge cases checked, junk values not", instead of a bare approval.
- **Closure-aware reviews:** a review of a definition counts fully only when what it rests on is
  reviewed too. Referee's coverage, applied to shared reviews.
- **Disagreement reports:** the definitions on which two reviewers reached different verdicts
  (Referee's `TRUST-GAPS.md` §5).

### B. Theorems about the definition

The kernel checks these, and they keep being checked at every build: they don't go stale, they
break. Their statements still have to be read.

#### 3.3 Characterization

**What it is.** A property `P` characterizes `d` up to a relation `R` when two theorems hold:
*existence*, `d` satisfies `P`; and *uniqueness*, anything that satisfies `P` is `R`-related to
`d`. Trust then moves from the construction of `d` to `P` and `R`. This matters most when `P` is
the textbook definition and the construction is technical, as with conditional expectation or the
stochastic integral.

**Catches.**
- F1, and F2 when `P` is stated as in the source.
- F6: a characterized object is canonical up to `R`.
- F5: existence shows that `P` is satisfiable.
- Much of F3, because `P` decides the edge cases and `P` is closer to the textbook.

**Cannot show.** Whether `P` itself is the intended property, or whether `R` is as strong as
expected ("almost everywhere" is not "everywhere"). A characterization is only as good as the
reading of `P` and `R`.

**Exists.**
- `@[characterization]` from the Characterization package, which checks the shapes of the three
  parts.
- Referee's "via property" view of a characterized definition's graph.
- trust's `characterize` marks and its trusted-mode cut.
- Many uniqueness theorems already in Mathlib, such as `condExp`'s.

**Could be built.**
- **Characterization discovery:** find pairs of theorems in Mathlib that already have the
  existence and uniqueness shapes, and propose annotations.
- **Link `P` to a source**, so that a reader compares `P`, not the construction, with the
  textbook.

#### 3.4 Specification properties

**What it is.** Theorems the author offers as evidence that the definition is the intended one:
agreement with the textbook formula in a special case, basic identities, behaviour under
operations. They pin the definition down partly.

**Catches.** Part of F1 and F2. How much is unknown without measuring it (section 3.11).

**Exists.**
- `@[specifies]` from the Characterization package, recorded in the source and checked for its
  target.
- Referee's Specifications page, which ranks definitions that have no specification by how much
  uses them.
- Reviewed-by's "key results" (`Test:` lines) and trust's `characterize` marks record the same
  kind of link outside the source, where nothing checks it.

**Could be built.**
- **One vocabulary for all tools:** Reviewed-by and trust read these links from `@[specifies]` in
  the source instead of keeping their own files.
- **Specification strength**, measured by mutation (section 3.11).

#### 3.5 Examples, non-examples and values

**What it is.** Small theorems about concrete cases:
- `example : IsFoo a`, a positive example;
- `example : ¬ IsFoo b`, a negative example;
- `example : f 3 = 7`, a value;
- the degenerate inputs: `f 0`, the empty set, the trivial ring.

**Catches.** This is the cheapest strong evidence there is.
- F2: a different normalization shows up in values, for instance the transform of a Gaussian.
- F3: the degenerate cases are checked directly.
- F5: a positive example proves the predicate is satisfiable; a negative example proves it doesn't
  hold for everything.
- Often F1, and some F4 when an example sits at a junk point.

**Exists.**
- Tau Ceti's `example`s, which Reviewed-by counts as unit tests.
- Reviewed-by's suggested tests, as free-text issues.
- `decide`, `norm_num` and `#eval` to prove or compute values.

**Could be built.**
- **Recorded examples:** new kinds of `@[specifies]`, such as `@[specifies d example]`,
  `@[specifies d nonexample]` and `@[specifies d value]`. They are checked like `@[specifies]` and
  read by every tool. Reviewed-by's unit-test discovery then becomes exact instead of relying on
  regexes.
- **Missing-examples report:** predicates with no positive or no negative example, and structures
  and classes with no instance, ranked by how much uses them.
- **Values from databases:** for definitions tagged `@[lmfdb]` or `@[dlmf]`, or matching an OEIS
  sequence, generate goals `f n = v` from the database and try to close them automatically. A goal
  that is proved false is a finding.

#### 3.6 Known results recovered

**What it is.** Proving a known theorem about the definition, such as `ζ(2) = π²/6` for a zeta
function. If the definition were wrong, the known result would usually fail. This is arguably the
main source of trust in Mathlib's definitions: they have a large API that works.

**Catches.** F1 reliably; F2 and F3 often; F7 partly, since the result exercises the closure. It
also exposes F9: a known result can only be stated in the definition's own setting, so a restriction
shows as soon as someone tries to state the source's version.

**Cannot show.** Junk values can make a "known result" hold vacuously or by an unintended route.
The statement has to be read and its hypotheses checked (sections 3.8 and 3.9).

**Exists.** Tau Ceti roadmaps' named results; Reviewed-by's key results; Referee's Claims page.

**Could be built.** **Results credited to definitions:** each named result counts as evidence for
the definitions its statement mentions. A definition's evidence card (section 5) then lists the
known results proved about it.

#### 3.7 Agreement with an independent definition

**What it is.** Two definitions of the same object, built independently, proved equal or
equivalent up to `R`. Examples: exponential as a power series and as the limit of `(1 + x/n)^n`;
the determinant by permutations and by cofactor expansion; a Tau Ceti definition and Mathlib's.
Reviewed-by's key results already include agreements with a Mathlib notion.

**Catches.** An error would have to appear identically in both constructions, so this is the
strongest evidence after a characterization: F1, F2, F3 and F6. It also exposes F9 when the second
definition is more general and is proved to agree on the special case, such as a Banach-space
definition that agrees with the `ℝⁿ` one on `ℝⁿ`.

**Cannot show.** A misconception both authors share, or a second definition that was really
copied from the first.

**Exists.** Proved equivalences throughout Mathlib, but not recorded as such.

**Could be built.**
- **Recorded agreement:** a kind of `@[specifies]` marking a theorem as an agreement between two
  definitions, so tools can list the definitions that have one construction and nothing to compare
  it with.
- **Blind re-definition by AI:** an agent defines the notion from the source without seeing the
  library's definition, then tries to prove the two equal. Independence is what gives this its
  value, and every disagreement is a finding.

### C. Analyses and attacks

#### 3.8 Junk values and domains

**What it is.** Find where a definition or statement relies on a default value outside the
intended domain.

**Catches.** F4 directly; the vacuity risk it creates in statements (F5).

**Exists.** JunkValues: the `@[junk_value]` attribute, a linter, `#junk_check`, and a catalogue of
Mathlib's junk values. It separates findings in statements, a vacuity risk, from findings in
bodies, a meaning risk.

**Could be built.**
- **Domain annotations:** declare the intended domain of a total function, such as `0 < x` for
  `Real.log`. This could be derived from junk-value rules, which state its complement.
- **Junk findings shown by default** on the statements of claims, in every review view.
- **Unguarded hypotheses:** flag theorems whose hypotheses don't rule out a junk value their
  statement uses.

#### 3.9 Vacuity and triviality

**What it is.** Check that predicates, structures and hypotheses are neither impossible nor
automatic.

**Catches.** F5; part of F3.

**Exists.** The `unusedArguments` linter for unused hypotheses. Referee's `PROPOSED-TOOLS.md`
proposes an inhabitation check (§4) and a triviality check (§7); neither is built.

**Could be built.**
- **Inhabitation:** for each structure, class and predicate in a claim's closure, look for an
  instance or witness, with library search, automation or an agent.
- **Triviality:** try to prove a predicate for all objects, and try to prove its negation.
- **Consistent hypotheses:** for each claim, try to derive `False` from its hypotheses. Success is
  a serious finding.

#### 3.10 Counterexamples, disproof attempts and challenges

**What it is.** Try to break the expected properties of a definition.

**Catches.** F3 especially; part of F1, F2 and F5.

**Exists.**
- `plausible`, property-based testing, which Mathlib depends on.
- Reviewed-by's suggested tests: free text, left open until the property is proved.
- Comparator, which certifies that a solution proves exactly the statement of a challenge.

**Could be built.**
- **Formal challenges:** a reviewer, human or AI, writes `theorem challenge : P d := sorry`.
  Contributors prove it or prove `¬ P d`, and either outcome is recorded next to the reviews.
  ChallengeGen can produce a self-contained challenge file, and Comparator can certify the answer.
- **AI adversaries:** for each definition, generate the properties the source implies, and attempt
  both a proof and a disproof of each.

#### 3.11 Measuring specification strength by mutation (proposal)

**What it is.** Change the definition slightly, in a scratch copy: a constant, a normalization,
the order of two arguments, a `<` into a `≤`. Then check whether the specification,
characterization and example theorems still hold for the changed version. A variant for which they
all still hold is one the theorem evidence cannot tell apart from the intended definition, and that
is exactly the gap a reader must fill.

This is mutation testing from software engineering, applied to specifications. It measures how much
of family B's evidence (3.3 to 3.7) actually rules out F1 to F3.

**Cost.** A proof may break for reasons unrelated to whether the property still holds. So a
property should be re-established on each variant rather than by replaying the proof:
- for computable definitions, by evaluating it on samples;
- otherwise, by automation or an agent attempting both a proof and a disproof.

**Exists.** Nothing.

#### 3.12 Choice, instance and generality reports (proposal)

**Choice.** List the definitions whose *data* depends on `Classical.choose` or `Classical.choice`,
as opposed to their proofs. MeaningGraph's `dataDeps` already separates the two. For each one, ask
for a theorem that the result doesn't depend on the choice, or for a characterization, which
implies it. This catches F6.

**Instances.** List the instances a definition's statement actually uses: the norm, topology,
order, measure. Flag those where the library's default differs from a common convention, as with
the max norm on products. This catches F7 and part of F2.

**Generality.** List the concrete types and the assumptions in a definition's signature, and flag
two things:
- concrete types where a general class exists: `ℝ` where an `RCLike 𝕜` would do,
  `EuclideanSpace ℝ (Fin n)` where a normed space would do, `Fin n` where a `Fintype ι` would do;
- assumptions the definition never uses.

Then **try to generalize automatically**: replace a concrete type by a variable with the matching
class, or drop an assumption, and check whether the definition and its theorems still elaborate.
An automated rewrite or an agent can do this; each success is a finding. Where the source's
setting is recorded, compare it with the signature. This catches F9.

**Exists.**
- Referee's statement anatomy shows the typeclass structure on each object, and the flat printer
  writes instances out.
- For unused assumptions only: the `unusedArguments` linter, and Lean's
  `linter.unusedSectionVars`, which covers section variables in theorem bodies.

Nothing reports on choices, on instances, or on generality beyond unused assumptions.

### D. Context

#### 3.13 Links to informal sources

**What it is.** A reference that states the intent: a docstring citation; Mathlib's cross-reference
attributes (`@[stacks]`, `@[kerodon]`, `@[wikidata]`, `@[lmfdb]`, `@[pibase]`, `@[dlmf]`); a
blueprint's `\lean{…}` link; `literature_dependencies` in `formalization.yaml`.

Links are not evidence by themselves. They make every comparison possible and cheap, because they
say what all other evidence is measured against.

**Could be built.**
- **Missing sources:** report the definitions in a claim's closure that cite no source.
- **Checks driven by the source:** values from databases (section 3.5), and an AI comparison
  between the definition at the tag and the Lean definition, labelled as machine-generated.

#### 3.14 History and use

**What it is.** Circumstantial evidence:
- how long the meaning has been stable (Referee's provenance: "meaning unchanged since v0.1");
- how widely the definition is used, and how many known results rest on it;
- the review process it went through;
- whether problem reports about it are open or closed.

**Catches.** Weakly, F1; it helps notice F8.

**Cannot show.** Much on its own, and it can mislead: Mathlib's junk-value conventions are stable
and widely used, and still surprise readers.

**Exists.** Referee's provenance and use counts; Reviewed-by's problem reports.

---

## 4. Which evidence catches which failure

● catches it reliably, ○ partly, blank not at all. These are judgement calls; section 3 gives the
reasons.

| evidence | F1 object | F2 convention | F3 edge cases | F4 junk | F5 vacuity | F6 choice | F7 underneath | F8 drift | F9 generality |
|---|---|---|---|---|---|---|---|---|---|
| 3.1 reading | ● | ○ | ○ | ○ | ○ | ○ | ○ | | ● |
| 3.2 reviews by others | ● | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| 3.3 characterization | ● | ● | ○ | ○ | ● | ● | ○ | ● | |
| 3.4 specification | ○ | ○ | ○ | | ○ | | | ● | |
| 3.5 examples and values | ○ | ● | ● | ○ | ● | | | ● | |
| 3.6 known results | ● | ○ | ○ | | ○ | | ○ | ● | ○ |
| 3.7 independent definition | ● | ● | ● | ○ | | ● | | ● | ○ |
| 3.8 junk values | | | | ● | ○ | | | | |
| 3.9 vacuity checks | | | ○ | | ● | | | | |
| 3.10 disproof and challenges | ○ | ○ | ● | | ○ | | | | |
| 3.12 choice, instances, generality | | ○ | | | | ● | ● | | ● |
| 3.13 sources | | ○ | ○ | | | | | | ○ |
| 3.14 history and use | ○ | | | | | | | ○ | |

Section 3.11 is not a row: it measures how much of family B's evidence really rules out F1 to F3.
In the F8 column, theorems are marked ● because a change that breaks them breaks the build. A review
only notices a change if it is tied to a hash, and then someone has to review again.

---

## 5. Recording evidence

What makes evidence usable by more than one tool and one reader:

1. **Tied to a version.** Evidence must name the version of the definition it is about. The
   natural key is the proof-irrelevant semantic hash, which also changes when anything the
   definition rests on changes. Theorems in the source need no key: they are re-checked at every
   build.
2. **Labelled by how it is backed:** checked by the kernel, computed by a tool, asserted by a
   person, or asserted by an AI.
3. **Attributed:** who or what produced it, and when.
4. **Linked to the definition it supports.** Today `@[specifies]` and `@[characterization]` are the
   only links that live in the source and are checked. Reviewed-by's `Test:` lines and trust's
   `characterize` marks record the same kind of link outside the source, unchecked.
5. **Combined over the closure.** Evidence for a definition is complete only when what it rests on
   has evidence too: Referee's "coverage", applied to every kind of evidence.

**Proposal: an evidence card per definition.** One view gathering everything above:

| field | from |
|---|---|
| characterization, with `R` in full | `@[characterization]` |
| specification theorems, and their measured strength | `@[specifies]`; mutation (3.11) |
| examples, non-examples and values | proposed `@[specifies]` kinds; `example`s |
| known results proved about it | named results, claims |
| agreements with independent definitions | proposed `@[specifies]` kind |
| junk-value findings; vacuity checks | JunkValues; proposed checks (3.9) |
| choices and instances used; generality: concrete types, unused assumptions, successful generalizations | proposed reports (3.12) |
| informal sources | docstrings, cross-reference attributes, blueprints |
| reviews: people and AI, current or stale; open problems | Referee, trust, Reviewed-by |
| meaning stable since; use count | Referee provenance |
| coverage: how much of the closure has evidence | the statement closure |

The card is a view. Its data would be shared by Referee, trust and Reviewed-by, which can each
render it.

---

## 6. Recommendations

In a suggested order: the first ones are cheap and unblock the rest.

1. **One vocabulary for theorem evidence, in the source.** Extend `@[specifies]` with kinds:
   property, example, non-example, value, agreement, known result. Every tool then reads the same
   checked links from the code, and Reviewed-by's `Test:` lines and trust's `characterize` marks
   become readable from it.
2. **Examples and non-examples for everything in a claim's closure.** Report every predicate,
   structure and class without a positive or a negative example, ranked by use. This is the
   cheapest evidence against F2, F3 and F5.
3. **The evidence card, with coverage over the closure,** as a data format shared by the three
   tools, and as a view in each.
4. **Junk-value findings on claims' statements, by default.** JunkValues exists; this only means
   showing its output in review views.
5. **Formal challenges.** Challenges are Lean statements, answered by a proof or a disproof,
   certified with ChallengeGen and Comparator, and stored with the reviews.
6. **Vacuity checks on claims:** inhabitation of what the closure assumes, and consistency of each
   claim's hypotheses.
7. **Mutation testing of specifications,** starting with computable definitions, where a changed
   version can be evaluated instead of proved.
8. **Blind re-definition by AI,** then equality attempts. This is the cheapest source of
   independent definitions.
9. **Choice, instance and generality reports** for the definitions in claims' closures, including
   automatic generalization attempts.
10. **Sources for claim-closure definitions,** and value checks against databases where a
    cross-reference exists.

**Since this snapshot** (2026-09-26; see [status.md](status.md)):
- **Recommendation 1** has begun. `@[specifies]`, `@[example_of]`, `@[nonexample_of]` and
  `@[characterization]` live in one package
  ([TrustAnnotations](https://github.com/LeanTrustBuilders/annotations)) and are exported to
  datasets; the `value`, `agreement` and `known result` kinds are not built.
- **Recommendation 3** has begun: a claim's page shows, for each declaration its statement rests
  on, the theorems Lean checks about it and its reviews, which failure modes were checked, and
  coverage.
- **The others are not started.**

In terms of the types of tool discussed in review-tools-comparison.md §9, plus the fifth type
proposed in discussion (tools that generate evidence by trying to break the code):
- **type 1, data in the code:** recommendation 1, the examples that recommendation 2 asks for, and
  the sources of recommendation 10;
- **type 2, analyses:** the report of recommendation 2, and recommendations 4 and 9;
- **type 3, views:** recommendation 3;
- **type 4, storage and handling:** recommendation 5;
- **type 5, attacks:** recommendations 6, 7 and 8, and the database value checks of
  recommendation 10.
