# Reviews: what is reviewed, what a review contains, how it is used

Snapshot of 2026-09-25, with the decisions taken while building it added on 2026-09-26. This note
specifies the review part of the suite proposed in [suite-design.md](suite-design.md): the subjects
of reviews, the content of a review record (its S3 fields), verdicts, lifecycle, and how reviews are
used.

**As built:** the fields are those of S3 in
[LeanTrustBuilders/specs](https://github.com/LeanTrustBuilders/specs/blob/main/S3-evidence.md);
[evidence-core](https://github.com/LeanTrustBuilders/evidence-core) computes statuses, threads and
coverage; [evidence-store](https://github.com/LeanTrustBuilders/evidence-store) takes reviews in from
GitHub issue forms and comments; and a claim's page in
[referee-site](https://github.com/LeanTrustBuilders/referee-site) shows them
([an example](https://leantrustbuilders.github.io/review-sandbox/)). See [status.md](status.md) §6. It draws on the three existing models:
Referee's verdicts with derived coverage, trust's marks and signed certificates, and Reviewed-by's
marks, problem reports and AI labelling (review-tools-comparison.md). The failure modes F1 to F9 are
those of [trusting-definitions.md](trusting-definitions.md) §2, and the personas P1 to P9 those of
[interfaces-by-audience.md](interfaces-by-audience.md) §2.

A review is a judgement about **meaning**. The kernel has checked the proofs; a review answers
"does this declaration say what it is supposed to say?".

---

## 1. What is reviewed

| subject | the question | notes |
|---|---|---|
| **definition** (`def`, `structure`, `class`, `inductive`, `abbrev`) | is this the intended notion? | the core case: F1 to F6 and F9 |
| **theorem statement** | does it state what its name, docstring or paper says? | the statement only, never the proof |
| **instance** | is this the intended structure on that type? | easy to overlook, and where F7 hides (for example the max norm on products) |
| **a link recorded about a declaration** | does this `@[specifies]` theorem really specify `d`? Are `P` and `R` the right property and relation for this characterization? Is this Lean theorem the paper's Theorem 3.2? Does this Stacks tag match? | reviews of *correspondence*. They matter because other evidence is built on these links |
| **name and docstring** | does the text mislead? | Reviewed-by already has this category of problem |

Two things are **not** reviewed here: proofs, which the kernel checked, and code quality, which is
the job of pull-request review (as TauCetiReview does).

### A review is local

A review says: "this declaration is right, *given what the declarations it uses mean*". Coverage
then composes local reviews over the statement closure, as Referee does.

trust's certificates work the other way: one certificate vouches for everything below the
declaration. That cannot say what was actually read. So reviews stay local, and claims about whole
closures are derived from them.

### Two hashes, two kinds of staleness

A review records two hashes of its subject:

- **a local hash:** the declaration's own statement and data, with references by name, like trust's
  `structural-v1` hasher. If it changes, the declaration itself changed: the review is **stale** and
  needs a new look.
- **the deep meaning hash:** the proof-irrelevant semantic hash. If only this one changes, something
  underneath moved: the review is **stale underneath**. That is a lower priority, and the view shows
  which dependency changed.

This is Referee's distinction between a statement change and an indirect invalidation, applied to
reviews.

---

## 2. What a review contains

| field | content | required? | exists today |
|---|---|---|---|
| **subject** | the S1 key: name at a commit and toolchain, local and deep hashes; and the kind of subject (section 1) | yes | Referee: name and hash. trust: name and commit, or hash. Reviewed-by: name and text hash |
| **verdict** | `accept`, `problem` or `question` (section 3) | yes | Referee: accepted or query. trust: trusted. Reviewed-by: `Reviewed-by`, or a problem report |
| **reference** | what the subject was compared with: a paper or book with section, a Stacks tag, a URL, or "the reviewer's own knowledge". Without it, "intended" has no meaning | strongly encouraged | only in free text |
| **what was checked** | a checklist: F1 to F9 plus naming, each marked checked, not checked, or not applicable. Also whether the closure was read, and which evidence was consulted (examples, characterization, junk-value findings) | encouraged | nowhere |
| **caveats** | for example "correct except at `n = 0`", or "correct but less general than the source", each with a failure-mode category | optional | nowhere |
| **rationale** | free text: why it is right, the counterexample, the step that fails | required for AI reviewers and for problems; optional for people, as in Reviewed-by | Reviewed-by's `evidence`; Referee's note; trust's note |
| **reviewer** | a GitHub account, or an AI agent (tool, model, session), or an agent acting through an account; involvement (author of the declaration, contributor, outsider); optional self-declared expertise | yes: never anonymous (decided 2026-09-26; keys come later, for signing) | Reviewed-by: GitHub and agent. trust: key |
| **context** | the dataset commit and the version of the evidence card the reviewer saw; origin (web, issue, CLI, agent run); timestamp | yes | partly, in every tool |
| **links** | supersedes (the reviewer's earlier review), replies to. What resolves a problem is a `status` record naming it | optional | nowhere |
| **signature** | over the canonical bytes, as in trust | optional | trust |

The **checklist** and the **reference** are the fields no current tool has, and they are what make a
review worth something to someone else. "Accepted" says little. "Accepted against Folland §2.3;
conventions and edge cases checked; junk values not checked" says what is left to do.

---

## 3. Verdicts

- **`accept`**, optionally with caveats. Whether an acceptance with caveats counts depends on the
  reader's policy (section 5).
- **`problem`**, with a category: F1 different object, F2 convention, F3 edge cases, F4 junk value,
  F5 vacuous, F6 choice, F7 wrong thing underneath, F9 generality, or a misleading name or
  docstring. The rationale is required, as in Reviewed-by. A problem stays open until it is
  resolved.
- **`question`**, such as "what is this at 0?". Something between a comment and a challenge. It can
  be answered by a person, by an AI grounded in the code, or turned into a formal challenge. An
  answer is a `comment` record replying to it, and the asker or a maintainer marks it answered.

---

## 4. Lifecycle

- **Reviews:** current → stale (the local hash changed) or stale underneath (only the deep hash
  changed) → renewed, or left to lapse. Renewing shows the diff, and takes one click when nothing
  changed in substance.
- **Problems:** open → one of:
  - fixed, by a commit, and ideally confirmed by the reporter or another reviewer;
  - closed as intended: the convention is deliberate, so it gets documented, with an example that
    pins it down;
  - closed as invalid.
- **Renames** keep their reviews, since the meaning hash is unchanged.
- **Generated twins,** such as `@[to_additive]` copies, have different hashes. The attribute that
  generates them can link them, so a review of one is offered for the other, flagged as carried
  over.
- **Withdrawal** by the reviewer, signed if the review was signed, like trust's revocations.
- **As built,** every change of state is a `status` record naming the record it changes: `withdrawn`
  (by its author); for a problem, `fixed` (with the commit), `intended` or `invalid` (by its
  reporter or a maintainer); for a question, `answered`; and `reopened`. A later acceptance or
  problem by the same reviewer supersedes their earlier acceptance of the same declaration. Intake
  checks who may make each change, and a page offers them as buttons.
- **Disagreement:** an acceptance and a problem on the same subject and version. It is shown, not
  averaged away.

---

## 5. How reviews are used

1. **Status on the evidence card:** counts, with people and AI kept apart as in Reviewed-by; current
   or stale; open problems; caveats; and the parts of the checklist nobody has checked. "Nobody has
   checked edge cases" is as useful as a count. *Built on the claim's page, per declaration.*
2. **Coverage, under a policy the reader chooses.** A claim is covered when every declaration in its
   statement closure has a current acceptance **that counts under the reader's policy**. The policy
   says whose reviews count: all of them, people only, a trust list (as in trust), signed ones only,
   or ones whose checklist includes F4. **Reviews are data; trust is a policy that each reader
   applies.** The same records then serve a cautious referee and a relaxed user. *Built: whether AI
   agents count, reviews made before something underneath changed, acceptances with caveats,
   authors' own reviews, and whether upstream declarations must be reviewed too. Trust lists and
   signed-only are not.*
3. **The review queue:** the statement closures of claims first, then what is most used, stale,
   uncovered, without evidence, in disagreement, or asked about. It can match subjects to reviewers'
   declared areas.
4. **Changes:**
   - the pull-request bot lists the reviews a change makes stale and notifies their reviewers, like
     `CODEOWNERS` but for meaning;
   - policy gates can block a release whose claims lose coverage.
5. **Reuse across projects.** Keyed by meaning hash, a review of a Mathlib definition applies to
   every project where that definition hashes the same. Trusting Mathlib, today all or nothing
   (Referee's `--trust mathlib`), becomes graded: which of the Mathlib definitions your statements
   use have current reviews.
6. **Feeding the strengthening loop.** This makes reviews more than opinions:
   - a checkable claim in a rationale ("I checked `f 0 = 1`") becomes a proposed `example`. If it
     proves, part of the review is now checked by the kernel; if it fails, the review is
     contradicted;
   - a problem leads to a fix, and a question to a challenge, then to an example or a theorem.
7. **AI reviews** do first-pass triage (ordering the queue, pre-filling checklists) and gather
   evidence. By default they don't count toward coverage, unless the reader's policy says so. They
   are calibrated by meta-review, as TauCetiReview's A/B judging does. Their rationale is required.

Reviewers are not ranked by reliability. Disagreements are shown instead, following Referee's rule
against drawing conclusions about people from data that doesn't support them.

---

## 6. Who reviews what

| reviewer | best at | typical subjects |
|---|---|---|
| domain expert without Lean (P1) | F1 to F3, F9, correspondence | definitions, claims, links between paper and Lean, through the math-language layer |
| Lean expert (P2) | F4 to F7 | instances, junk values, choices, the closure |
| author (P3) | context and fixes | answering problems and questions. Their reviews of their own code are recorded, but flagged as such |
| AI agent (P9) | covering the checklist, collecting evidence | everything, labelled as AI |
| referee of a paper (P6) | whether claims match the paper | claims and their statement closures |

---

## 7. Open questions

1. **Granularity.** Is a structure one subject, or one per field? A `def` with auxiliary
   definitions? A whole file reviewed at once, with bulk intake? *Open; a pull request can add many
   records at once.*
2. **Acceptances with caveats in coverage:** count them, count them under a policy, or never?
   *Settled: under the reader's policy, counted by default.*
3. **Anonymous reviews:** keep them private (Referee's audit mode), or allow publishing them too?
   *Settled: private only. A published record names a GitHub account or an AI agent.*
4. **Checklist vocabulary:** F1 to F9 as a closed list, or extensible through the facet registry
   (suite-design.md §3.1)? *For now closed: F1 to F9 (without F8, drift, which is not something a
   reviewer checks) and naming.*
5. **Withdrawn or superseded reviews** in federation: how long nodes keep them. *Open, with
   federation.*
