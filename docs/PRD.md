# PRD — RAG Football Wiki

**Status:** Draft v1
**Owner:** Pranav Sagar
**Last updated:** 2026-05-19
**Stage:** Business requirements. TRD and architecture follow this document.

---

## 1. Problem

General-purpose LLMs answer football questions from frozen, unattributed
training data. They cannot cite a source, cannot be corrected by feeding new
material, and confidently invent specifics (transfer fees, scorelines, dates).
For a fan or researcher asking about a specific club, an unsourced answer is
worth little — they cannot tell a fact from a hallucination.

## 2. Goal

A question-answering product that answers natural-language questions about a
defined set of football clubs **using only a curated Wikipedia corpus**, and
returns the **source** behind every answer. If the corpus does not contain the
answer, the product says so instead of guessing.

## 3. Users

| Persona | Need |
|---|---|
| **Football fan / researcher** (primary) | Ask club questions in plain English, get a correct, sourced answer fast. |
| **Technical reviewer** (secondary stakeholder) | Evaluate the product as a portfolio piece — judges answer quality, sourcing, and refusal behaviour. |

## 4. Scope — Clubs

Real Madrid CF, FC Barcelona, Manchester United F.C., Liverpool F.C.,
Arsenal F.C.

## 5. Scope — Corpus (v1, bounded)

Decision: **main + players + seasons**, bounded for verifiability.

| Source type | v1 cutoff | Rationale |
|---|---|---|
| Main club article | 1 per club (5) | Core narrative & facts. |
| Notable players | ~10 per club (~50) | Bounded so an evaluation set can be authored. |
| Seasons | last 10 completed per club, 2015–16 → 2024–25 (~50) | Recent, high-interest, bounded vs. ~120 historical seasons. |

**≈ 105 documents total.**

> **Why bounded:** unbounded player/season ingestion makes correctness
> impossible to verify, and evaluation is the known failure point for RAG
> products. Cutoffs are an explicit, overridable decision — they can widen in
> a later version once an evaluation harness exists. See Open Question OQ-1.

Data is a **one-time static snapshot** taken at build time. Answers are "as of"
the snapshot date. Tracking live Wikipedia edits is out of scope for v1.

## 6. Functional Requirements

- **FR-1** Accept a natural-language question via an API.
- **FR-2 — Factual lookup.** Answer single-fact questions about a club, player,
  or season (e.g. "When was Liverpool founded?", "Who managed Arsenal in
  2018–19?").
- **FR-3 — Cross-club comparative.** Answer questions spanning multiple clubs
  (e.g. "How did Real Madrid's and Barcelona's transfer approaches differ in
  the 2010s?"). Requires evidence drawn from more than one document.
- **FR-4 — Sourced answers.** Every answer cites the corpus document(s) it was
  derived from, traceable back to the originating Wikipedia article.
- **FR-5 — Grounded refusal.** When the corpus does not support an answer, the
  product explicitly states it cannot answer rather than guessing. This is a
  hard requirement, not best-effort.
- **FR-6 — Answer faithfulness.** Answers must be derived from retrieved corpus
  content, not from the underlying model's prior knowledge.

## 7. Non-Functional Requirements

Framed as a hypothetical public product. Concrete numeric budgets are set in
the TRD; this section states intent and the bar.

- **NFR-1 — Latency.** Interactive response time. A p95 end-to-end budget is
  defined in the TRD; the product must feel responsive, not batch.
- **NFR-2 — Cost.** A per-query cost ceiling is defined in the TRD. Design must
  favour low marginal cost per query.
- **NFR-3 — Abuse & input safety.** Reject malformed, oversized, or
  prompt-injection-style inputs. Question length is bounded.
- **NFR-4 — Rate limiting.** Per-client request limiting to protect cost and
  availability.
- **NFR-5 — Observability.** Every query logs: question, retrieved sources,
  refusal/answer outcome, latency. Required for evaluation and debugging.
- **NFR-6 — Privacy.** No user accounts, no PII collected. Logged questions
  treated as non-identifying.
- **NFR-7 — Evaluability.** A fixed evaluation set (curated Q&A over the
  bounded corpus) must exist before launch; release is gated on its results.

## 8. Success Metrics

| Metric | Target (v1) |
|---|---|
| Factual answer correctness (eval set) | ≥ 85% |
| Cross-club answer faithfulness (judged) | ≥ 90% grounded, no invented facts |
| Correct refusal on out-of-corpus questions | ≥ 90% |
| Answer carries a valid source | 100% of non-refused answers |
| p95 end-to-end latency | meets TRD budget |

## 9. Out of Scope (v1)

- Subjective / ranking questions ("which club had the best decade").
- Standalone temporal-narrative questions not framed as factual or comparative.
- Clubs outside the five listed.
- Languages other than English.
- Live / scheduled Wikipedia refresh.
- Multi-turn conversation or memory across questions.
- Authentication and user accounts.

## 10. Assumptions

- English Wikipedia is accepted as ground truth for this product.
- Snapshot is taken once at build time; staleness is acceptable for v1.
- Wikipedia article structure is stable enough to parse at snapshot time.

## 11. Open Questions

- **OQ-1** Exact corpus cutoffs (player count per club, season window) — current
  values are a recommendation; confirm or adjust before ingestion.
- **OQ-2** Source of the "notable players" list per club (main-article section
  vs. explicit list) — affects ingestion determinism.
- **OQ-3** Numeric NFR budgets (latency p95, per-query cost ceiling,
  rate-limit thresholds) — to be fixed in the TRD.
- **OQ-4** Authorship and size of the evaluation set (NFR-7).

## 12. Out of This Document

Technology choices, component design, data/metadata schema, retrieval
strategy, and chunking are **deliberately excluded** — they belong in the TRD
and architecture documents that follow.
