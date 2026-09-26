# One notion of meaning: the graph and the hash from the same rule

Proposal of 2026-09-26. The suite uses two computations of what a declaration's meaning depends on:
MeaningGraph's `meaning` graph, and semantic_hash's proof-irrelevant hash. They disagree, and the
suite uses them together. This note proposes deriving the meaning hash from the graph's own rule,
so that for any choice of rule the two agree by construction. It is a companion to
[dependency-testing.md](dependency-testing.md) (§9, the self-checks) and to S1, the declaration key,
in [LeanTrustBuilders/specs](https://github.com/LeanTrustBuilders/specs).

---

## 1. The problem

The views and the evidence core use the two together:

- **The graph** says what a statement rests on. Coverage, the trust surface, "what it rests on" on
  a claim's page, and the name given for "changed underneath: X" all come from it.
- **The hash** says whether a review still applies: current, stale underneath, or stale.

So a reader can be told that a review of D is current while the graph shows that something D
rests on changed meaning. Or they can be told that D changed underneath with nothing in its graph
to point to.

**Measured.** Between two Tau Ceti datasets (8befae0 and c59177e, across a Lean and Mathlib bump,
81,970 declarations in both):
- **4 declarations** went stale underneath with nothing in their graph closure changed;
- **335 declarations** kept their meaning hash although their graph closure reached a declaration
  whose meaning changed.

**Why they differ.** The two draw different lines (dependency-testing.md §6, shortcoming 5):

| | `meaning` graph (MeaningGraph) | meaning hash (semantic_hash, proof-irrelevant) |
|---|---|---|
| proofs in a definition's own value | skipped where they fill a `Prop` parameter of a function returning a structure; kept elsewhere | followed when written inline |
| lifted proofs (`_proof_N`) | looked through, **whole proof included** | hashed by their statement only |
| helpers (`match_N`, private declarations, …) | looked through, **whole value included, proofs too** | hashed as constants in their own right |
| notation and coercion instances | included (source dependencies) | not seen: not in the elaborated term |
| constructors, recursors | looked through to their type | hashed with their type |
| beyond the project | leaves in the graph | the hash is deep: it covers everything |

The second and third rows are the likely source of the 335 cases: the graph reads the proofs inside
helpers, which the hash does not. It is confirmed in the code, but not yet traced for each case.

## 2. The principle

Fix one **rule** for what a declaration's meaning depends on, and derive both the graph and the
hash from it, in the same walk:

- **nodes:** which constants are declarations in their own right. The others are helpers, whose
  content counts as part of the declaration that uses them;
- **erasure:** which parts of a declaration are not meaning (proofs, by some definition of "proof");
- **the local content** of D: its type, and its value if it is not a proof, erased, with helpers
  inlined;
- **the edges** of D: the nodes its local content refers to;
- **the meaning hash** of D: a hash of its local content in which each reference to a node is
  replaced by that node's meaning hash. This is a Merkle hash over the graph, with each strongly
  connected component (mutual definitions) hashed as a block;
- **the local hash** of D: the same content, with references by name. It says whether D itself
  was rewritten.

**Consequence.** For every rule, a declaration's meaning hash changes exactly when its own local
content changes, or when the meaning hash of a node it refers to changes. By induction, that is
exactly when something in its graph closure changed (up to 64-bit collisions). "Stale underneath"
and "what it rests on" then agree by construction, and the declarations to blame for a
stale-underneath review are exactly the closure members whose local hash changed.

**That means building our own hash.** It can live in MeaningGraph, next to the graph it follows: a
module `MeaningGraph.Hash`, computed in the walk `declDeps` already does.

## 3. The decisions to make

Each is a choice of rule. The graph and the hash follow whatever is chosen.

1. **Erasure.**
   - *Every proof subterm* (`isProof` in `MetaM`, as ChallengeGen's flat printer does). This is
     principled: by proof irrelevance, no proof affects meaning.
   - *The present mask* (`Prop` parameters of functions returning a structure). Cheap and
     context-free, but it keeps some proofs, which then become edges.

   Either way, applied **everywhere**, helpers included: today the mask applies only to a
   declaration's own value. Recommended: every proof subterm, if its cost at Mathlib scale is
   acceptable (to measure). The case that motivated the mask's restriction was
   `IsPreBrownianReal.mk X h := h.exists_continuous_modification.choose`. Under full erasure its
   meaning is "a choice of an object satisfying the predicate", which is right by proof
   irrelevance. What such a definition really depends on is a *choice*, and that is for the choice
   report of trusting-definitions.md §3.12, not for the graph.
2. **Nodes.** Which constants are declarations and which are helpers.
   - The present rule (`isAuthored`) makes private declarations helpers. So their content is
     inlined into their users, and nobody can review them.
   - The alternative is completion's rule, which trust uses.

   Recommended: keep `isAuthored`, but make private declarations nodes. They are written by a
   person, and a reviewer should see them.
3. **Source dependencies** (notation, coercion instances) are not meaning: the elaborated term does
   not depend on them. Recommended: move them out of `meaning` into a fourth notion, `source`, which
   is what standalone files need. The import-visibility filter, whose purpose is to remove the
   impossible edges source recovery can create, then matters only there.
4. **Past the project.**
   - The hash must be deep, so it is computed over the whole closure, upstream included. That is
     what semantic_hash does over the whole environment.
   - The graph may still stop at the project. The hash of an upstream leaf is then its deep hash
     under the same rule, so the two stay consistent.
   - Cost: hashes for the upstream closure of a library on Mathlib. These can be cached per module
     and per rule, keyed by the `.olean`'s hash.
5. **The local hash's sensitivity** (S1's known limit). A declaration whose text is unchanged can
   elaborate with different implicit or instance arguments when a constant it uses changes its
   signature. The local hash then reads it as rewritten. A variant of the local hash that erases
   implicit and instance arguments would make that read as "changed underneath". Decide with the
   rule.

## 4. Several rules, several hashes

Readers may reasonably want different rules: our `meaning`, trust's closure, a rule that keeps the
proofs that use choice. Each rule is then a **profile**: a named, versioned set of the choices of
§3.
- A dataset says which profiles it computed, and for each, its edges and its hashes.
- S1's `hasher` field names the profile a record was keyed under.
- A record made under one profile is compared only with the hashes of the same profile; the others
  read as `incomparable`, as they already do.

Comparing profiles is then a tool of its own. On a corpus, it says how the graphs and the
staleness they imply differ between two rules: which declarations gain or lose dependencies, and
which reviews would go stale under one and not the other. That is how the decisions of §3 can be
made on evidence rather than argument, and it is what dependency-testing.md §9's check 1 becomes.

## 5. What stays of semantic_hash

- **The content hash.** trust's certificates are keyed by semantic_hash's proof-relevant hash at
  its pinned revision, which is the datasets' `content` hash. Keeping it keeps trust's
  certificates matchable.
- **An independent reference.** A graph and a hash derived from one walk agree even when the walk
  is wrong: a constant it misses is missing from both. semantic_hash walks terms with its own code.
  Where the two disagree about whether a declaration changed, beyond what the documented rule
  differences explain, one of the walks has a bug. The kernel check of dependency-testing.md §9
  (check 2) is the other independent verdict, and it becomes more important, not less, once graph
  and hash share a walk.

## 6. Consequences

- **S1:** the meaning and local hashes are defined by a profile of MeaningGraph, not by
  semantic_hash; the content hash stays semantic_hash's. This is a new version of S1.
- **S2:** a `source` notion is added, and `meaning` changes meaning. Datasets name their profile.
  This is a new version of S2.
- **Migration.** This is open question 2 of suite-design.md §9, made concrete. For a transition,
  datasets carry both the old and the new meaning hashes. A record keyed by an old hash is
  resolved through a dataset of its own commit that has both. The pilots hold about 630 records
  today, so the switch is cheapest now.
- **The checks** (dependency-testing.md §9):
  - graph against hash becomes an invariant test, which must find nothing;
  - comparing profiles becomes the measuring tool;
  - the kernel check and the comparison with semantic_hash are the independent ones.
