# Dependency lists: how they are computed and how they are tested

Snapshot of 2026-09-25. This is a companion to
[review-tools-comparison.md](review-tools-comparison.md). Every tool reviewed there, and several
libraries they rely on, needs to know what a Lean declaration depends on. This note reviews how
each one computes that list, how the result is tested, and what is missing. Shortcomings and
recommendations are at the end.

Repositories read at:

| repository | commit |
|---|---|
| [mathlib-initiative/aftk](https://github.com/mathlib-initiative/aftk) | `17d641f` (2026-08-29) |
| [chrisflav/trust](https://github.com/chrisflav/trust) | `7fdbd6a` (2026-09-12) |
| [RemyDegenne/meaning-graph](https://github.com/RemyDegenne/meaning-graph) | `0b630e4` (2026-09-22) |
| [LeanMachineLearning/exposition](https://github.com/LeanMachineLearning/exposition) (Referee) | `bb51dc2` (2026-09-22) |
| [mathlib-initiative/semantic_hash](https://github.com/mathlib-initiative/semantic_hash) | `0496f6d` (2026-08-07), the commit trust pins |
| [RemyDegenne/challenge-gen](https://github.com/RemyDegenne/challenge-gen) | `9b5a133` (2026-09-22) |
| [CBirkbeck/tauceti-reviewed-by-test](https://github.com/CBirkbeck/tauceti-reviewed-by-test) (Reviewed-by) | `63a81e6` (2026-09-24) |
| Lean core, `src/lean/Lean/Util/FoldConsts.lean` and `Lean/Replay.lean` | `v4.33.0` |

---

## 1. Why dependency lists matter

| tool | what it uses a dependency list for |
|---|---|
| Referee | whether an accepted claim is *covered* (everything its statement rests on is accepted too); the upstream packages a statement rests on; *indirect invalidation* in the revision diff; the scope of claims-only builds |
| trust | the graph a reader navigates; the "trusted mode" that cuts the graph at trusted declarations |
| semantic_hash | making a hash cover everything below a declaration. This is what lets Referee's verdicts and trust's certificates go stale when something underneath changes |
| ChallengeGen | which declarations a standalone file must contain |
| Reviewed-by | which `example`s count as unit tests of a declaration |

A list can be wrong in two directions:

- **Missing a dependency.** This is the dangerous direction. A reader is told that a statement
  doesn't rest on something it does, a review counts as covered when it isn't, and a change
  underneath goes undetected.
- **Having an extra dependency.** This inflates closures and the amount of work shown to a reader,
  and it reports changes that don't matter. It can also hide the first kind of error: extra
  dependencies can drag in a declaration that a correct list would have needed and missed
  (section 4.3 gives a real case).

---

## 2. Four notions of "dependency"

The tools don't all compute the same thing, and some of the differences are deliberate:

| notion | what it contains | who computes it |
|---|---|---|
| **term dependencies** | every constant in the elaborated type and value, proofs included | aftk; MeaningGraph `deps`; semantic_hash (proof-relevant variant) |
| **statement dependencies** | the statement, plus the data in a definition's value, without proofs. Referee calls this `meaningDeps` | MeaningGraph `typeDeps` and `dataDeps`; trust's statement and body edges; ChallengeGen's flat printer; semantic_hash (proof-irrelevant variant), approximately |
| **source dependencies** | term dependencies, plus what the source text needs to elaborate again: notation, coercion instances, lemmas named in tactic blocks, sibling declarations from the same command | MeaningGraph's recoveries; ChallengeGen's readable tier |
| **names written in source** | the identifiers written in a statement, resolved to declarations | Reviewed-by |

Statement dependencies are what a reader has to trust, so most of what the tools show a reader
rests on that notion. Two lists should only be compared within the same notion.

---

## 3. The shared starting point in Lean core

Five of the six mechanisms start from Lean core's `Lean/Util/FoldConsts.lean`:

- `Expr.getUsedConstants` collects the constants in an expression, through `Expr.foldConsts`.
- `ConstantInfo.getUsedConstantsAsSet` takes the constants of the type and of the value together.
  For a declaration with no value, it returns the constructors of an inductive type and the whole
  family of a recursor.

**Blind spot:** `Expr.foldConsts` skips the structure name carried by an `Expr.proj` node. A
structure that appears only inside a projection is not reported.

Source code rarely produces `Expr.proj`: `p.x`, `p.1`, `let ⟨a, _⟩ := p` and `{ p with … }` all
elaborate to applications of the projection *function* `Point.x p`. Compiled code produces it
often. After `import Lean`, 3,417 constants that are not projection functions contain an
`Expr.proj` node, and 2,545 (constant, structure) pairs have the structure only inside one. Two
examples, checked on Lean v4.33.0:

- **Definitions by structural recursion.** For `def total : List Nat → Nat`, Lean generates a
  helper `total._f` that reads the recursive call's result as `x_1.1`, which is `Expr.proj PProd 0 x_1`.
  `PProd` is not among the helper's `getUsedConstantsAsSet`.
- **Well-founded recursion with a custom relation.** `String.Model.positionsFrom` projects
  `WellFoundedRelation` out of `invImage …`, and `WellFoundedRelation` is not among its direct
  dependencies.

**Why closures are unaffected.** For `Expr.proj S i b` to typecheck, the type of `b` must reduce to
`S …`, so `S` occurs in the closure of whatever `b`'s type is built from. In the two examples,
`PProd` is reached through `List.below` and `WellFoundedRelation` through `invImage`. So the blind
spot drops a *direct* dependency but not a transitive one. It matters wherever direct dependencies
are used on their own: in a drawn graph, or in semantic_hash's past bug (section 4.4).

---

## 4. The six mechanisms

### 4.1 aftk

- **Code:** `AFTK/Dependency.lean`, `directDependencies`.
- **Computes:** flat sets of transitive dependencies (`deps`) and transitive reverse dependencies
  (`rdeps`). A direct dependency is anything in `ConstantInfo.getUsedConstantsAsSet` that exists in
  the environment. These are term dependencies: proofs are followed.
- **Choices:** compiler-generated constants are traversed but hidden from the output. The
  `Expr.proj` blind spot is inherited. Everything imported is in scope.
- **Used by:** `aftk deps` and `aftk rdeps`, and trust's `trust rdeps`.
- **Tested by:** `tests/dependency.sh` in CI. It builds a toy project of one-line `Nat` definitions
  and checks the exact output of queries. It tests how queries are scoped (module, library,
  package) and how names are resolved (private names, suffixes, quoted names). It does not test
  whether the dependencies are right, and nothing runs on real library code.

### 4.2 trust

- **Code:** `Trust/Deps.lean`, `definitionalClosure`. Reverse dependencies come from aftk
  (`Trust/Reverse.lean`).
- **Computes:** a graph with two kinds of edges: *statement edges* (constants in the type) and
  *body edges* (constants in the value, and in the constructor types of an inductive).
- **Choices:**
  - Traversal enters a node only if it carries data, meaning its type is not a `Prop`. Theorems
    and other proofs are leaves: they appear in the graph, but nothing below them is followed.
  - Compiler-generated constants such as matchers and `_proof_n` are contracted: their own
    successors take their place. So when a definition's value contains a lifted `_proof_n`, the
    lemmas that proof uses appear as leaves among the definition's body edges.
  - `--with-proofs` adds the edges of theorem proofs to the export.
  - The `Expr.proj` blind spot is inherited.
  - Everything imported is in scope, core and Mathlib included.
  - `trust rdeps` follows types and values (proofs included) on purpose.
- **Used by:** `trust deps`, `trust export`, the trust-web graph and its trusted-mode cut.
- **Tested by:**
  - `lake test` covers command-line options (for example, that proof edges are exported only when
    asked), signatures and the federation protocol, but not which dependencies are computed.
  - trust's integration CI exports an index of Lean core's `Init` and runs trust-web's tests
    against it (`src/data/index.test.ts`). Those tests check a few hand-picked facts: `Nat.gcd`'s
    statement depends only on `Nat`; its body closure includes `WellFounded.Nat.fix` and `InvImage`
    but not `Nat.ble_eq_true_of_le`, a lemma used only in a proof; it has dependents; a theorem is
    classified as a proof; and a closure has no dangling edges. This is the only check on real
    data among the six mechanisms, apart from the flat printer's compile check.

### 4.3 MeaningGraph

- **Code:** [meaning-graph](https://github.com/RemyDegenne/meaning-graph), `Context.declDeps`.
- **Computes:** three lists per declaration:
  - `typeDeps`: what the statement mentions;
  - `deps`: the statement and the proof or body;
  - `dataDeps`: the statement and the data in the body.

  Referee's `meaningDeps` is `typeDeps` for a theorem and `dataDeps` for a definition. Referee's
  `closureDeps`, used for extraction, is `deps`.
- **Choices:**
  - It starts from `getUsedConstants` and adds four things the elaborated term drops: compiler
    helpers are expanded through to the declarations a human wrote; the structure of each
    `Expr.proj` is recovered; notation expansions and coercion instances are recovered. The last
    two are source dependencies.
  - `dataDeps` skips proofs by position: for a function whose result is a structure, arguments in
    `Prop` positions are not walked (`constPropMask`). This is conservative: other proofs inside a
    value are still walked.
  - A dependency whose module is not visible through imports is dropped.
  - Only the project is in scope. Upstream constants are leaves.
- **Used by:** Referee's site (coverage, the trust surface, revision diffs, claims scope) and
  ChallengeGen's readable tier.
- **Tested by:**
  - `lake build MeaningGraphTest`: `#guard` checks of name classification, `Expr`-level constant
    collection, the notation and coercion recoveries, and the graph passes. The test file states
    that the functions needing a full environment (`usedConstantsOf`, `expandThroughInternals`,
    `Context.declDeps`), which compute the actual dependencies, are "exercised against a real
    project instead" and have no unit tests.
  - `lake build MeaningGraphProofs`: proofs that the project boundary is a component-wise prefix
    order, that internal names are inherited downwards, that `topologicalClosure` has no duplicates
    and is closed under dependencies, and that `projStructureNames` is complete.
  - Nothing checks the dependencies against an outside reference. The readable tier's compile rate
    covers only `deps`, the closure with proofs (section 5).
- **Past bugs:** Referee's `WEBSITE-DESIGN.md` records two, both found by reading one extracted
  file:
  - Coercion instances were keyed too coarsely, so one instance was attached to 208 declarations
    that did not use it.
  - Nothing checked that a dependency could be reached through imports. Adding that check removed
    188 edges, all of them impossible.

  Together, the fixes cut transitive edges across the corpus by 34%. They also made two extracted
  files stop compiling: those files had compiled only because the extra edges pulled in a
  declaration that was really needed but missing for another reason.

### 4.4 semantic_hash

- **Code:** `SemanticHash/Hashing/Expr.lean`, `foldReferencedConsts`.
- **Computes:** the constants each expression references, so that a declaration's hash can include
  the hashes of its dependencies. The list is never output; it shapes the hash.
- **Choices:**
  - The `Expr.proj` structure name is recovered.
  - The proof-relevant variant follows proofs. The proof-irrelevant variant skips theorem bodies,
    including `_proof_n` theorems, but still follows proofs written inline.
  - Constructors and recursors are hashed together with their inductive type.
  - Everything imported is in scope.
- **Used by:** every hash. Referee stamps verdicts with the proof-irrelevant hash; trust keys
  certificates by the proof-relevant hash.
- **Tested by:**
  - `HashingTests.lean`: about 230 equality and inequality checks on small hand-written fixtures,
    covering renaming, declaration kinds, propagation through references, and every kind of `Expr`
    node.
  - `SelectiveHashTests.lean`: on-demand hashing agrees with whole-environment hashing, in both
    variants.
  - Both run at elaboration time, so CI runs them on every build.
  - trust's `hash-invariants` re-checks the renaming invariants on real declarations.
  - On real code there are only benchmarks and duplicate counts (`BENCHMARKS.md`).
- **Past bug:** the walk used to miss the `Expr.proj` structure, like Lean core. A projection was
  then hashed by the structure's *name*, so renaming the structure changed the hash: the hash was
  no longer invariant under renaming. A regression test now covers it. A change to the
  structure's content still changed the hash, because the structure reaches the closure by
  another path (section 3).

### 4.5 ChallengeGen's flat printer

- **Code:** `ChallengeGen/Flat.lean`, `renderConst`, `resolveRef` and `assembleTarget`.
- **Computes:** each project declaration is printed once, in fully explicit form: every
  application as `@f a b …`, every constant qualified with `_root_.`. A constant is a dependency if
  and only if the printer wrote it. A target's closure is `MeaningGraph.topologicalClosure` over
  those records.
- **What is printed:**

| kind | printed as | dependencies recorded |
|---|---|---|
| theorem | `theorem n : T := sorry` | the statement |
| definition | `def n : T := v` | the type and the value, with proofs removed |
| value over 100k characters, `partial`/`unsafe` definition, `opaque`, axiom | `axiom n : T` | the type only |
| inductive, structure, class | the matching command | parameters and constructor or field types |
| mutual family | one `mutual … end` block, printed by its first member | the other members point to the first |

- **Choices:**
  - Every proof subterm is printed as `sorry` and not entered, as decided by `isProof` in
    `MetaM`. This includes proofs inside statements. `Decidable` instances are data, so they are
    kept.
  - The `Expr.proj` structure is not recorded as a direct dependency. It still reaches the closure
    through the term being projected.
  - Compiler-generated constants (constructors, recursors, projections, `casesOn`, `noConfusion`,
    and so on) are redirected to the declaration that owns them.
  - Only the project is in scope; MeaningGraph supplies the project boundary. External
    dependencies come in by importing every external module that the project's modules import
    directly.
- **Used by:** `referee extract-flat`.
- **Tested by:** compiling every extracted file: 3164 of 3164 compile, across three projects
  (brownian-motion, LML, alpha-rar). `lake build ChallengeGenTest` covers only string helpers and
  file naming.

**The readable tier is not another mechanism.** It takes the closure its caller passes in
(`ChallengeDecl.transDeps`, which Referee fills with MeaningGraph's `deps`). It then widens that
closure until nothing changes, adding the notation commands the source uses and the sibling
declarations produced by the same source command (for example `@[to_additive]` pairs), together
with their closures.

### 4.6 Reviewed-by

- **Code:** `scripts/fetch_declarations.py`, functions `scan` and `resolve_tests`.
- **Computes:** it reads Tau Ceti's source text with regular expressions. For each `example`, it
  links the example to the Tau Ceti declarations that its statement names. Those examples are
  shown as the declaration's unit tests.
  - An identifier is a candidate unless it is bound locally: by the example's own binders, or by
    `∀`, `∃` or `fun` in its statement.
  - A candidate is looked up in the namespaces around the example, from the innermost outwards,
    then as written, then under each namespace the file opens. The first match wins.
- **Choices:**
  - Only direct links. Only Tau Ceti declarations that are not private.
  - No closure, and no compiled data at all.
  - Separately, a review mark is pinned to a hash of the declaration's own text: the whole text of
    a definition, or a theorem up to its first `:=`. The hash is SHA-256 of that text with
    whitespace normalized, truncated to 12 hex characters. Nothing below the declaration is
    covered. The script's docstring notes that a deployment inside Tau Ceti "would hash the
    elaborated terms instead".
- **Used by:** the "Tested by · N unit tests" counts on the page, and the staleness of review marks.
- **Tested by:** `tests/test_fetch.py`, run in the Pages workflow. It checks on a three-example
  fixture that names resolve through namespaces and `open`, and that a local name does not resolve
  to a declaration.

---

## 5. Side by side

| mechanism | notion | direct dependencies from | proofs | `Expr.proj` structure | generated constants | scope | checked against an outside reference |
|---|---|---|---|---|---|---|---|
| aftk | term | `getUsedConstantsAsSet` | followed | missed | traversed, hidden from output | everything imported | no |
| trust | statement (graph) | `getUsedConstants` on type and value, constructor types | proof nodes are leaves; lemmas from lifted `_proof_n` appear as leaves | missed | contracted | everything imported | a few facts about `Nat.gcd` in Lean core |
| MeaningGraph | term, statement, source | `getUsedConstants` plus four recoveries | `typeDeps` includes proofs inside statements; `dataDeps` skips proof arguments of structure-valued functions | recovered | expanded through | project | only `deps`, through the readable tier's compile rate |
| semantic_hash | term or statement, depending on the variant | its own walk | proof-relevant follows them; proof-irrelevant skips theorem bodies but follows inline proofs | recovered | hashed with their inductive | everything imported | no |
| ChallengeGen flat | statement | constants its printer writes | every proof subterm erased | missed as a direct dependency | redirected to owner | project | yes: every file compiles |
| Reviewed-by | names written in source | regex over source text | not applicable: only statements are read | not applicable | not applicable | Tau Ceti | no |

**What the compile checks show:**

- **Flat tier, 100%:** the flat printer's own closure is sufficient for the project part. External
  dependencies are over-supplied by importing whole modules, so the check says nothing about them.
- **Readable tier, 99.6%:** MeaningGraph's `deps`, the closure *with* proofs, is sufficient.
- **Neither tier** checks a statement-only closure from any tool other than the flat printer. Only
  the flat printer's list is checked, not the one Referee shows readers.
- **Neither tier** checks minimality. The 188 impossible edges passed the compile check.
- **Neither tier** checks that the file states the same thing as the original. That is fidelity,
  not dependency correctness; a Comparator run is proposed for it in Referee's `TRUST-GAPS.md`.

---

## 6. Shortcomings

1. **The closures readers rely on are not checked from outside.** Referee's coverage and upstream
   trust, trust's graph, and staleness through semantic_hash all rest on statement dependencies.
   The only statement closure checked systematically from outside is the flat printer's, and no
   tool shows that one to a reader.
2. **Tests barely exercise dependency semantics.** aftk tests query scoping; MeaningGraph
   unit-tests only helpers and leaves `Context.declDeps` to "a real project"; ChallengeGen tests
   string helpers; semantic_hash tests small fixtures; Reviewed-by tests a three-example fixture.
   trust has the only real-data assertions, and they are a few facts about one declaration of Lean
   core.
3. **Bugs so far were found by reading output.** MeaningGraph's coercion keying, the 188 impossible
   edges, and semantic_hash's `Expr.proj` bug were all found by hand. The first two were extra
   dependencies that made extracted files compile by accident, so the compile check hid them
   instead of finding them.
4. **The `Expr.proj` blind spot is handled inconsistently.** MeaningGraph and semantic_hash recover
   the structure; aftk, trust and the flat printer miss it as a direct dependency. Closures are
   unaffected (section 3), but direct edges can be missing one, such as those drawn in trust-web's
   graph. Structurally recursive definitions are affected through their `._f` helpers.
5. **The proof/data line is drawn five different ways.** The flat printer erases every proof
   subterm; MeaningGraph's `dataDeps` skips only proof arguments of functions that return a
   structure; trust stops at `Prop`-typed constants but keeps lemmas from lifted `_proof_n` as
   leaves; semantic_hash's proof-irrelevant variant skips theorem bodies but follows inline proofs;
   aftk follows everything.
   - Referee's `Collect.lean` describes the proof-irrelevant hash as "`meaningDeps` computed at the
     `Expr` level by an independent implementation". The two differ on inline proofs and on proofs
     inside statements.
   - So Referee's revision diff, which uses the hash, and its graph, which uses `meaningDeps`, can
     disagree about whether a declaration's meaning moved. The disagreement goes in the safe
     direction: reporting changes that don't matter.
6. **Reviewed-by uses no dependency information.**
   - A mark on a theorem stays valid when a definition its statement uses changes, because only
     the theorem's own text is hashed, and only 48 bits of it.
   - Unit-test discovery by name misses notation (`∑`, `‖x‖`), field notation on local variables
     (`p.foo`), and names introduced by `variable`.
   - Where Lean would report an ambiguous name, it takes the first match instead.
   - The `sorry` check on an example reads only that example's text, so it cannot see a `sorry` in
     a lemma the proof uses. Tau Ceti's CI enforces an axiom allowlist before merging, which should
     cover that case.
7. **Scopes differ.** MeaningGraph and the flat printer stop at the project; aftk, trust and
   semantic_hash cover everything imported; Reviewed-by covers Tau Ceti only. Results cannot be
   compared without restricting them to a common scope.
8. **Six implementations, no shared definitions.** Each rebuilds the same starting point with its
   own choices, and none states which notion of section 2 it implements. Comparing two outputs
   first needs normalization, for example of generated constants (kept, contracted, expanded,
   redirected, or hashed with their family).

---

## 7. Recommendations

In the suggested order:

1. **Write down the notions.** Define term, statement and source dependencies (section 2), with the
   choices each makes about proofs, projections and generated constants. Tag each tool's output
   with the notion it implements. This is cheap, and every comparison below depends on it.
2. **Compare every tool against the flat printer.** It is the only statement closure already shown
   sufficient, on 3164 files.
   - For each declaration, compare its closure with MeaningGraph's `meaningDeps` closure, trust's
     statement and body closure, and semantic_hash's walk.
   - A constant the flat file needed that a tool lacks is a missing dependency in that tool. A
     constant a tool has that the flat file didn't need is an extra dependency or a deliberate
     difference: MeaningGraph's source recoveries, or a different proof/data line.
   - Before comparing: redirect generated constants to their owners, restrict to project
     constants, set aside definitions the flat tier printed as axioms, and compare closures rather
     than direct dependencies.
   - Both sides are already computed on the three corpora, so this should give numbers quickly.
3. **Build that comparison into extraction.** Let the flat tier receive a second closure through
   `ChallengeDecl`, for example MeaningGraph's `meaningDeps` closure, and report every constant the
   printer needed that the given closure lacks. Every extraction run then tests MeaningGraph at
   almost no cost.
4. **Check sufficiency with the kernel, in CI.** Lean core's `Environment.replay` (in
   `Lean/Replay.lean`, the replay lean4checker is built on) adds `ConstantInfo`s to an empty
   environment through the kernel.
   - Give it a tool's claimed closure, with theorems turned into axioms and proofs inside
     definitions erased the way the flat printer does it. An "unknown constant" error means the
     closure is missing something.
   - No elaboration or pretty-printing is involved, so a failure is about the closure itself. This
     is the only check whose verdict comes from the kernel rather than from one of the tools being
     tested.
   - For minimality, on a sample: drop one constant from a set that replays. If the rest still
     replays, the kernel didn't need that dependency.
5. **Test semantic_hash and the graph against each other by changing one definition at a time.**
   - Change a definition D, either in a scratch copy of the project or in memory, as the hash's own
     tests do for fixtures. Rehash, and compare the declarations whose proof-irrelevant hash changed
     with everything whose statement depends on D, according to the graph.
   - Changed but not dependent on D: the graph is missing a dependency, or the hash follows proofs
     it shouldn't.
   - Dependent on D but unchanged: the hash's walk is missing a dependency, or the graph has an
     extra one.
6. **Close the known gaps.**
   - Recover the `Expr.proj` structure in aftk and trust, as MeaningGraph and semantic_hash already
     do.
   - Either align semantic_hash's proof-irrelevant variant with the chosen proof/data line, or
     correct the description in Referee's `Collect.lean`.
7. **Give Reviewed-by compiled dependency data.**
   - Key marks by the proof-irrelevant semantic hash of the elaborated declaration, as its own
     docstring anticipates. A change underneath then makes a mark stale.
   - Take a statement closure from trust's index or Referee's `data.json`, so the page can show
     "reviewed, but rests on unreviewed definitions".
   - Find unit tests from elaborated examples rather than from names in the text, for example by
     walking the info trees of `example` commands, which aftk already traverses for `tech-debt`.
8. **Publish check results per declaration.** Compile and replay status next to each declaration
   is the one claim a reader can verify without trusting the tool that made it, as Referee's
   `TRUST-GAPS.md` §6 argues for compile status.
