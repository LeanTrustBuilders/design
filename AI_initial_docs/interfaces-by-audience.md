# Front ends and interfaces, by audience

Snapshot of 2026-09-25. Which front ends and interfaces do trust tools for Lean libraries need, and
for whom? This note sorts the people (and agents) who use them along several dimensions, describes
the personas those combine into, lists the interfaces that would serve them, and suggests an order
for building them. It is a companion to [review-tools-comparison.md](review-tools-comparison.md),
[dependency-testing.md](dependency-testing.md) and
[trusting-definitions.md](trusting-definitions.md). The failure modes F1 to F9 are defined in
trusting-definitions.md §2.

Two points shape the rest of the note:

- **Lean expertise and domain expertise are separate dimensions.** The person best placed to judge
  whether a definition is the intended notion is often a domain expert who doesn't read Lean.
- **What a person is trying to do determines the interface** more than any other dimension.

---

## 1. Dimensions

| dimension | range | why it changes the interface |
|---|---|---|
| **Lean expertise** | none · reads Lean · writes Lean · knows how elaboration works | decides whether formal text can be the main view, and whether the reader can spot junk values, instances and coercions unaided |
| **domain expertise** | area expert · mathematician in another area · non-mathematician | decides who can judge F1–F3 and F9 (wrong object, convention, edge cases, generality) |
| **involvement** | author · contributor · outsider | outsiders need context and a way in; authors need to see what's weak and to act on it |
| **goal** | review · use or build on · referee a paper · maintain · explore or learn · report status | each goal comes with a different main question (section 2) |
| **human or AI** | person · agent | agents need structured data and a way to submit, not pages |
| **scope** | one claim · one definition · a change · a whole library | sets the unit of a page and of a queue |
| **frequency** | once · returning | a returning reader needs memory: what they've read, and what changed since |
| **stance** | adversarial auditor · cooperative author · consumer who wants a verdict | find holes, fix holes, or "can I rely on this?" |
| **time budget** | 5 minutes · hours · ongoing | a summary card, a workspace, or notifications |
| **accountability** | anonymous · identified · signed | whether a judgement counts for others. More accountability means more friction: domain experts won't use GPG |
| **where they already work** | browser · editor · CI and pull requests · Zulip · terminal or agent loop | meet people there rather than asking them to come to you |

---

## 2. Personas

The dimensions combine into the people who actually show up:

| persona | profile | main question | what they need | where |
|---|---|---|---|---|
| **P1 domain expert without Lean** | area expert, outsider | "is this *the* notion from my field?" | the definition in math language next to the source's; conventions, edge cases, junk values and generality **stated in math words** ("here 1/0 = 0", "the norm on ℝ² is the max norm"); examples and values; a way to give a verdict or report a problem without Lean | browser, no install |
| **P2 Lean expert, not in the area** | Mathlib-style reviewer | "is anything off in how it's written?" | the fully explicit form; resolved instances; the dependency closure split into statement and proof; junk-value findings; axioms and `sorry`; a jump to the source or the editor | browser, editor |
| **P3 author or maintainer** | involved, usually expert in both | "where is my library weak, and what did my change break?" | definitions without evidence, ranked by use; missing examples; reviews made stale by a change; open problems; checks in CI | editor, pull requests, CI |
| **P4 returning reviewer** | either expertise, involved or not | "what should I review next?" | a ranked queue, claims' closures first; checklists by failure mode; progress and coverage; what changed since the last visit | web workspace with state |
| **P5 user of a result** | expert or not, outsider | "can I rely on this, and what do I have to trust?" | a verdict-style summary: the exact statement and hypotheses, the definitions it rests on and their evidence, upstream packages, `sorry` and axioms, who reviewed it | web summary, badge |
| **P6 referee of a paper** | domain expert, may or may not read Lean, one-off, deep | "does the formalization prove what the paper claims?" | claims first; the paper's theorems mapped to Lean statements; the gap between them (scope, generality, literature dependencies); an exportable report | web, claims-only build |
| **P7 explorer or learner** | non-expert, outsider | "what is this about?" | concepts, examples, a map of the ideas. Low priority for trust, but it reuses the same examples | web, in the style of blueprints or LeanExposition |
| **P8 manager or funder** | neither, outsider | "how far along is this, and how solid?" | honest counts: claims, coverage, open problems, stale reviews, trends. Count what's missing, not what's done, as Referee does | dashboard |
| **P9 AI agent** | reviewer, contributor or consumer | "what needs doing, and how do I submit it?" | machine-readable evidence cards and queues; a submission channel that requires evidence and labels the result as AI | CLI, JSON, MCP, GitHub comments |

**The pair that matters most for trust is P1 with P2.**
- A domain expert without Lean can judge meaning (F1–F3, F9) but can't see the Lean pitfalls
  (F4–F7).
- A Lean expert outside the area sees the pitfalls but can't judge meaning.

So an interface for P1 has to translate into mathematics what P2 would notice in the Lean.

---

## 3. Interfaces

| interface | serves | exists today | missing |
|---|---|---|---|
| **I1 definition or claim page with an evidence card**, in two layers: a math-language summary with a conventions panel, then the formal anatomy, closure and proofs | P1, P2, P5, P6 | Referee pages (the formal side, with statement anatomy); trust-web hover cards; the Reviewed-by page (source text only) | the math-language layer; a conventions panel produced from the analyses; the evidence card (trusting-definitions.md §5) |
| **I2 review workspace**: queue, checklists by failure mode, verdict and problem controls, stale detection, coverage, disagreements | P4, P1, P2 | Referee's audit controls (private, one reader); Reviewed-by's issue forms (public, no queue); trust's marks and certificates | a shared queue; structured checklists; a flow for people without Lean; a view of disagreements |
| **I3 editor integration**: hovers showing a definition's evidence status; linters for junk values, generality and missing examples; code actions such as "add `@[specifies]`" or "create example stubs" | P3, P2 | the JunkValues linter; Reviewed-by's review lines in docstrings, a static version of the hover | everything else |
| **I4 pull-request bot and CI gates**: which definitions a change invalidated, including indirectly; which reviews it made stale; which claims are affected; policy checks | P3 | Referee's revision diff, on the site only; `trust check` | a comment on the pull request; policy gates on evidence |
| **I5 dashboard** | P8, P3 | Referee's landing page, partly | trends; a view across projects |
| **I6 machine interface**: query and submit over the same data | P9 | trust's index files; Referee's `data.json`; Reviewed-by's comment lines | a query API or MCP server; submissions that require evidence |
| **I7 questions**: "what is this at 0?", answered by an AI grounded in the code, and turned into a proved `example` when possible | P1, P5, P7 | nothing | all of it. It turns a non-expert's questions into evidence |
| **I8 paper-to-formalization map** | P6 | Referee's Claims page; blueprints | the paper's statements side by side with the Lean ones, with the gap in scope stated |

**Since this snapshot** (2026-09-26; see [status.md](status.md)):
- **I1:** a claim's page, with what the claim rests on and each declaration's evidence and review
  threads, and a Referee-style site from the suite's datasets. Both have the formal layer only; the
  math-language layer and the conventions panel are not built.
- **I2:** partly, on the same page. It has a "review next" list, the failure modes nobody checked,
  coverage under the reader's policy, and buttons to review, report a problem, ask a question,
  withdraw or resolve, which open prefilled GitHub issue forms. It has no personal queue or
  progress.
- **I6:** partly. Datasets and evidence stores are files, evidence-core has a command line, and
  agents submit and comment through evidence-store's command line. There is no query API or MCP
  server.
- **I3, I4, I5 and I7** are not built.

---

## 4. Design principles

- **One data layer, many views.** Every interface above reads the same evidence. Views are cheap
  only once that shared data model exists.
- **Show both layers, force neither.** Never hide the Lean from experts; never require it of domain
  experts.
- **Surface what the reader can't see.** Junk values, instances, generality and the closure are
  invisible to P1 in Lean syntax, so they have to be stated in math words. The analyses in
  trusting-definitions.md §3.8 and §3.12 are where those statements come from.
- **Label every statement by its backing:** checked by the kernel, computed by a tool, judged by a
  person, or judged by an AI. This matters most for any math-language summary.
- **Match friction to the audience.** Signed certificates for accountable reviews; a browser form
  with GitHub identity for experts without Lean; a structured API for agents.
- **Keep personal state per reader, and evidence shared.** Progress and queue position belong to
  the reader. Reviews, problems and challenges belong to everyone.

---

## 5. Tensions to settle

- **Math-language summaries help P1 and P5, but can mislead.** Generate them from structured data
  where possible, such as the conventions panel built from analyses. Label any AI paraphrase, and
  show the formal text beside it.
- **Private, public or signed reviews.** Referee, Reviewed-by and trust each chose differently, and
  each choice suits some personas. A shared store needs all three levels of accountability.
  *Settled on 2026-09-26: private judgements stay in the reader's browser; published ones name a
  GitHub account or an AI agent, and are never anonymous; signing comes later.*
- **Static site or service.** Static sites are easy to host and easy to trust. Queues, identity,
  questions and agents need a service.

---

## 6. Suggested order

1. **I1, the definition and claim page,** with the evidence card and the conventions panel in math
   language. It serves the most personas, including the core pair P1 and P2.
2. **I2, the review workspace,** over a shared store, including a flow for people without Lean.
3. **I6, the machine interface.** It is cheap once the data exists, and libraries like Tau Ceti
   need it for AI reviewers.
4. **I4, the pull-request bot,** then **I3, editor integration,** for authors.
5. **I7, questions; I8, the paper map; I5, the dashboard.**

In terms of the types of tool in review-tools-comparison.md §9, every interface here is type 3 (views)
or type 4 (storage and handling, for I2 and the submission side of I6). All of them depend on the
shared data layer that trusting-definitions.md §5 calls for.
