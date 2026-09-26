# Data formats across the projects

Snapshot of 2026-09-25. This note surveys the data formats used by the projects reviewed in
[review-tools-comparison.md](review-tools-comparison.md), with the pros and cons of each, the
problems they share, and recommendations. The notions of dependency it refers to are defined in
[dependency-testing.md](dependency-testing.md) §2.

The projects use about twenty formats, which fall into four groups: data stored in the Lean code,
data derived from the compiled library, stores of human judgements, and metadata supplied by
projects. The problem they share is that **no two stores identify a declaration by the same key**.

Repositories read at:

| repository | commit |
|---|---|
| [LeanMachineLearning/exposition](https://github.com/LeanMachineLearning/exposition) (Referee) | `bb51dc2` (2026-09-22) |
| [chrisflav/trust](https://github.com/chrisflav/trust) | `7fdbd6a` (2026-09-12) |
| [CBirkbeck/tauceti-reviewed-by-test](https://github.com/CBirkbeck/tauceti-reviewed-by-test) (Reviewed-by) | `63a81e6` (2026-09-24) |
| [mathlib-initiative/semantic_hash](https://github.com/mathlib-initiative/semantic_hash) | `0496f6d` (2026-08-07) |
| [mathlib-initiative/aftk](https://github.com/mathlib-initiative/aftk) | `17d641f` (2026-08-29) |

---

## 1. Data stored in the Lean code

| format | project | contents | pros | cons |
|---|---|---|---|---|
| `@[specifies]`, `@[characterization]` | Characterization | links from a theorem to the definition it specifies or characterizes, stored in an environment extension in the `.olean` | checked when compiled: a typo is a build error. Versioned with the code, so it can't go stale silently | reading it needs a Lean process on the exact toolchain, with the package linked in: extensions are matched by name and silently dropped otherwise. Changing it means a code change and a pull request. It can't hold judgements by outsiders |
| `@[junk_value]` | JunkValues | junk-value rules, recorded on the theorems that prove them | as above; the rule is itself a proved theorem | as above |
| `@[stacks]`, `@[lmfdb]` and the other cross-references | Mathlib (`CrossRefAttribute`) | the tag, in an environment extension, plus a link appended to the docstring | a person sees it in the docs, and a program can read it | as above, for a program |
| review lines in docstrings, and git trailers | Reviewed-by, in the fork `CBirkbeck/TauCeti` | review and test counts and a link in docstrings; one `Reviewed-by:` trailer per mark in the commit message | visible where people read code; trailers are a convention reviewers know | a copy of the ledger that a daily job must resync; counts only; a pushed trailer can't be corrected |

---

## 2. Data derived from the compiled library

### Referee's `data.json`

**Encoding.** One JSON document.
- Format version 14; a reader accepts versions back to 11.
- Repeated subtrees are stored once, in a numbered table (the `InternedData` envelope), and
  referenced by index.
- Each declaration has about 30 fields: signatures, docstrings (as Verso `Block Manual` values),
  source, kind, `sorry` and axioms, specification and characterization links, semantic hashes,
  direct dependencies of three kinds (`deps`, `typeDeps`, `dataDeps`), reverse dependencies, and
  the statement split by binder role. Transitive closures are recomputed on load, not stored.
- The file also holds the package graph, the upstream declarations that statements mention,
  `formalization.yaml`, and the scope of the build.

**Pros.**
- Self-contained: the site renders from this file alone, without Lean.
- An explicit version, with a minimum readable version.
- The encoding round trip is proved (`Proofs/Collect.lean`).
- Storing direct edges only removed most of the file's size: stored closures were 69.9% of it on
  `Mathlib.Analysis`.

**Cons.**
- **Monolithic:** it is loaded whole. `build-site` peaks at 10.9 GB for 93k declarations.
- **Opaque to other tools** without Referee's decoder, because of the shared-subtree table.
- It embeds Verso's `Block Manual` type, so every reader depends on Verso's JSON encoding.
- One project only, and keyed by name only.
- It doesn't record the toolchain or the library commit it was built from.

### Referee's `provenance.json`

**Encoding.** An append-only ledger: the list of revisions, oldest first; for each declaration, the
revision at which its meaning last changed; the latest `git blame` for each declaration; and a flag
for a tree with uncommitted changes. Kept on its own git branch.

**Pros.** Small: about one integer per declaration per fact. Built by a pure function whose
properties are proved.

**Cons.** Referee-specific. Requires semantic hashes. Its resolution is only that of the build
cadence.

### Extracted files (Referee, ChallengeGen)

**Encoding.** One `.lean` file per declaration, plus highlighting JSON from SubVerso.

**Pros.** The unit a reader checks is real Lean, and compiling it is a test.

**Cons.** Large, and tied to one toolchain.

### trust's index

**Encoding.** A directory:
- `meta.json`: schema version, repository, commit, toolchain, counts, flags saying what the edges
  include (`hasBodyEdges`, `hasProofEdges`), whether code and hashes are present, the hasher, and
  the edge format;
- `decls.jsonl`: per declaration, an id, name, module, kind, whether it is a proof, and the hash;
- `stmt-edges.bin` and `body-edges.bin`: little-endian 32-bit integer pairs;
- `code/<n>.jsonl`: rendered declarations with UTF-16 constant ranges, 2000 per shard;
- `marks.json`: human judgements, with protection status resolved.

**Pros.**
- **Built for scale.** The binary edges load straight into an `Int32Array`; code shards load
  lazily; everything streams. It handles Lean core and Mathlib.
- `meta.json` says what the files mean.
- Hosted statically, from a branch, a release or a folder.

**Cons.**
- Ids are per export, so two indexes can't be compared by id, only by name or hash.
- The binary files are unreadable without `meta.json`, and whether edges include proofs is
  recorded only there.
- Little data per declaration: no `sorry`, axioms or specification links.
- Schema version 1, with no migration yet.

### semantic_hash's export

**Encoding.** JSONL: one `{"name", "hash", "proofIrrelHash"}` object per line. The `duplicates`
command writes `{"hash", "count", "names"}`.

**Pros.** Trivial to produce, stream, and join by name.

**Cons.** **No header.** The toolchain, the semantic_hash revision, the hashing options and the
imports are not recorded, so hashes from different setups can be compared without anyone noticing.

### aftk's query output

**Encoding.** Tab-separated `module  declaration` rows, or JSONL records carrying the query, a
status, the scope, the results, and errors with codes and candidates.

**Pros.** Tab-separated for shells; explicit status and errors for programs; deterministic order.

**Cons.** Answers to queries, not a dataset. Flat sets without edges. No version field.

### Reviewed-by's declaration index

**Encoding.** `data/declarations.json`, generated and not committed: declarations with their text
hash, source and line range, and the examples found. For the site: `search.json` as compact
positional arrays indexing module and keyword tables, `docs.json`, and one JSON file per module.

**Pros.** Simple; loaded per module; fits GitHub Pages.

**Cons.** Built from regexes over source text. A 48-bit text hash. The positional arrays follow an
unwritten schema.

---

## 3. Stores of human judgements

### Referee's audit state

**Encoding.** `localStorage` in the reader's browser, exported as:

```json
{"version": 1, "project": "…", "dataId": "…", "exportedAt": "…",
 "verdicts": {"<name>": {"verdict": "accepted", "note": "…", "at": "…", "meaning": "<hash>"}}}
```

`meaning` is the proof-irrelevant semantic hash. The page can also generate a Markdown report.

**Pros.** Private; no infrastructure. The hash makes an exported file check itself against a later
build, with no access to the build it was made against.

**Cons.** Unauthenticated, by design. Lives in one browser. Keyed by name. No reviewer identity, and
no merging of two readers' files.

### trust's `trust-marks.json`

**Encoding.**

```json
{"version": 1,
 "trusted": [{"name": "…", "commit": "…", "note": "…"}],
 "characterizations": [{"definition": "…", "theorems": ["…"], "note": "…"}],
 "protected": [{"name": "…", "snapshots": [{"commit": "…", "hash": "…", "hasher": "…"}]}]}
```

**Pros.** Plain JSON meant for git, easy to diff. Pinned to a commit. The hasher is recorded per
snapshot.

**Cons.** Trusted marks are keyed by name and commit, not by hash, so they don't follow meaning. No
identity: whose marks these are is whoever commits the file. Characterizations are not checked.

### trust's certificates

**Encoding.**
- A claim has eight string fields: `asserted`, `commit`, `decl`, `hash`, `hasher`, `note`, `repo`,
  `toolchain`.
- Its **canonical bytes** are those fields in alphabetical order, as compact JSON with
  `JSON.stringify` escaping.
- An entry is `{claim, signature, key, fingerprint, hints}`, with an armored PGP signature and
  public key.
- Around them: revocations, bundles with a cursor, a node descriptor, the `trust/1` protocol, and
  SQLite storage on the server.

**Pros.**
- **Each entry can be verified on its own:** the key travels with it.
- The canonical form is pinned by conformance vectors that every implementation must pass.
- Keyed by hash, so a certificate applies in any repository where the declaration still hashes the
  same.
- A versioned protocol.

**Cons.**
- Heavy: PGP, and about 2 KB of key per entry.
- The hash is proof-relevant and tied to a pinned semantic_hash commit.
- Hard for people to read.
- No field says what was checked, although a certificate vouches for everything below the
  declaration.

### Reviewed-by's ledgers

**Encoding.** One JSONL file per kind of record. Each record is tagged with a schema
(`reviewed-by/v1`, `tests/v1`, `named/v1`, plus problem reports and suggested tests) and has:
`decl`, the text `hash`, the Tau Ceti commit, `by` (a GitHub login), `kind` (person or agent),
`agent`, `evidence`, `source` (the issue or comment), and `at`. Records are written by a bot from
issue forms and from a comment-line grammar (`Reviewed-by: <decl> — <evidence>`), with an HTML
comment marking AI agents. They are exported as `reviews.json`.

**Pros.**
- Append-only, versioned in git, easy to diff and read.
- A schema tag on every record.
- Traceable to the issue or comment it came from.
- Identity from GitHub, with AI marks labelled.

**Cons.**
- A text hash.
- One writer at a time.
- Specific to Tau Ceti.
- The comment grammar is fragile to parse.
- No signatures: it relies on GitHub and on the repository's admins.

---

## 4. Metadata supplied by projects

| format | contents | pros | cons |
|---|---|---|---|
| **`formalization.yaml`** (Palomar registry) | main results, literature dependencies, scope, links to Comparator configs | an existing community standard; the project's own statement of intent | YAML, which Referee reads with a partial parser that can misread silently; asserted, not checked; declaration names drift |
| **Comparator config** (JSON) | challenge module, solution module, theorem names, allowed axioms | minimal and precise; a tool acts on it directly | per module; no metadata |

---

## 5. What cuts across all of them

| issue | state today |
|---|---|
| **identifying a declaration** | Names everywhere. Hashes differ: Referee has both variants, optionally; trust's index has an optional hash, and its certificates a required proof-relevant one; Reviewed-by has a 12-character text hash. trust's integer ids exist within one export only. **The only key common to all is a name at a commit.** |
| **versioning** | Referee: `data.json` 14 (readable from 11), audit file 1. trust: index schema 1, a marks version, protocol `trust/1`. Reviewed-by: a schema tag per record. **None** for semantic_hash or aftk. |
| **recording what produced the data** | trust's `meta.json` records the commit, toolchain and hasher; each Reviewed-by record records the Tau Ceti commit. Referee records commits in the provenance ledger but not in `data.json`. **semantic_hash records nothing.** |
| **what a dependency means** | Referee keeps three kinds in separate fields; trust records flags in `meta.json`; aftk has one flat notion. Only trust says in the file which notion it uses. |
| **scale** | Monolithic JSON (Referee), against streamed, binary and lazily loaded files (trust), against files loaded per module (Reviewed-by). |

---

## 6. Recommendations

1. **A common key for a declaration.** Its name at a given commit and toolchain, plus the
   proof-irrelevant semantic hash, recording the semantic_hash revision and the variant. Every
   judgement store should carry this key.
2. **A header on every data file.** Record the producing tool and its version, the toolchain, the
   library commit, the hasher details, the notion of dependency (dependency-testing.md §2), and the
   scope. trust's `meta.json` is the model; semantic_hash's export needs it most.
3. **Separate a neutral shared dataset from each application's cache.** trust's layout is the best
   starting point: JSONL declarations, edge files, and a meta file. Extend it with one edge file
   per notion of dependency, `sorry` and axioms, and specification links. Referee's `data.json`
   stays Referee's own cache, since Verso blocks and shared-subtree tables are rendering concerns.
4. **One record format for judgements, with three levels of identity.** Start from Reviewed-by's
   JSONL records with a schema tag, keyed as in recommendation 1, with a label for how each record
   is backed (see trusting-definitions.md §5). Make identity optional:
   - anonymous, as in Referee;
   - GitHub, as in Reviewed-by;
   - signed, with trust's canonical claim and signature.

   The three stores then become three writers of one format.
5. **Keep checked annotations in the code, and export them.** Put `@[specifies]`,
   `@[characterization]`, `@[junk_value]` and cross-reference links into the shared dataset. Then a
   reader without Lean, or an agent, can use them without linking the Characterization package
   into their own process.

**Since this snapshot** (2026-09-26): these recommendations are the specifications in
[LeanTrustBuilders/specs](https://github.com/LeanTrustBuilders/specs).
- **The key and the headers:** S1 is the key of recommendation 1, with a third, local hash added.
  Every dataset has a `meta.json` header, as recommendation 2 asks.
- **The neutral dataset** of recommendation 3 is S2.
- **One record format** (recommendation 4) is S3, with one change: records are never anonymous.
  They name a GitHub account or an AI agent, and signing comes later.
- **Annotations** (recommendation 5) are exported as facets for `@[specifies]`,
  `@[characterization]` and the suite's other attributes. `@[junk_value]` and Mathlib's
  cross-references are not yet.

Reviewed-by's ledgers, Referee's audit exports and trust's marks convert into S3 (evidence-core's
`migrate`), and a dataset converts into trust's index (`trust-site trust-index`). Referee's
`data.json` has no converter: the extractor replaces its producer. See [status.md](status.md).
