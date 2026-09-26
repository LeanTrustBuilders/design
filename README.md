# LeanTrustBuilders: design

Design notes for the [LeanTrustBuilders](https://github.com/LeanTrustBuilders) suite of tools for
trusting Lean formalizations: what a formalized result rests on, who reviewed it, and what is left
to check.

**Where things stand:** [AI_initial_docs/status.md](AI_initial_docs/status.md).

The notes in [AI_initial_docs/](AI_initial_docs/) were written with an AI assistant. Most are
snapshots of 2026-09-25, from before anything was built, with short "since this snapshot" notes
where the proposal has moved.

| note | about |
|---|---|
| [review-tools-comparison.md](AI_initial_docs/review-tools-comparison.md) | Referee, trust and Reviewed-by for Tau Ceti: what each does, and where they meet |
| [dependency-testing.md](AI_initial_docs/dependency-testing.md) | the six ways the tools compute what a declaration depends on, and how each is tested |
| [trusting-definitions.md](AI_initial_docs/trusting-definitions.md) | how a definition can be wrong, and the kinds of evidence that it is right |
| [interfaces-by-audience.md](AI_initial_docs/interfaces-by-audience.md) | who uses these tools, and the interfaces each needs |
| [data-formats.md](AI_initial_docs/data-formats.md) | the data formats of the existing tools, and a common key |
| [reviews.md](AI_initial_docs/reviews.md) | what is reviewed, what a review contains, and how reviews are used |
| [claims-and-importance.md](AI_initial_docs/claims-and-importance.md) | finding what matters in a formalization, and guiding readers to it |
| [suite-design.md](AI_initial_docs/suite-design.md) | the proposal: a suite of small pieces around three shared specifications |
| [meaning-hash.md](AI_initial_docs/meaning-hash.md) | a proposal: derive the meaning hash from the meaning graph's rule, so the two agree by construction |
| [status.md](AI_initial_docs/status.md) | what has been built of it, what has not, and the decisions taken on the way |
