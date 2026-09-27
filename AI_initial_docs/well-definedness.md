# Well-definedness: junk values, choices, and objects defined up to a relation

Proposal of 2026-09-27. A textbook definition carries three obligations that Lean lets a formal
definition skip: to apply each operation inside its domain, to not depend on the choices it makes,
and to not depend on the representative it picks from a class. This note describes what goes wrong
when they are skipped, the options for detecting it, and a proposal. Authors declare what they
intend: where a definition applies, what it is determined up to, and which choices are meant to be
arbitrary. A new analyzer states each obligation against that intent, discharges what it can, and
shows what is left open. It extends
[trusting-definitions.md](trusting-definitions.md) (failure modes F4, F5 and F6; §3.3, §3.8 and
§3.12) and fills in component 3 of [suite-design.md](suite-design.md), the analyzers. The Mathlib
facts quoted were checked at Mathlib commit `065356127b` (2026-09-16), and the counts were taken on
the suite's dataset of that commit.

---

## 1. The problem

A mathematician who writes "let f(x) = log x / (x − 1)", "choose a basis, and let t be the trace
of the matrix", or "E[X | 𝒢] is the 𝒢-measurable function such that …" owes the reader three
things, usually silently:

| obligation | in a textbook | how Lean lets a definition skip it |
|---|---|---|
| **defined** | every operation is applied inside its domain | operations are total: outside the domain they return a value anyway, the *junk value* |
| **independent of choices** | "this does not depend on the basis chosen" | `Classical.choose` picks a witness and asks for nothing |
| **independent of representatives** | "this is well defined, since functions equal a.e. give the same …" | a representative can be taken (`Quotient.out`, an a.e.-defined function evaluated at a point) and used in ways the class does not determine |

None of these makes a theorem false: the kernel checks proofs, whatever the definitions are. Each
can make a definition denote something other than what was intended, or make a statement hold for a
reason the reader does not see. In the terms of trusting-definitions.md §2, these are:
- F4, junk values;
- F5, a statement that holds vacuously;
- F6, an arbitrary choice;
- through F7, everything built on such a definition.

The three have the same structure: an obligation that the textbook states and the formal definition
does not. So they can be handled the same way: **state the obligation, try to discharge it, show
what is left open**. What differs is the knowledge needed about each operation:
- its domain (§2);
- the specification of a choice (§3);
- which operations respect a relation (§4).

The obligations themselves come from intent, and intent is the author's to declare: nothing in the
term says where a definition is meant to apply, what it is meant to be determined up to, or whether
a choice is meant to be arbitrary. So the proposal is that authors declare these with attributes,
and the tools check each definition against its declaration. Where there is no declaration, the
tools show what they can derive, and a reviewer judges.

Section 5 puts the three together, and section 6 places them in the suite.

---

## 2. Junk values

### 2.1 Kinds

| kind | examples in Mathlib |
|---|---|
| **collapse** when a side condition fails | `∫` of a non-integrable function is 0 (`integral_undef`); `∑'` of a non-summable family is 0 (`tsum_eq_zero_of_not_summable`); `deriv f x = 0` where `f` is not differentiable (`deriv_zero_of_not_differentiableAt`); `condExp` of a non-integrable function is 0 (`condExp_of_not_integrable`); `Ring.inverse` of a non-unit is 0 (`Ring.inverse_non_unit`); `LinearMap.det` on a module with no finite basis is 1 (in its definition) |
| **a value at a point** | `x / 0 = 0`, `0⁻¹ = 0`, `Real.log 0 = 0`, `[].head! = default` |
| **truncation or clamping** into the codomain | subtraction in `ℕ` and `ℝ≥0∞`; `Int.toNat`; `Real.toNNReal`; `√x = 0` for `x ≤ 0` (`Real.sqrt_eq_zero_of_nonpos`); `arcsin x = π / 2` for `x ≥ 1` (`Real.arcsin_of_one_le`) |
| **extension by another formula** | `Real.log x = Real.log \|x\|` (`Real.log_abs`); `x ^ y = exp (log x * y) * cos (y * π)` for `x < 0` (`Real.rpow_def_of_neg`); branch cuts, such as `Complex.arg z ∈ (-π, π]` (`Complex.arg_mem_Ioc`). This kind overlaps F2, conventions |
| **0 for "infinite"** | `Nat.card`, `Set.ncard`, `Module.finrank`, `Subgroup.index` and `orderOf` in the infinite case (`Nat.card_eq_zero_of_infinite`, `Set.Infinite.ncard`, `Module.finrank_of_not_finite`, `Subgroup.index_eq_zero_iff_infinite`, `orderOf_eq_zero_iff`); `ENNReal.toReal ∞ = 0` |
| **suprema and infima that don't exist** | `sSup` of an unbounded or empty set of reals is 0 (`Real.sSup_of_not_bddAbove`, `Real.sSup_empty`); `sInf (∅ : Set ℕ) = 0` (`Nat.sInf_empty`) |
| **one notation, a different default per type** | `(0 : ℝ)⁻¹ = 0` but `(0 : ℝ≥0∞)⁻¹ = ∞`. So `ProbabilityTheory.cond μ s`, defined as `(μ s)⁻¹ • μ.restrict s`, is the zero measure when `μ s = 0`, through `∞ • 0 = 0` |
| **a chosen default** | the image of a nonzero measure under a map that is not a.e.-measurable is a Dirac mass at a chosen point, not 0 (`Measure.map_of_not_aemeasurable_of_ne_zero`). This is a junk value and a choice (§3) at once |

One operation can have several kinds: `Real.log` has a value at a point (`log 0 = 0`) and an
extension (negative arguments).

### 2.2 Intent and collision

Two properties matter more to a reader than the kind.

**Intent.**
- *Fallback:* nothing was meant, a value was needed (`integral_undef`).
- *Convenience:* chosen so that identities hold without side conditions, and the library relies on
  it. `add_div : (a + b) / c = a / c + b / c` needs no `c ≠ 0` because `x / 0 = 0`.
- *Convention:* the literature agrees: `0 ^ 0 = 1`, the empty sum is 0, `Nat.choose n k = 0` for
  `k > n`.
- *Encoding:* the value carries information: `orderOf x = 0 ↔ ¬IsOfFinOrder x`.

**Collision:** whether the default is also a genuine value.
- `deriv f x = 0` is also what a differentiable function gives at a critical point.
- `Nat.card α = 0` is also the cardinality of an empty type.
- `InformationTheory.klDiv μ ν` is `∞` when `μ` is not absolutely continuous with respect to `ν`,
  or when the log-likelihood ratio is not integrable: the cautious direction.

Only a default that collides can make a statement hold for no reason.

Both properties are judgments about an operation, not about each use of it, so they are recorded
once, in the declaration of the operation's domain (§2.5).

### 2.3 What exists: JunkValues

[JunkValues](https://github.com/LeanMachineLearning/exposition), which depends on Lean core only, is
a linter and a batch scan.

- **Rules.** A rule is an existing theorem of the shape `guards → lhs = rhs`, such as
  `integral_undef`. It is either tagged `@[junk_value]` in the source or listed in a catalogue of
  Mathlib's rules (22 entries).
  - The rule is kernel-checked, so the tool cannot claim a default that is not real.
  - For a value at a point, `generalizing` turns `div_zero : a / 0 = 0` into the pattern `?a / ?b`
    with the guard `?b = 0`.
- **Checks.** Each occurrence of a rule's left side is checked in the declaration's local context.
  A discharger (`fun_prop`, `norm_num`) tries to prove that the guard fails. The result is
  `guarded`, `unguarded` (a finding), or `triggered` (the guard is proved).
- **Findings** are split between statements, a vacuity risk, and bodies, a meaning risk.
- **Propagation:** `inheritRisk` marks a declaration whose direct dependencies have findings.
- **Discovery** proposes rules by their shape: 1417 candidates over Mathlib, for the catalogue's 22.

**Ideas worth keeping**, though not its code:
- facts that are checked;
- checks made in the real local context;
- a discharger named as a string, so that a tactic runs only if the project imports it;
- the split between statements and bodies;
- the lessons its README records: rules resolved per instance (subtraction in `ℝ` was once reported
  as truncating), and the bodies of recursive definitions found in the helpers the compiler
  generates.

**What is wrong with it:**

1. **A rule's guard is not the complement of the domain.** The guard covers every input where the
   value equals the default, including inputs where the default is the genuine value.
   - `Real.sqrt_eq_zero_of_nonpos` has the guard `x ≤ 0`, yet `√0 = 0` is the intended value.
   - `tsub_eq_zero_of_le` has the guard `a ≤ b`, yet `a - a = 0` is intended.

   So deriving a domain from rules, as trusting-definitions.md §3.8 proposes, gets the boundary
   wrong exactly where the default collides.
2. **Several kinds have no usable rule.**
   - An extension needs a theorem with a guard, which Mathlib often lacks:
     `Real.log_neg_eq_log : log (-x) = log x` matches only a literal `log (-x)`.
   - A chosen default has no theorem saying which value it is.
   - A value at a point needs `generalizing`, chosen by hand, since `Real.log_zero` and
     `Real.log_one` have the same shape.
3. **Rules cannot record intent.** That is why the shape cannot tell 1417 candidates from 22 rules.
4. **The unit of report is wrong.**
   - Definitions rarely take hypotheses, so every integral in a definition comes out `unguarded`:
     the finding tells the reader what they can already see.
   - In library lemmas, uses outside the domain are often deliberate (convenience).

   In its README's words, turning the linter on "produces a wall of true-but-unactioned findings".
5. **Propagation is one step deep, and yes or no.**

These come from its starting point, the value outside the domain, rather than from its
implementation. So this proposal starts a new tool instead of evolving JunkValues, and takes the
domain from the author (§2.5). JunkValues stays prior art.

### 2.4 Options

**Where knowledge about an operation comes from:**

| source | what it gives | checked? | what it covers |
|---|---|---|---|
| **the author's declaration** (an attribute, §2.5) | the intended domain, from the one person who knows it | its shape; then the definition is checked against it (§2.5) | the author's own definitions |
| **a catalogue's declaration**, with the same attribute in a separate module | the intended domains of operations the project does not own, such as Mathlib's | the same. As evidence of intent it is weaker: a third party's reading | as far as the catalogue goes |
| **domains read off specifications** | the hypothesis of a lemma that links the operation to its defining property: `Summable.hasSum`, `DifferentiableAt.hasDerivAt`, `tendsto_nhds_limUnder`, `Real.exp_log : 0 < x → exp (log x) = x`, `tsub_add_cancel_of_le : a ≤ b → b - a + a = b`, `Real.sq_sqrt : 0 ≤ x → √x ^ 2 = x` | that the operation behaves as specified on the domain. Which lemma counts as the specification is a judgment (`Real.log_nonneg` assumes `1 ≤ x`, which is not log's domain), the same judgment that `@[specifies]` asks for | every kind, with the right boundary. Good for proposing catalogue entries |
| **domains derived from a body** | a definition's domain, from the domains of what its body uses | mechanically | any definition. The domain is phrased in terms of the inner expressions (`log b ≠ 0`, not `b ∉ {-1, 0, 1}`) |
| **rules** (JunkValues) | the value outside the domain | yes | collapses, values at a point, truncations. Not extensions without a guarded theorem, nor chosen defaults. Boundaries come out wrong |
| **none: readers and agents** | judgment | only through what they produce | anything, unsystematically |

**How a use is checked:**

| method | what it answers | cost | limits |
|---|---|---|---|
| **names in the dataset** | a claim's conclusion uses `tsum`: does a hypothesis mention `Summable`? | Python in evidence-core, on today's dataset | useless for arithmetic: `hx : x ≠ 0` mentions only `Ne` |
| **each occurrence** (JunkValues) | is this occurrence's guard ruled out? | a Lean analyzer | the flood of findings of §2.3 |
| **well-definedness conditions**, as Event-B computes them (PVS's type-correctness conditions are the same idea) | build one condition per statement from the domains: WD(a / b) = WD(a) ∧ WD(b) ∧ b ≠ 0, and WD(P → Q) = WD(P) ∧ (P → WD(Q)). Discharge what the hypotheses give, and report the **residual** | a Lean analyzer | the same discharger problem. An operation with no declared domain is missed silently |
| **proof terms** | does the claim's proof apply `integral_undef`, `div_zero`…? Then its statement provably covers a junk case, and holds there by convention | the dataset's edges and a list of lemmas | precise, but misses a lot: a Mathlib lemma that uses the convention inside its own proof, such as `add_div`, is invisible |
| **perturbation** | would the proof survive a different default? This is the exact question | re-elaborating against altered definitions | research |
| **an instance at the boundary** | for example, "at `x = 0` the theorem reads `0 = 0`" | an agent picks the point, Lean checks the instance | picking the point |
| **an agent's review** | the F4 judgment, with a boundary instance or a well-definedness proof as checkable output | cheap | a record like any AI review |
| **checklist and catalogue** (today) | a reviewer's judgment | nothing to build | doesn't scale |

**How the results are shown:**
- a flag per occurrence;
- the residual per statement;
- the domain per definition;
- a restatement with the defining property, such as "either `f` is not differentiable at `x`, or
  `f′(x) = 0`";
- an instance at the boundary;
- items to tick in a review.

For authors, there could be a guideline for claims only: state them with the defining property
(`HasDerivAt`, `HasSum`) or with explicit domain hypotheses. It should not apply to library lemmas,
where Mathlib rightly drops hypotheses it does not need.

### 2.5 Proposal: domains declared by authors

**The attribute.** The author declares the intended domain on the definition itself, as a
proposition about its arguments under their own names:

```lean
@[domain (0 ≤ p ∧ 0 < q) "0 · log (0 / q) = 0, by convention"]
noncomputable def klTerm (p q : ℝ) : ℝ := p * Real.log (p / q)
```

No predicate is declared for it. A predicate written only to carry an annotation clutters the
library with a definition nobody is meant to use.
- **Arguments without names**, as in a definition by pattern matching, take a function of the
  explicit arguments: `@[domain (fun n => 0 < n)]`. The term goes in parentheses, since otherwise
  `q "…"` would be read as `q` applied to the note.
- **Definitions a project does not own** get their domain from a catalogue module, with the same
  attribute: `attribute [domain (0 < x)] Real.log`. The other attributes of TrustAnnotations refuse
  an imported declaration, because their entry would sit in a module that the declaration's readers
  never import. For a catalogue that is the point: the tools import it.
- **Storage.** A tool needs the elaborated proposition, and TrustAnnotations' entries hold only
  names and a string. So the attribute also stores the domain as a hidden predicate over the
  definition's arguments, `klTerm._domain`, the way Lean stores equation lemmas.
  - Its name is internal, so documentation, search and completion skip it, and so does
    MeaningGraph: it is not a node of the dataset.
  - Its body is exported under the module system.
  - The entry is anchored to it, which is what lets a catalogue record a domain for an imported
    definition.

The entry records the domain as written, the note, and its `source`: `"author"` when declared in
the definition's own module, `"catalogue"` otherwise, so a page can say who declared it. The payload
could later also carry the intent of the value outside the domain (fallback, convenience,
convention, encoding), whether it collides (§2.2), and the theorems that say what that value is
(`Real.log_zero`, `Real.log_neg_eq_log`).

**Built** on 2026-09-27, in TrustAnnotations (a3f48c6, on all three toolchains): `@[domain]`, the
hidden predicate, catalogue entries, and `domainEntries` to read them. It was tried on Mathlib with
`klTerm` and a catalogue entry for `Real.log`.

**Two obligations**, one for the definition and one for each use of it.

1. **Inside: under its declared domain, a definition never depends on a junk value.** For each
   occurrence `t = op args` in the body of `d`, where `op` has a declared domain:

       ∀ xs, d._domain xs → op._domain args ∨ ∀ c, body[t := c] = body

   Either the arguments are in `op`'s domain, or the value of `t` does not matter: replacing it by
   any `c` leaves the body unchanged. For `klTerm`:
   - `p / q` is inside the domain of division, since `0 < q`;
   - `log (p / q)` is outside log's domain at `p = 0`, but there `0 * c = 0` for every `c`, so the
     junk value is irrelevant. Mathlib's `Real.negMulLog x = -x * log x` works the same way at 0.

   A definition `log x / x` with the declared domain `0 ≤ x` fails at `x = 0`. There, `log 0 / 0`
   is a division by zero, and its value, 0, is a junk value taken inside the domain. That failure
   is an F4 finding, proved rather than suspected.
2. **Outside: every use stays inside the domain.** For each application `d args` in a claim, a
   specification theorem or another definition, the hypotheses in scope must imply
   `d._domain args`, or the value must be irrelevant as above. What is left is the statement's
   **residual**, which the claim page shows.

   The theorems cited for the value outside the domain give the boundary test: substitute the
   value, then try to close the statement from the hypotheses with `simp`, `norm_num` or
   `positivity`.
   - If it closes, the statement holds in that case for no reason.
   - If it doesn't, the statement excludes that case by itself: `∫ f ∂μ = 1` forces
     integrability.

A definition is checked once, against its own declaration, and its uses are checked against its
domain. That is what removes JunkValues' flood of findings: nothing is reported just because an
integral appears somewhere. The obligations are checked on claims, specification theorems and
definitions only. In other lemmas, uses outside the domain are often deliberate (convenience).

**Built** on 2026-09-27, for the outside obligation on statements:
[LeanTrustBuilders/well-defined](https://github.com/LeanTrustBuilders/well-defined) (Lean core and
TrustAnnotations), run by the extractor as `trust-extract welldefined` (0.8.0).
- **What is in scope** is read as Event-B reads well-definedness conditions:
  - hypotheses, for what follows them, including their parts (both sides of `∧`, the witness and
    property of `∃`);
  - the left side of `∧` on its right, and the left side of `∨` being false on its right;
  - the condition of `if` in its branches.

  A variable bound inside the statement has no hypothesis about it, and the obligation is
  reported "for every" such variable.
- **A domain that is a conjunction** gives one obligation per part. `condExp`'s domain,
  `∃ hm : m ≤ m₀, SigmaFinite (μ.trim hm) ∧ Integrable f μ`, is reported part by part, and
  `condExp_add` shows integrability, from `hf` and `hg`, but not `m ≤ m₀`.
- **Five outcomes:**
  - `discharged`;
  - `irrelevant`: the irrelevance test above, as `∀ c, F[c] ↔ F`, where `F` is the hypothesis or
    conclusion the use sits in, proved from the theorem's hypotheses alone;
  - `refuted`: the negation is proved, so the statement is about the value outside the domain.
    `integral_undef` and `condExp_of_not_integrable` come out this way, which classifies a
    definition's lemmas about its junk value without a list;
  - `open`;
  - `unapplied`: the definition is used as a function.
- **Dischargers** are tactics named as text, as JunkValues had it:
  - the defaults are `omega`, `infer_instance`, `positivity`, `fun_prop`, `norm_num` and
    `simp_all`, each with a budget of 10000 (in the unit of `maxHeartbeats`) per obligation;
  - a catalogue adds its own tactic. The Mathlib catalogue's `mathlib_catalogue_discharger` is a
    `solve_by_elim` over the facts that show integrability in probability theory: `MemLp`,
    martingales, set integrals, stopped processes.
- **On Mathlib's probability theory** (every theorem of `Mathlib.Probability` and
  `Mathlib.InformationTheory`, 4,181), with the catalogue's three domains:
  - 477 theorems use one of the three definitions, with 1,092 obligations;
  - with the default dischargers alone, 179 are discharged and 907 open;
  - with the catalogue's discharger and a budget of 10000: 409 discharged, 6 irrelevant, 677
    open, in 84 seconds on 16 threads.
- **Most open obligations are genuine:** library lemmas that hold by convention.
  - `integral_neg` and `integral_const` assume no integrability.
  - `condExp_add` does not assume `m ≤ m₀`.
- **Of the site's nine claims:**
  - the strong law is discharged by its hypothesis `hint`;
  - the central limit theorem is discharged by the catalogue's tactic, from `MemLp (X 0) 2 P`;
  - optional stopping leaves one obligation open. The integrability of `stoppedValue f τ` needs
    `τ ≤ π ≤ N`, a step through the pointwise order that `solve_by_elim` does not take.
  - The others use none of the three definitions.
- **Not built:**
  - the inside obligation, for definitions' bodies;
  - the boundary test with the cited theorems;
  - obligations for choice and for `@[up_to]`.

**What the other sources become.**
- **Derived domains** are shown for definitions with no declared domain. The missing declaration is
  an item for a reviewer.
- **Bridge lemmas and agents** propose entries for the Mathlib catalogue, which reviewers accept.
- **Rules** say what the value is outside the domain, and are cited in the declaration.

**In a statement, the position of an occurrence decides the risk:**

| where | the default makes it true | the default makes it false |
|---|---|---|
| conclusion | the theorem says nothing in that case (**the risk**) | the theorem also proves the side condition |
| hypothesis | the theorem also covers the junk case | the hypothesis silently assumes the side condition |

---

## 3. Choice

### 3.1 What Mathlib chooses

About 960 of Mathlib's 42,809 definitions use a choice constant directly in their data:
`Exists.choose`, `Classical.choose`, `Nonempty.some`, `Quotient.out`, `Classical.choice`,
`Classical.arbitrary`, and a few others. A quarter of them are in category theory. They differ in
what the choice leaves open:

| the chosen object is | examples |
|---|---|
| **unique**: the choice is canonical | a witness of `∃!`; the limit of a convergent filter in a Hausdorff space, which `Filter.lim` picks with `Classical.epsilon` |
| **unique up to a relation** | `Submodule.IsPrincipal.generator` (up to a unit); `Module.finBasis` (up to a change of basis); `MeasureTheory.Measure.haar`, built from a compact set picked with `Classical.arbitrary` (up to a scalar); limits and colimits in category theory, picked with `Classical.choice` (up to a unique isomorphism); `Measure.rnDeriv` (up to a.e. equality, §4) |
| **arbitrary** | `Function.invFun f y` off the range of `f` (`Classical.arbitrary`); `Filter.lim` of a filter with no limit; the Dirac mass of §2.1 |

### 3.2 Theorems are safe; definitions are not

`Classical.choice` is an axiom with no computation rule. A proof can learn about
`Classical.choose h` only two things: what `Classical.choose_spec h` says, and that the same choice,
made twice, gives the same object. So **whatever Lean proves about a chosen object holds for every
object with the specified property**. The meaning hash already reads a choice this way:
`Classical.choose h` means "some object with that property" (trusting-definitions.md, the note at
the end of §6).

Three consequences:
- **Choice never weakens a theorem.** A theorem about `Module.finBasis` holds for every basis.
- **The risk is a definition that isn't canonical.** If a definition's value depends on the choice
  while the intended object is canonical, then it is not the intended object, whatever is proved
  about it. This is F6 in trusting-definitions.md.
- **Its value cannot be pinned down.** No specification or test can fix what the choice leaves
  open. A definition with an open choice shows up as one whose values nothing proves.

### 3.3 The obligation: independence

Take a definition `D xs := body[Classical.choose h]`, with `h : ∃ w, P w`. The obligation is

    ∀ w₁ w₂, P w₁ → P w₂ → body[w₁] = body[w₂]

It can be discharged in three ways.
- **In the term.** `Quot.lift` and `Trunc.lift` take the independence proof as an argument, so the
  obligation is part of the definition. `LinearMap.det` chooses a finite basis, then goes through
  `Trunc.lift` with `det_toMatrix_eq_det_toMatrix` as the proof.
- **By uniqueness up to a relation, plus invariance.** If `P` determines the witness up to a
  relation `R`, the obligation reduces to "the body respects `R`" (§4). The uniqueness comes from a
  uniqueness theorem or a `@[characterization]`.
- **By a separate theorem**, for example that a quantity defined through a chosen basis equals its
  value in any basis.

### 3.4 Detection and propagation

- **Choice sites** can be read from today's dataset. The meaning edges of a definition reach
  `Classical.choose`, `Exists.choose` and the others in data positions only, because
  `ltb-meaning/1` erases proofs. This was checked on:
  - `Submodule.IsPrincipal.generator`, `Function.invFun`, `Measure.rnDeriv`;
  - `Filter.lim`, `Measure.haar`, `LinearMap.det`, `AEEqFun.cast`.

  So a report at the level of names needs no Lean.
- **Obligations** need the term.
  - The analyzer abstracts each choice, states the obligation, and tries to discharge it: first the
    term's own `lift`, then a uniqueness theorem with the invariance check of §4, then a discharger.
  - What stays open goes to the author or to an agent.
  - A proof is kernel-checked, wherever it comes from (§6).
- **Declared intent.** By default a definition is meant to be canonical. An author marks the
  exceptions with an attribute (proposed: `@[noncanonical "why"]`): `Module.finBasis` is meant to
  be *some* basis. An open independence obligation on an unmarked definition is a finding. On a
  marked one it is expected, and the mark is what a reader sees.
- **Propagation.** Each definition is **determined up to** something: `=` (it is canonical), a
  relation `R`, or only its specification.
  - A definition is canonical if it uses canonical objects, and uses `R`-determined objects only
    through operations that respect `R`.
  - Otherwise it inherits the relation, or becomes undetermined.
  - The concern stops at a definition that proves its independence, or at a review with F6
    checked.

---

## 4. Objects defined up to a relation

### 4.1 Two forms

- **A quotient type:** `MeasureTheory.Lp`, `AEEqFun`, `Quotient`. The object is the class, and a
  representative enters only through a coercion or through `out`. The coercion of an element of
  `Lp` to a function is `AEEqFun.cast`, which takes `Quotient.out` then `AEStronglyMeasurable.mk`:
  two choices.
- **A function that is unique only up to the relation:**
  - `MeasureTheory.condExp`, unique up to a.e. equality
    (`ae_eq_condExp_of_forall_setIntegral_eq`);
  - `Measure.rnDeriv` (`Measure.eq_rnDeriv`), and `pdf`, defined from it;
  - `condDistrib` and `Measure.condKernel`, kernels unique up to a.e. equality in their first
    argument;
  - `Filtration.limitProcess`, chosen among the a.e. limits.

The relations that occur are:
- a.e. equality, and a.e. equality in one argument;
- isomorphism;
- association (a unit factor);
- scalar multiples.

**How the representative is picked matters.**
- `rnDeriv` and the a.e. part of `condExp` are chosen, so §3.2 applies: a theorem about them holds
  for every representative with the specified properties.
- But `condExp` returns `f` itself when `f` is already measurable for the smaller σ-algebra
  (`condExp_of_stronglyMeasurable`). That representative is a construction, not a choice.

### 4.2 What goes wrong

**In definitions: a use that the class does not determine.**
- Evaluating at a point: `pdf X ℙ x₀`, `condExp μ m f x₀`.
- A pointwise supremum `⨆ x, f x` where the essential supremum was meant.
- `Measurable f` or `Continuous f` where `AEMeasurable f μ`, or "has a continuous
  representative", was meant.
- Sets built from values, such as `{x | f x > 0}`, then used topologically.
- Uncountably many a.e. statements at once. Define `κ ω A := condExp μ m (A.indicator 1) ω` for
  every measurable `A`. For each `A`, this gives a function determined up to a null set. But nothing
  makes `κ ω` a measure: a union of uncountably many null sets need not be null. Regular
  conditional probabilities solve this problem, which is why `condDistrib` and `condKernel` exist.

**In statements: facts about the representative.** `measurable_rnDeriv` and
`stronglyMeasurable_condExp` are true of Mathlib's representatives, and useful. Read as statements
about the mathematical object, they say "has a measurable representative".
- If the representative was chosen, they hold for every representative with the specified
  properties.
- If it was constructed, they are facts about Mathlib's construction.

Either way this is a risk in how the statement is read, not a wrong theorem.

### 4.3 Knowledge

- **The relation.** What a definition is meant to be determined up to is intent, and the author
  declares it with `@[up_to R]` on the definition, a relation on its result type with the
  definition's arguments in scope: `@[up_to (· =ᵐ[μ] ·)]`.
  - **Built** on 2026-09-27 in TrustAnnotations. It is stored like a domain, as a hidden relation
    `d._upTo`.
  - The relation's arity is read off the relation: a definition returning a function has more
    binders in its type than arguments.
  - A catalogue declares it for another library's definitions. The Mathlib catalogue does so for
    `condExp` and `Measure.rnDeriv`.
  - What proves the declaration is a `@[characterization]`, whose relation is read off a uniqueness
    theorem's conclusion (trusting-definitions.md §3.3). A definition's page shows the two side by
    side; a declaration no characterization proves is a claim.

  Without a declaration, the relation comes from:
  - the type (a quotient);
  - a uniqueness theorem that nobody has tagged.

  A characterization needs no predicate either. `@[characterization]` with no keyword goes on the
  theorem that states it:
  - an iff, `R x (d …) ↔ Q x`, where `Q` is the property and the theorem gives both halves;
  - or a uniqueness theorem, `H₁ x → … → Hₙ x → R x (d …)`, whose hypotheses on the candidate `x`
    are the property.

  The definition and the relation are read off the conclusion. That the definition satisfies the
  property is shown by reflexivity of `R` for an iff. For a uniqueness theorem it is shown from the
  definition's `@[specifies]` theorems, applied a few deep with the theorem's other hypotheses in
  context. So existence needs no new declaration either: the specification theorems a definition
  has anyway are what show it. A condition nothing shows is recorded as open, and the
  characterization as incomplete. Every other assumption of the theorem is recorded too, as where
  the characterization holds: its other hypotheses and all its instance arguments, with nothing
  judged irrelevant, since a missing assumption would make the characterization look more general
  than it is.

  Existence often needs more than uniqueness: `⟨M⟩` compensates `M²` only for an adapted,
  square-integrable `M`, while any two compensators agree without that. A premise of a
  specification theorem that the context does not provide is therefore assumed, and recorded as
  where the definition has the property. The uniqueness theorem then needs no hypothesis its own
  proof does not use, so no linter exception. Only premises can be assumed, never a condition
  itself, and only propositions about the theorem's own variables that don't mention the
  definition.

  Mathlib's own uniqueness theorem for conditional expectation,
  `ae_eq_condExp_of_forall_setIntegral_eq`, is such a characterization as it stands:
  - its three hypotheses on `g` are the defining property;
  - its conclusion `g =ᵐ[μ] μ[f | m]` gives the relation;
  - the hypotheses `m ≤ m₀` and `Integrable f μ` say where it holds;
  - existence is shown from `integrable_condExp`, `setIntegral_condExp` and
    `stronglyMeasurable_condExp`. `setIntegral_condExp` implies the hypothesis it proves but doesn't
    match it syntactically, since it needs no `μ s < ∞`.

  The predicate form (`@[characterization property d]`) stays, for a property worth a name of its
  own. **Built** on 2026-09-27 (TrustAnnotations a3f48c6; evidence-core 0.7.2 reads it), and
  checked on the conditional expectation above, through restatements carrying the attributes.
- **Which operations respect it**, from congruence lemmas:
  - `integral_congr_ae`, `lintegral_congr_ae`, `eLpNorm_congr_ae`;
  - `condExp_congr_ae`, `withDensity_congr_ae`, `Measure.map_congr`, `Integrable.congr`;
  - the `Filter.EventuallyEq` combinators.

  These lemmas can be found by their shape: a hypothesis `R a b`, and a conclusion relating `F a`
  and `F b`. Unlike junk-value rules, here the shape says exactly what is wanted, with no intent to
  judge.

### 4.4 The check and the obligation

The analyzer walks a definition's body:
1. It marks the positions that hold an `R`-determined value.
2. It follows them through operations that have congruence lemmas.
3. Each position that reaches an operation with no congruence lemma is a **non-invariant use**.

For each non-invariant use, the obligation is

    ∀ g, g ~R f → body[g] = body[f]

It is discharged as in §3.3, or left open.
- In a definition, an open obligation is the finding: the definition's value depends on the
  representative.
- In a statement, a non-invariant use is shown ("about Mathlib's representative"), not reported.

### 4.5 Up to isomorphism

Types keep most statements invariant under isomorphism: short of unfolding `AlgebraicClosure k` (a
quotient of a polynomial ring), nothing tells it apart from another algebraic closure. The risk is
generality (F9): a claim about Mathlib's construction does not apply to another algebraic closure,
such as the algebraic numbers inside `ℂ`, without a transfer.

Mathlib's answer is a characteristic predicate: `IsAlgClosure`, `IsLocalization`, `IsFractionRing`,
`IsSplittingField`, `IsColimit`. The report to make is: claims stated on a construction where a
characteristic predicate exists. That belongs to the generality report of trusting-definitions.md
§3.12, not to well-definedness.

A type can also be characterized by one theorem, with no predicate (TrustAnnotations 2768fa0,
90e4a33):
- The candidate is a type, its property is its instance arguments, and the relation is an
  isomorphism type: `Nonempty (K ≃+*o ℝ)`, or `∃ e : K ≃+*o ℝ, P e`.
- Existence finds the definition's instances in order, and fits the theorem's universes to the
  definition's.
- The catalogue characterizes `ℝ` this way, as the conditionally complete linearly ordered field.

**An isomorphism pins down only the structure its statement names.** `K ≃+*o ℝ` names `ℝ`'s `+`, `*`
and `≤`. Mathlib defines `0`, `1`, `-`, `⁻¹`, `<`, `max`, `min` and the four casts by instances of
their own, each on the Cauchy sequences, and nothing in the statement ties them to `+`, `*` and `≤`.
- With the bare isomorphism, taking `ℝ` from its characterization in `condExp`'s graph removed only
  7 declarations. 43 of the Cauchy construction stayed, reached through those instances.
- The catalogue's statement now says that the isomorphism preserves each of them too, which it
  does, like any isomorphism of ordered fields. Then 15 remain, all through `Real.commRing`:
  its casts are defined on the construction directly, not through `Real.instNatCast`.
- A statement could name a bundle such as `Real.commRing` as well. But naming one of its projections
  pins that field, not the bundle. The graph treats an instance named by the statement as pinned
  whole, which is right for the one-field classes named so far and not in general (§8).

### 4.6 Reading a graph through characterizations

A definition's dependency graph shows its construction. A characterization gives a second graph:
the characterization's statement in place of the construction. It is shown only when the reader asks,
and only for the definitions they choose.
- **Never all at once.** A characterization can hold on a smaller domain than the definition's uses.
  The catalogue's integral characterization is for real-valued functions only, and `condExp` uses
  the integral of vector-valued ones. Substituting every characterized definition would claim more
  than is known.
- **What the reader can do** (referee-site 5070ab3):
  - take any characterized definition in the graph from its characterization, from a row of
    switches or from its node's card;
  - go back to "As defined" at any time.
- **What a substitution does.**
  - The definition, and the instances its characterization pins down (`Real.instMul`…), rest on
    the characterization's theorem alone.
  - The theorem rests on what its statement uses, except the definition and those instances.
- **The note under the graph** gives, for each substitution:
  - the relation;
  - where the characterization holds: its assumptions and variables, and what it fixes (**only for**
    `G := ℝ`). The graph is only as general as that.
  - how many declarations are left out and added;
  - what of the construction the graph still reaches, and through which definitions.
    - The construction is what the definition and the pinned instances rest on, less what
      anything unrelated to the definition also rests on.
    - On a slice where everything rests on `ℝ`, generic pieces like `abs` count as `ℝ`'s own. The
      note errs towards showing too much.

**Could a characterization's restricted applicability be detected?** Partly.
- **Built:** the arguments a characterization fixes rather than quantifies over (`G := ℝ` for the
  integral) are recorded as `specialized`. The graph and the pins show them as "only for".
- **Possible, not built:** hypotheses stronger than the declared domain. For `condExp`, the
  characterization assumes `m ≤ m₀`, σ-finiteness of the trimmed measure and integrability.
  - When the definition has a declared domain, the question is whether the domain implies the
    characterization's context. The existence prover could try that, with the same lemmas.
  - What it cannot prove is a gap to report, not an error.
- **That a given use is covered** needs more:
  - the use's arguments, and the hypotheses in force at the use;
  - instance-level edges from the extractor, which today records only which declarations a
    declaration uses, not at which arguments.
- **Most characterizations will be restricted.** Mathlib's uniqueness theorems carry the hypotheses
  their proofs need. So this is coverage information, shown next to the characterization: not a
  warning, and not a verdict.

---

## 5. One mechanism

| obligation | knowledge | stated for | discharged by | open means |
|---|---|---|---|---|
| **domain** (§2) | domains declared with `@[domain]`, by the author or a catalogue; derived ones where none is declared; theorems for the value outside | inside each definition with a declared domain, and at each use in claims, specifications and definitions | hypotheses, types, `if` branches, fields of structures, through a discharger; or the value is irrelevant | in a definition, a junk value taken inside its declared domain (F4); in a statement, it may hold for no reason |
| **choice** (§3) | the choice's specification; uniqueness theorems; `@[noncanonical]` for the intended exceptions | each choice in a definition's data | the term's `lift`; uniqueness plus invariance; a theorem | the definition is not canonical, unless marked so |
| **representative** (§4) | the relation, declared through `@[characterization]` or read off the type; congruence lemmas | each non-invariant use in a definition | congruence lemmas; a theorem | the definition depends on the representative |

**Theorems and definitions are not at risk in the same way:**

| | a theorem | a definition |
|---|---|---|
| junk value | can hold for no reason (F5) | can mean something else (F4) |
| choice | safe: holds for every choice | can be non-canonical (F6) |
| representative | safe if the representative was chosen; if it was constructed, a fact about the construction | can depend on the representative (F6) |

**What propagates.** Each definition gets a *well-definedness profile*: its domain, what it is
determined up to, and its open obligations. Profiles are derived through definitions' bodies, from
the profiles of what they use, and never through proofs.

Two things propagate separately:
- **The fact** (a domain, a relation) is mathematics. Only a guard, a congruence lemma or a proof
  stops it. A declared domain stops the derivation: what is derived below a definition with a
  declared domain is checked against the declaration (§2.5), not passed on.
- **The concern** is "nobody has looked at this".
  - It is settled by a review with F4 or F6 checked, at the definition where the fact first
    appears.
  - It comes back when that definition's meaning hash changes.

**Up to claims.** A claim lists what its statement leaves open, anywhere below it:
- the residual of its own statement;
- the domain of each definition it uses, where its hypotheses don't cover it;
- each open choice or representative obligation.

This is what trusting-definitions.md §3.8 proposed ("junk findings shown by default on claims"),
extended to the choices of §3.12.

---

## 6. In the suite

- **The attributes** `@[domain]` and `@[up_to]` (built) and `@[noncanonical]` (not yet), in TrustAnnotations. The
  extractor exports them as the facets `annotation.domain` and `annotation.noncanonical`, with no
  release of its own.
- **A new analyzer** (suite-design.md, component 3), in a new repository of LeanTrustBuilders that
  depends on Lean core only: one term walk in the local context, three kinds of knowledge, one
  discharger interface. It is not an evolution of JunkValues, whose ideas it borrows (§2.3).
  **Built** for domains on statements (§2.5):
  - [LeanTrustBuilders/well-defined](https://github.com/LeanTrustBuilders/well-defined);
  - the facet `welldefined/1`, written by `trust-extract welldefined`, and by the extract action
    with `welldefined: true`;
  - carried by `evidence-core merge` from a catalogue's dataset;
  - shown by referee-site on each theorem's page, and on the claims list.
- **A Mathlib catalogue**, a separate package because it imports Mathlib. It declares the domains
  of Mathlib's operations with `attribute [domain …]`, starting with those of §2.1. Entries are
  proposed from bridge lemmas and by agents, and reviewed like code.
  - **Built** on 2026-09-27: [LeanTrustBuilders/mathlib-catalogue](https://github.com/LeanTrustBuilders/mathlib-catalogue).
    It holds:
    - the domains of `MeasureTheory.integral` and `MeasureTheory.condExp`;
    - a characterization of the real integral by `∫⁺ − ∫⁻`;
    - characterizations of `condExp` and the Radon–Nikodym derivative by Mathlib's uniqueness
      theorems;
    - a characterization of `ℝ` up to isomorphism (§4.5).
  - Mathlib's own theorems can't carry an annotation written outside Mathlib, so the catalogue
    restates the characterizations and specification lemmas, each proved by the lemma it restates.
  - CI publishes the catalogue's own small dataset. It holds the catalogue's theorems, and the Mathlib
    declarations they mention with the annotations.
  - `evidence-core merge` adds it to the Mathlib dataset of the same tag. The merge checks that the
    two agree on the meaning hash of every declaration they share.
  - The Mathlib probability site is built from the result. A definition's page shows its domain, and
    a catalogue's theorems appear apart from what Mathlib's authors wrote.
- **A dataset facet, `welldefined/1`**, produced by the extractor as it produces `check`.
  - Per declaration, the profile: domain, and what it is determined up to.
  - Each obligation, with:
    - its kind;
    - its place (statement or body);
    - its site;
    - its statement, pretty-printed;
    - its status: `discharged` (with how), `open`, or `refuted`;
    - the keys of the lemmas used.
  - In the header: the knowledge sources with their revisions, and the discharger. Results depend
    on the discharger, so the facet records which one was used.
- **Proofs of obligations.**
  - An author proves an obligation in the source, with an attribute from TrustAnnotations. The
    attribute checks that the theorem's statement is the generated obligation.
  - An outsider or an agent cannot edit the source. Their proofs go into a side module that CI
    compiles against the library.
  - The next extraction reads both kinds of proof back.
  - Who hosts and compiles side modules is open (§8).
- **evidence-core** reads the facet: the profiles, the lists for claims, and a risk signal for
  `queue`. Not a policy switch at first: an open obligation asks for attention, it is not a
  verdict.
- **S3 records.** `checked.F4` and `checked.F6` can list the obligations the reviewer looked at, and
  a `problem` with F4 or F6 can point at one.
- **Front ends.**
  - referee-site:
    - a definition's page shows its profile;
    - the claim page shows the list of what is left open;
    - the review form lists the open obligations, to tick.
  - mathlib-explorer:
    - a page of the conventions that surprise newcomers, built from the catalogue's convenience and
      convention entries, each with its reason;
    - "up to" made plain on pages such as `condExp`'s: "one particular function among those equal
      almost everywhere".

---

## 7. Order of work

1. **Names only, with no Lean**, in evidence-core on today's dataset. This gives referee-site's
   claim page a first signal. It covers:
   - choice sites in definitions' data;
   - representative sites in definitions' data (`AEEqFun.cast`, `AEStronglyMeasurable.mk`,
     `AEMeasurable.mk`; 20 Mathlib definitions use one directly);
   - operations prone to junk values, against domain predicates among the hypotheses;
   - the proof-term scan.
2. **The attributes.** `@[domain]` is done, together with characterizations without a predicate
   (TrustAnnotations a3f48c6). `@[noncanonical]` remains. The facets come with no work on the
   extractor, so a page can show declared domains at once.
3. **The analyzer**, for domains: the inside and outside obligations with the irrelevance test, on
   claims, specification theorems and definitions. The outside obligation on statements is done
   (§2.5), and runs on Mathlib's probability theory. The inside obligation, for definitions' bodies,
   remains.
4. **The Mathlib catalogue.** Started with the Bochner integral and conditional expectation, and
   merged into the Mathlib probability site. Next: the rest of the operations of §2.1.
5. Choice obligations, and invariance with mined congruence lemmas.
6. Proofs of obligations, from the source and from side modules.
7. The review form and the front ends.

---

## 8. Open questions

- **Where outsiders' proofs live**, and who compiles them against which revision.
- **Domains and the meaning hash.** Changing the domain of `klTerm` does not change `klTerm`'s
  meaning hash, and the hidden predicate is not a node of the dataset, so a review of `klTerm`
  would not go stale. Either a review records the domain it was made against (the entry's text can
  be hashed), or the domain is reviewed on its own.
- **An operation with no declared domain is treated as total**, so the catalogue's coverage decides
  what is found. A list of the operations reached with no declared domain shows the gaps.
- **Under binders.** `∫ x, log (f x) ∂μ` needs `0 < f x` only for almost every `x`. The irrelevance
  test should know that an integral ignores null sets, which is a congruence lemma of §4.
- **Dischargers are the limit.** What a claim is said not to show depends on them. A catalogue's
  own tactic helps. Before an open obligation is presented as a problem, it should come with a way
  to close it: a proof written by the author, or by an agent, in a side module.
- **Hypotheses under binders.** `∑ i ∈ s, f i` does not bring `i ∈ s` in scope, nor does `∀ᵐ x ∂μ`
  bring anything about `x`. Both need a table of binders and what they bring, like the connectives.
- **The irrelevance test** needs case analysis (`p = 0` or `0 < p`) that a generic discharger may
  not do. How much to automate, and how an author supplies the rest, is open.
- **How deep to derive**, for definitions with no declared domain. Derived domains and relations
  grow with depth, and so do the discharger's failures. A reviewed definition is a candidate
  stopping point.
- **The shapes of congruence lemmas** beyond a.e. equality: isomorphisms, units, scalars.
- **Stability across versions.** A change of Mathlib or of the discharger can open or close
  obligations with nothing in the library changed. The facet records both, and a diff should say
  which one changed.
- **Objects that are a.e.-defined in one argument only**, such as kernels and disintegrations, also
  need measurability in the other argument. There are fewer congruence lemmas for them, so the
  check will over-report there first.
- **Perturbation** (§2.4) stays research.
- **Bundled instances.** A characterization pins the instances its statement names, and the graph
  treats each named instance as pinned whole. For a bundle (`Real.commRing`), naming one
  projection pins one field. Deciding when a bundle is pinned needs the class's fields. The
  extractor could record them. Otherwise the rule could be restricted to one-field classes.
- **Whether a declared domain is the intended one** is a judgment. Reviewers now compare it with
  the source, as part of F3 and F4.
