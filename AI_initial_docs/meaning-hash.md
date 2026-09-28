# One notion of meaning: the graph and the hash from the same rule

Proposed and implemented on 2026-09-26. The suite used two computations of what a declaration's
meaning depends on: MeaningGraph's `meaning` graph, and semantic_hash's proof-irrelevant hash. They
disagreed, and the suite used them together. The meaning hash is now derived from the graph's own
rule, in the same walk, so that for any choice of rule the two agree by construction. This note
records the problem, the rule chosen (`ltb-meaning/1`), and what it changed on real libraries. It is
a companion to [dependency-testing.md](dependency-testing.md) (§9, the self-checks) and to S1 and S2
in [LeanTrustBuilders/specs](https://github.com/LeanTrustBuilders/specs).

---

## 1. The problem

The views and the evidence core use the two together:

- **The graph** says what a statement rests on. Coverage, the trust surface, "what it rests on" on
  a claim's page, and the name given for "changed underneath: X" all come from it.
- **The hash** says whether a review still applies: current, stale underneath, or stale.

So a reader could be told that a review of D is current while the graph shows that something D
rests on changed meaning. Or they could be told that D changed underneath with nothing in its graph
to point to.

**Measured** with check 1 of dependency-testing.md §9 (`evidence-core check-graph`). Between two
Tau Ceti datasets (8befae0 and c59177e, across a Lean and Mathlib bump, 81,970 declarations in
both):
- **4 declarations** went stale underneath with nothing in their graph closure changed;
- **335 declarations** kept their meaning hash although their graph closure reached a declaration
  whose meaning changed (246 after discounting renames, which the first prototype miscounted).

**Why they differed.** The two erased different proofs:

| | `meaning` graph (MeaningGraph, before) | meaning hash (semantic_hash, proof-irrelevant) |
|---|---|---|
| proofs in a definition's own value | skipped where they fill a `Prop` parameter of a function returning a structure; kept elsewhere | followed when written inline |
| lifted proofs (`_proof_N`) | looked through, **whole proof included** | hashed by their statement only |
| helpers (`match_N`, private declarations, …) | looked through, **whole value included, proofs too** | hashed as constants in their own right |
| notation and coercion instances | included (source dependencies) | not seen: not in the elaborated term |
| constructors, recursors | looked through to their type | hashed with their type |
| beyond the project | leaves in the graph | the hash is deep: it covers everything |

§10 compares the graph semantic_hash's hash implicitly follows with the rule's.

**Traced.** `MeaningGraph.Context.sources` says, for each edge, where it comes from. On the
8befae0 environment, the 38 edges at the root of 231 of the Tau Ceti findings were traced:
- 228 came through helper chains containing proofs: a lifted `_proof_N` (136) or a private theorem
  (92). The graph read those proofs; the hash did not.
- The rest were a renaming artefact of the check itself (fixed), and the mirror case: a proof
  written inline, followed by the hash and skipped by the graph (`TauCeti.CurvedDuplex.toPeriodicComplex`
  reaching `zero_comp`).

## 2. The principle

Fix one **rule** for what a declaration's meaning depends on, and derive both the graph and the
hash from it, in the same walk:

- **nodes:** which constants are declarations in their own right. The others are helpers, whose
  content counts as part of the declaration that uses them;
- **erasure:** which parts of a declaration are not meaning (proofs);
- **the content** of a constant: its type, and its value if it is a definition, erased;
- **the edges** of a declaration: the declarations its content mentions, looking through helpers;
- **the meaning hash** of a constant: a hash of its content in which each reference to another
  constant is replaced by that constant's meaning hash. A Merkle hash;
- **the local hash** of a declaration: the same content, with references to other declarations by
  name. It says whether the declaration itself was rewritten.

**Consequence.** A declaration's meaning hash changes exactly when its own content (with the helpers
it looks through) changes, or when the meaning hash of something it refers to changes. By
induction, that is exactly when something in its graph closure changed, up to 64-bit collisions.
"Stale underneath" and "what it rests on" then agree by construction, and the declarations to blame
for a stale-underneath review are exactly the closure members whose local hash changed.

**Two simplifications came out of building it.**
- *No cycles to handle.* The kernel only lets a constant refer to constants added before it, or to
  its own mutual block. With inductive types taken as blocks (the types, their constructors and
  recursors), the references form a DAG, and the Merkle hash is well founded without computing
  strongly connected components. On LeanMachineLearning and Tau Ceti the walk found no unresolved
  reference.
- *The meaning hash does not depend on the node rule.* Every constant, declaration or helper, gets
  its meaning hash from its own content and its references' hashes. Which constants are nodes
  changes only the graph (which declarations the edges stop at) and the local hash. So rules that
  differ only in their nodes (ours, trust's) share one meaning hash.

## 3. The rule `ltb-meaning/1`, as decided

It lives in MeaningGraph, next to the graph it draws: `MeaningGraph.Hash` (`Walk.visit`,
`Walk.targets`, `Walk.meaning?`, `Walk.localHash`).

1. **Erasure: every proof, everywhere**, in statements, values and helpers alike.
   - An argument is a proof when its expected type, read off the type of the function applied
     (instantiated with the arguments before it), is a proposition; a let-bound value when its
     declared type is one. Either becomes a marker. The expected type comes from the function, so
     what is left mentions nothing that the proofs alone mentioned.
   - A declaration whose type is a proposition means its statement: theorems, `Prop`-valued
     definitions, and instances of `Prop` classes (`[IsProbabilityMeasure μ]`, `[Fact p]`), which
     are proofs too.
   - Rejected alternative: the old mask (`Prop` parameters of structure-returning functions only).
     It keeps some proofs, which then become edges, and it was the source of the disagreement.
   - Consequence, accepted: a definition made with a choice (`argmax := Classical.choose
     (exists_argmax …)`) no longer rests on the existence lemma. By proof irrelevance its meaning
     is "some choice of an object with that property". Which choice principle it uses is for the
     axioms facet and the choice report of trusting-definitions.md §3.12, not for the graph.
2. **Content.**
   - A definition: its type and its erased value.
   - A theorem, an axiom, an opaque constant (and so a `partial def`): its type.
   - An inductive type: its mutual block, taken as one unit: the types and their constructors,
     with the number of parameters, indices and constructors. The recursors follow from them and
     are not hashed. A reference to a constructor or recursor is a reference to its block and its
     position there; for the graph, an edge to the inductive type.
   - Left out of the hash: names (the declaration's own, the constants it refers to, binders),
     binder kinds, metadata, the names of universe parameters (by position instead).
3. **Nodes.** The declarations a person wrote, **private ones included** (`isDeclaration`, and
   `Display.declared` for the other walks). Before, private declarations were helpers: their
   content was inlined into their users, and nobody could review them. Matchers are helpers
   whatever their name (Lean names a second one `match_1_1` inside a private declaration).
   Projections are helpers: a use of `HAdd.hAdd` is an edge to `HAdd`.
4. **Source dependencies move out.** Notation and coercion instances are not meaning: the elaborated
   term does not depend on them. They are a fourth notion, `source` (`Context.sourceDeps`), which is
   what standalone files need.
5. **Past the project.** The hash is deep: the walk goes down to Lean core. The graph stops at the
   project, and its upstream targets are declarations under the same rule (upstream helpers are
   looked through too, `term` included). No cache was needed: on LeanMachineLearning the walk reaches
   12,336 blocks (the library and the part of Mathlib its meanings rest on) in 2 seconds.
6. **The local hash** (`ltb-local/2`): the same content, with references to other declarations,
   and to the helpers they own, by name. Helpers the declaration owns (named under it: `foo.match_1`)
   and helpers nobody owns are looked through, by their own local content. Still open: S1's known
   limit, that unchanged text can elaborate with other implicit or instance arguments when a
   constant it uses changes its signature, and so read as rewritten. A variant erasing implicit and
   instance arguments would fix it; not decided.
7. **Identity.** A rule has a name (`ltb-meaning/1`), which records carry as their hasher. Any change
   that moves a hash is a new name. The content hash stays semantic_hash's proof-relevant hash, which
   trust's certificates are keyed by.

## 4. Profiles

Each rule is a **profile**: a named set of the choices of §3. `MeaningGraph.Hash.Rule` holds the node
rule; `Rule.meaning` is the suite's (`ltb-meaning/1`), `Rule.completion` trust's node rule on the same
erasure. Since the meaning hash does not depend on the node rule, only rules that change the
erasure or the content would need hashes of their own.

**Comparing rules** is a tool of its own: `evidence-core compare-rules` takes two datasets of one
commit under two rules and says which declarations, edges and closures one has and the other lacks,
by kind of target. It is how the decisions of §3 were checked (§5).

## 5. What it changed

**LeanMachineLearning** (1,452 declarations before, on Mathlib), `ltb-dataset/0` against
`ltb-dataset/1` at the same commit:
- 16 more declarations: private ones;
- direct `meaning` edges: 25,126 in both, 6,635 only before, 5,775 only now. Gone: projections and
  constructors as targets (now their structure or inductive type: 4,808 new edges to classes),
  2,466 edges to upstream instances (mostly of `Prop` classes: `IsZeroOrProbabilityMeasure`,
  `NeZero`, …, now erased), 219 to upstream theorems (proofs inside statements), and 98 to project
  declarations (4 of them now `source` edges of notations, the rest reached only through proofs);
- closures slightly smaller: mean 41.0 declarations against 43.9; 175 declarations lost some
  project declaration from their closure, 27 gained one.

**The kernel agrees.** Check 2 (`trust-extract check`) on the new dataset: every one of the 1,468
closures is enough for Lean's kernel, along `meaning` (proofs erased) and `term`, and no declaration
mentions anything its closure lacks. So the edges that went away were needed only by proofs.

**Graph against hash** (check 1), across a Mathlib bump (LeanMachineLearning 61e506b to d707a02):
nothing found. And the new hash is more precise: semantic_hash marked 67 declarations stale
underneath, all because one Mathlib instance, `unitInterval.instMeasureSpaceElemReal`, changed its
hash. The Mathlib change behind it removed proof fields (`zero_apply _ := rfl`) from instances of
`Prop` classes on `OuterMeasure`: a change of proofs only. Under `ltb-meaning/1`, no meaning hash
moved, and 989 declarations read "current, a proof in its closure changed".

**Tau Ceti** (8befae0, 81,999 declarations before): see §7.

## 6. The transition

- **S1 version 1, S2 `ltb-dataset/1`.** The meaning and local hashes are the rule's; `meaning` means
  the rule's graph; a `source` notion is added; private declarations are nodes. Extractor 0.7.1 and
  later (0.7.2 now), for Lean 4.35.0-rc2, 4.34.0 and 4.34.0-rc2.
- **Records keyed before.** Every node of an `ltb-dataset/1` dataset also carries its version-0
  hashes (`hashes.legacy`). evidence-core 0.3.0 resolves a record keyed by semantic_hash through a
  dataset of its own commit that has both (re-keyed by name and legacy hash, then judged as a new
  record), or, without one, by comparing with the current dataset's legacy hashes: the old verdict.
  The conformance vectors check both paths on the old records.
- **The views.** Records are not the only things keyed by the old meaning hash: the Tau Ceti
  pilot's marks and problem reports, a reader's private audit on the Referee-style site, and the
  site's history of when each declaration's meaning changed. All now compare an old hash with the
  `legacy` one, so the first dataset under the rule changes no verdict: on Tau Ceti 8befae0, 2,000
  marks made under the old datasets stay current (none would have without it). An audit verdict and
  a history entry move to the new hash the first time they are read.
- **The pilots move by themselves:** they take the newest extractor release at their next dataset.
  None had moved at the time of writing. What is left is to re-key the old records through datasets
  of their commits, so that they follow the rule rather than the old hash.
- **What stays of semantic_hash:** the content hash, and an independent reference. A graph and a
  hash derived from one walk agree even when the walk is wrong: a constant it misses is missing from
  both. The kernel check (check 2) is the independent verdict that catches that, and it passes.

## 7. Tau Ceti

At 8befae0 (Lean v4.34.0-rc2, 7,432 modules on Mathlib), `ltb-dataset/0` (extractor 0.4.0) against
`ltb-dataset/1` (0.7.1) at the same commit:
- **13,689 more declarations**, all private: 95,688 project declarations instead of 81,999. Most
  are private lemmas, which enter a `meaning` closure only when a statement or a definition's value
  mentions them.
- **Direct `meaning` edges:** 2.0 million in both, 969,000 only before, 803,000 only now. Gone:
  612,000 edges to upstream instances (instances of `Prop` classes, now erased), 265,000 to upstream
  definitions (mostly projections, which now stand for their class: 688,000 new edges to classes),
  28,000 to upstream theorems (proofs inside statements), and 25,000 to project theorems,
  instances and definitions reached only through proofs. New: 1,065 edges into private
  definitions, which were looked through before.
- **Closures 23% smaller:** mean 101.0 declarations against 130.5, median 60 against 64, largest
  2,168 against 3,027. 25,923 declarations lost some project declaration from their closure, 17,435
  gained one (a private definition).
- **Cost:** the rule's walk takes 21 to 28 seconds per part (about 37,000 blocks each, four parts).
  The whole extraction, without the statement and signature facets, takes 2 minutes 34 seconds,
  two parts at a time.

**The kernel agrees here too.** Check 2 along `meaning`: all 95,688 closures of Tau Ceti are enough
for Lean's kernel, and no declaration mentions anything its closure lacks (5 minutes, in four groups
of modules). The first run flagged 7 declarations; both causes were in the check, not in the graph
(dependency-testing.md §9).

**A bug found on the way.** The first run on Tau Ceti did not finish: one part was still in the
walk after 20 minutes. The cause was in the hash's memo: its match arms produced their value with
`return`, which in Lean leaves the function before the result is memoised, so every hash walked the
whole expression tree instead of its graph of distinct subterms. Tau Ceti's F4 root system has terms
of a few hundred distinct subterms that are trees of 10⁸ nodes; LeanMachineLearning has none, which
is why the first measurements did not show it. Fixed in 0.7.1 (MeaningGraph a44b22f), with a test on
a term of 2⁶⁴ nodes over 65 that did not finish before the fix. No hash changed: 0.7.0 and 0.7.1
write the same datasets.

## 8. Statuses on Tau Ceti under version 0

How the statuses of S1 fared on a real library with semantic_hash's hashes (S1 version 0), before
the rule replaced them. Moved here from S1 on 2026-09-28: a spec says what the statuses are, and
the design notes how they behaved.

Between Tau Ceti d3aec47 and 8befae0 (428 commits in 29 hours, same toolchain and Mathlib; datasets
by trust-extract 0.2), taking each of the 77,758 declarations of d3aec47 as if a review had been
made of it there:

| status at 8befae0 | declarations | share |
|---|---:|---:|
| current | 71,880 | 92.4% |
| current, a proof in its closure changed | 4,112 | 5.3% |
| stale underneath | 1,282 | 1.6% |
| stale | 394 | 0.5% |
| renamed | 21 | |
| orphaned (removed) | 69 | |

4,310 declarations were added. Against the source text at both commits:

* **current**: a handful of declarations whose statement text changed are still current, rightly:
  a name written fully qualified, or an attribute added;
* **stale underneath**: 96% read exactly the same, as they should; most are downstream of a few
  rewritten definitions (three rewritten weight tables are among the causes of 810, 319 and 300
  of them);
* **stale**: 153 read differently. 241 read the same: 106 because a `variable` line of their
  section changed (the statement did change: stale is right, but the change is outside the
  declaration's source range, so a page must show the elaborated statement, not only the source),
  and 135 whose section's variables did not change either. 73 of these are in files that did not
  change at all; in the cases examined, a constant they use changed its signature, so that the same
  text now elaborates with other instance or implicit arguments. For a reviewer, these are closer to
  stale underneath: nothing in the declaration was rewritten.

Across a dependency bump, from 8befae0 to c59177e (16 commits, among them the move from Lean
v4.34.0-rc2 to v4.34.0 and a Mathlib bump of 249 commits), of 81,999 declarations: 24,874 current,
35,369 current with a proof in their closure changed, 21,493 (26%) stale underneath, 237 stale.
Only 74 of the 15,945 upstream declarations Tau Ceti rests on were rewritten, and 28 removed; 17,076
of the stale-underneath declarations have only such upstream causes. The largest causes include
real refactors of definitions (`Bialgebra`'s `toBialgHom` now built from `AlgHom.ofClass`), and
changes of signature: `MeasureTheory.eLpNorm` gained an instance argument `[TopologicalSpace ε]`, so
`MeasureTheory.Lp`, whose source did not change, now elaborates with that argument, and its local
hash changed with it (826 declarations rest on it).

## 9. One graph per hash (2026-09-28)

§2 made the `meaning` graph and the meaning hash one walk. Two graphs were still computed apart from
any hash, and are now taken from the walks too (MeaningGraph 8cb71a2, extractor 0.11.0):

- **MeaningGraph's own "meaning" graph.** `Context` computed `dataDeps` by a traversal of its own,
  which skipped only the proofs filling `Prop` fields of a definition's value and read the proofs
  inside helpers: the graph of §1's table. The extractor no longer used it for `meaning`, but it
  was MeaningGraph's API. It is gone: `DeclDeps` is `{statement, meaning, term}`, each the edges of a
  walk (`Walk.edges`), and which constants are declarations is the walk's `Rule`.
- **`term` and the content hash.** `term` came from `Context.depsOf` (the whole value, looking
  through helpers, plus what a notation expands to), and the content hash from a second walk that
  keeps proofs. `term` is now that walk's edges, so the content hash follows it. On the extractor's
  fixture, two kinds of edges went: a notation's expansion (a `source` edge, not something the kernel
  checks), and a constructor's type (the edge goes to the inductive type, whose content covers it).
  The upstream closure along `term` went from 1,548 nodes to 98: it had followed a notation's name
  data into Lean's parser, the String library and `Int` lemmas.

**Checking it.** `evidence-core check-graph --hash content` checks the content hash against `term`
as check 1 does the meaning hash against `meaning`, in one direction: something in D's `term`
closure changed but D's content hash did not (**missed**). The other direction (D's content hash
changed with nothing changed in its closure) needs a local content hash, which datasets do not have.

**Every `term` target is a node** (extractor 0.12.0). A dataset used to keep only `term` edges whose
target was a node, and a lemma used only inside a proof was not one: a change of its proof moved the
content hash of what used it with nothing in the dataset's `term` graph to show it. Now every
declaration an edge points to is a node, in every notion; the lemmas only proofs use are leaves,
which the upstream closure does not start from. One kind of node: which ones statements and meanings
rest on is read off the edges. On Mathlib's probability modules up to `Moments.Variance` taken as a
project (1,154 declarations), that is 1,576 more upstream nodes than the 488 before; on the
extractor's fixture, 95 nodes instead of 36.

**No local content hash.** It would tell a declaration whose own proof was rewritten from one whose
proof uses a lemma whose proof changed, and let check 1 judge the content hash's "unexplained"
direction. Proofs are not reviewed, and the content hash's one use is "only a proof changed", so it
was left out.

**The local hash no longer depends on the split** (`ltb-local/3`). `Walk.localHash` read a helper's
owner off the caller's nodes, so a reference to a helper owned by another declaration counted by name
when that declaration was a node, and was looked through otherwise. With more nodes, the closure
extracted in three parts differed from the one in one part. The owner is now read off the
environment: the longest prefix of the helper's name that is a declaration. On the fixture, two
upstream instances' local hashes moved.

## 10. The graph semantic_hash implicitly follows

semantic_hash has no graph, but its hash defines one: a constant's hash mixes in the hash of every
constant its hashed content mentions (a Merkle hash, like ours), so the constants whose hashes it
mixes in are its edges. This compares that graph, for its proof-irrelevant hash (`runProofIrrel`, at
revision 0496f6d, the one the suite used), with the `meaning` graph of the rule `ltb-meaning/1`.

**What each leaves out.**

- **semantic_hash** hides the bodies of theorems and of `opaque` constants, and nothing else: it
  hashes a constant's value exactly when `ConstantInfo.value?` gives one, which is a definition's
  body. It asks no question of types (it runs without `MetaM`), so a proof that is not a theorem's
  body is hashed like any term. Its README calls this "a deliberately lightweight proof irrelevance".
- **The rule** erases every term whose type is a proposition, wherever it sits: an argument whose
  expected type (read off the type of the function applied) is a proposition, and a let-bound value
  whose type is one. A declaration whose type is a proposition means its statement.

**What semantic_hash's graph has and the rule's does not.** Four kinds of edges, all coming from
proofs:

| semantic_hash follows | example | the rule |
|---|---|---|
| a proof written inline in a statement or a definition's value, which Lean did not lift into an auxiliary theorem | the `h` of `Subtype.mk x h` or of `Classical.choose h`; a proof field `zero_apply _ := rfl` of an instance | erased: it fills a `Prop` argument |
| an instance argument of a `Prop` class, as a term | the instance filling `[IsProbabilityMeasure μ]` or `[NeZero n]` | erased: its type is a proposition |
| the statement of a lifted proof, through the reference to it | `foo._proof_1`, hashed by its proposition | erased: the reference fills a `Prop` argument |
| the body of a definition whose type is a proposition | a `def` or `instance` of a `Prop` type, whose value is a proof | its statement only |

**What the rule's graph has and semantic_hash's does not.** Nothing. Erasing only removes subterms,
and where nothing is erased the two hash the same content: a definition's type and value, a
theorem's or an `opaque` constant's type, an inductive family with its constructors (semantic_hash
also hashes the recursors and their rules, which follow from the constructors). Both cover a
helper's content through the reference to it. Neither sees notation or coercion instances, which the
rule keeps apart as `source` edges. So the rule's `meaning` graph, on constants, is a subgraph of
semantic_hash's.

**Differences that change no coverage.** semantic_hash hashes every constant in its own right; the
rule draws only declarations and looks through helpers, whose content its Merkle hash covers all the
same. Both leave out names, binder names and binder kinds by default (semantic_hash has options to
count them).

**What it did.** The four kinds of edges are proofs, so they are where semantic_hash's hash moved
when only a proof changed: on LeanMachineLearning, a Mathlib bump that removed proof fields from an
instance (the first row) made semantic_hash mark 67 declarations stale underneath, and the rule none
(§5). They are also why its hash could not agree with any graph that erases proofs everywhere (§1).
