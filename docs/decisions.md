# Decision Log

Questions we faced, what we chose, why, and what we rejected. The PRD holds
*requirements*; this holds the *reasoning trail*. A `Status:` line appears
only where a decision is not yet final.

Last updated: 2026-05-19.

---

## Project: RAG vs LoRA

Q: What to build next — a RAG system or LoRA fine-tuning of a larger model?
Chose: RAG first; LoRA deferred to a later project.
Why: In demand for hiring, builds directly on existing FastAPI + pipeline
skills, shippable in 2–3 weeks.
Rejected: LoRA-first — deeper ML but slower to a demonstrable result and less
immediately job-applicable.

## Domain: football clubs

Q: What corpus domain — technical papers, financial/legal filings, or football?
Chose: Football club histories.
Why: User knows the domain and can hand-verify answers (critical with no eval
harness yet); questions are naturally complex enough to stress retrieval.
Rejected: technical papers / SEC filings (less personally verifiable);
Wikipedia-dump / FAQ corpora (too easy, teaches nothing about chunking).

## The five clubs

Q: Which clubs?
Chose: Real Madrid, Barcelona, Manchester United, Liverpool, Arsenal.
Why: Different eras, tactical identities, and rivalries — naturally produces
cross-club comparative questions, the hard multi-document retrieval case.

## Data source

Q: Where does content come from?
Chose: Wikipedia via the `wikipedia-api` library.
Why: Returns clean text, no scraping needed.
Rejected: stats databases / squad numbers (structured data — SQL is the right
tool, RAG is for prose); scraping (unnecessary); ChatGPT Go / Google AI Pro /
Gemini subscriptions (UI access, not a programmatic API — unusable in an app).

## Embeddings

Q: What converts text to vectors?
Chose: Local `sentence-transformers/all-MiniLM-L6-v2` (384-dim).
Why: Free, no API key, fast, adequate for a learning corpus.
Rejected: OpenAI embeddings — paid, extra API surface, no learning benefit.
Status: Conceptual — ratify in TRD.

## Vector DB

Q: Where do vectors get stored and searched?
Chose: Chroma, local.
Why: Zero setup, local persistence, good for learning the mechanics.
Rejected: Pinecone (managed cloud, overkill); FAISS (lower-level, more glue
code for no conceptual gain).
Status: Conceptual — ratify in TRD.

## Generation LLM

Q: Which LLM for answer generation?
Chose: Claude Haiku 4.5.
Why: API access already on hand, cheap, reliable.
Rejected: Gemini free tier — limits unstable, risk of breaking mid-build;
OpenAI API — separate billing setup, no benefit over Haiku here.
Status: Conceptual — ratify in TRD.

## Chunking strategy

Q: Fixed-size, section-based, or semantic?
Chose: Hybrid — Wikipedia section is the logical unit; split with overlap only
if it exceeds a token limit; drop chunks below a minimum size.
Why: Keeps meaning at boundaries (section) and even sizes within (split).
Rejected: pure fixed-size (splits sentences, ignores structure); pure section
(uneven sizes, huge History section buries the answer in noise); pure semantic
(must embed before chunking — expensive — plus a threshold to tune).
Status: Conceptual — token params fixed in TRD.

## Retrieval: two-stage

Q: Retrieve top-5 directly, or retrieve wide then re-rank?
Chose: Vector search top-20 (recall) → cross-encoder rerank to 5 (precision).
Why: Vector search is approximate — the best chunk can sit at rank 8 and a
direct top-5 would miss it. Wide net + exact rerank on ~20 items is affordable.
Rejected: single-stage top-5 — approximate search misses good chunks.
Status: Conceptual — ratify in TRD.

## Confidence gating

Q: Vector search always returns something — how do we avoid answering from
irrelevant chunks?
Chose: Gate on similarity score; if nothing clears the threshold, refuse
without calling the LLM.
Why: Ungated retrieval feeds "least bad" chunks to the LLM on out-of-corpus
questions → confident hallucination. This is the mechanism behind PRD FR-5.

## Prompt ordering

Q: What order do retrieved chunks go in the prompt?
Chose: Highest-scoring chunks at the edges (first/last), not the middle.
Why: "Lost in the middle" — LLMs attend most to the start and end of a long
context; burying the best chunk wastes correct retrieval.
Status: Conceptual.

## Corpus scope

Q: How much of Wikipedia per club?
Chose: Main article + ~10 notable players/club + last 10 seasons/club
(2015–16 → 2024–25). ≈ 105 documents.
Why: User chose "main + players + seasons". Unbounded ingestion makes
correctness impossible to verify, and evaluation is the known failure point.
Bounded so an eval set can be authored; cutoffs explicitly overridable.
Status: Pending PRD review (PRD OQ-1).

## Question types

Q: What kinds of questions must v1 answer?
Chose: Factual lookup + cross-club comparative. Out: subjective/ranking,
standalone temporal-narrative.
Why: Subjective/ranking has no ground truth and forces caveated answers — not
objectively measurable for v1.

## Audience

Q: Personal demo or hypothetical public product?
Chose: Hypothetical public product.
Why: Forces honest NFRs (latency, cost, rate limiting, abuse handling) and is
a stronger portfolio story. Cost: more to build.

## Data freshness

Q: Static snapshot or scheduled re-ingest?
Chose: One-time static snapshot.
Why: Wikipedia drift is irrelevant for a fixed learning corpus; avoids a
scheduler and refresh ops. Answers are "as of" the snapshot date.

## Parametric knowledge contamination

Q: Football is heavily in the LLM's pretraining — if Claude answers correctly,
we can't tell if retrieval worked or it already knew. Is football a bad corpus?
Chose: Keep football. Fix the *evaluation*, not the data:
1. Closed-book vs open-book differential scoring — ask each eval question with
   and without context; exclude ones the model answers closed-book.
2. Counterfactual planted-fact subset — edit a few docs with false-but-
   plausible facts; a correct planted answer proves retrieval is used.
3. Long-tail bias — favour specific recent-season facts over headline facts.
Why: Mechanics are corpus-independent; switching loses the hand-verifiable
domain. Trading a solvable measurement problem for an unfamiliar domain is bad.
Rejected: switching to an out-of-training corpus.
Status: Proposed — pending review 2026-05-20; sharpens PRD NFR-7; new PRD OQ-5.
