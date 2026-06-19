# Bonus Design: Customer-Support Chatbot Flywheel for a Vietnamese E-commerce Platform

## The Problem

A mid-size Vietnamese e-commerce company (think Tiki or Lazada VN scale: ~2M
orders/month) runs a customer-support chatbot powered by a fine-tuned LLM. The
chatbot handles return requests, order status, and product questions in
Vietnamese. The current model was fine-tuned once six months ago on human-curated
Q&A pairs and has not been updated since.

**The constraint set that makes this hard:**

- Vietnamese text with regional accent variation (northern vs. southern tone marks,
  mixed loanwords, Gen-Z slang) — a tokenizer trained on English-heavy corpora
  misses ~15% of tokens.
- PDPL compliance (Law 91/2025): user chat logs are PII and cannot be shipped to
  a third-party LLM API for fine-tuning. The whole flywheel must run on
  on-premise or private-cloud infrastructure.
- The product catalog changes daily (new SKUs, new promotions). Static training
  data goes stale within weeks: users ask about products the model has never seen.
- The operations team has no ML engineers — only a data engineering team of 3.

The goal: **build a self-improving data pipeline** that turns the chatbot's daily
production traffic into a continuously-refreshed eval set and DPO preference
corpus, with PDPL-safe handling, at a total infra cost under $500/month.

---

## Architecture Sketch

```
 Production chatbot (FastAPI)
    │
    │  OpenTelemetry gen_ai.* spans (1 per turn)
    ▼
 Kafka topic: raw-traces  ──────────────────────────────┐
    │                                                    │
    ▼                                                    ▼
 Bronze (DuckDB / S3)          PII scrubber             │
  append-only, verbatim   ◄──  (regex + NER, on-prem)   │
    │                                                    │
    ▼                                                    │
 Quality gate (Pandera)                                  │
  quarantine: null output,                               │
  latency >5 s, refusals                                 │
    │                                                    │
    ▼                                                    │
 Silver: clean spans                                     │
    │                                                    │
    ├──► eval set (split='eval', human-confirmed)        │
    │         written weekly by a human reviewer         │
    └──► DPO pairs (ok vs error turns, same prompt)      │
              decontaminated against eval set            │
    │                                                    │
    ▼                                                    │
 datasets/ (eval_golden.jsonl, preference_pairs.jsonl)   │
    │                                                    │
    ▼                                                    │
 Fine-tuning job (on-prem GPU, weekly)  ◄────────────────┘
    │
    ▼
 New chatbot version (shadow-deployed, A/B tested)
```

---

## Open Questions and Decisions

### 1. Batch or Streaming? (Question 2)

**Decision: batch with micro-batch windowing, not true streaming.**

Argument for streaming: Kafka is already running for the trace bus; why not
consume continuously? Real-time feature updates sound appealing.

**Tradeoff:** True streaming requires a stateful consumer (e.g. Flink) that
tracks per-user conversation context across a sliding window to build DPO pairs.
At the scale of ~2M turns/month (~800 turns/hour), latency for fine-tuning data
is irrelevant — the model is retrained weekly regardless. A nightly DuckDB batch
job over the previous day's Bronze partition is 10× simpler, half the cost, and
easier for a 3-person team to operate. The Kafka topic is retained for 7 days;
batch consumption from an offset is reliable enough.

**Rejected alternative:** Lambda architecture (stream + batch). The complexity of
maintaining two compute paths with identical semantics is not justified here —
the only consumer of the fine-tuning data runs weekly.

---

### 2. PII and PDPL Compliance (Question 10 / Vietnamese context)

**Decision: scrub PII in the Bronze ingest stage, before anything is written to disk.**

Vietnamese chat logs contain user names, phone numbers, addresses, and order IDs
(which are PII under PDPL Article 2 if they can identify a person). PDPL
prohibits processing personal data outside approved jurisdictions without consent.

**Tradeoff:** Scrubbing before Bronze keeps the immutable raw layer PII-free,
which is clean but means Bronze is NOT truly verbatim — we lose some signal for
debugging (e.g. "did the user mention their address?"). The alternative is to
write verbatim to Bronze in a high-security on-prem partition with access logging
and scrub at the Silver stage. This is more powerful for debugging but requires
managing two security zones.

We chose pre-Bronze scrubbing because the operations team cannot manage two zones
reliably, and the lost debugging signal is acceptable: we can reconstruct "what
was asked" from the anonymized version.

**Rejected alternative:** Pseudonymization instead of deletion. Replacing user
names with UUIDs seems safer but introduces a mapping table that is itself PII
under PDPL — you have not reduced your compliance surface, just moved it.

---

### 3. Quality Gate: What to Quarantine (Question 4)

**Decision: quarantine on null output, latency > 5s, refusals, and hallucination-flagged turns; pass everything else.**

The quality gate cannot use an LLM judge (cost, PDPL, latency). We use rule-based
heuristics: turns where the chatbot said "I cannot help" or where a downstream
human reviewer marked the response as factually wrong are routed to the dead-letter
queue and excluded from DPO training.

**Tradeoff:** Rule-based quarantine misses subtle hallucinations. An embedding
similarity check (comparing the output against the known-correct RAG chunks)
catches more errors — but it requires an on-prem embedding model and adds 200ms
latency per turn. At 800 turns/hour, that is 160ms·800 = 128 CPU-seconds/hour:
feasible, but the operations team asked for simplicity at first launch. We will
add the embedding check in phase 2.

---

### 4. Eval Set Curation: Human vs Automated (Question 7)

**Decision: human-in-the-loop for the eval set; automated for DPO pairs.**

The eval set is the ground truth that determines whether the model improved. If it
is wrong, every training iteration lies. We require a human reviewer to mark
`split='eval'` on ~50 turns per week. This is the only human-in-the-loop step
in the pipeline.

**Tradeoff:** Automating eval selection (e.g. taking all turns with high BLEU
against the knowledge base) is faster but produces an eval set that is correlated
with the knowledge base — evaluating RAG quality, not model quality. A human
reviewer catches distribution shift (new question types), adversarial inputs, and
category errors.

DPO pairs are automated: pair any `ok`-turn with an `error`-turn that shares the
same normalized prompt. This is noisier but we have decontamination to prevent
the worst leakage.

---

### 5. Train/Serve Parity for the Catalog Feature (Question 5)

**Decision: ASOF join for all catalog-derived features.**

The chatbot uses product-level features (average return rate, price tier, promo
flag) as context. These features change daily. If we join the current feature
value to historical training events (naive join), every training row for a
product that ran a promotion last month gets the current no-promo price — a
future leak.

**Tradeoff:** Maintaining a feature history table (product × valid_from ×
feature_value) costs extra storage (~5 GB/year at this scale). The ASOF join is
also ~3× slower than a naive latest-value join in DuckDB on a 1M-row table.
Both costs are acceptable versus the training-serving skew a naive join would
introduce. A model trained on leaked features will show high offline accuracy
and low online win rate — the classic expensive failure mode.

---

### 6. Scaling Bottleneck at 10× (Question 3)

At 10× the current volume (~20M turns/month), the first bottleneck is **DuckDB
running on a single node**: a full nightly scan of 30-day Bronze (~60M spans)
takes ~4 minutes today and would take ~40 minutes, hitting the 1-hour batch
window. The fix is to partition Bronze by date and limit nightly scans to
the last 7 days (older turns rarely produce new eval signal).

The second bottleneck is the **human eval reviewer**: 50 turns/week does not
scale to 500 turns/week without tooling. The phase-2 plan is a lightweight review
UI that surfaces candidate eval turns ranked by embedding distance from the
current eval set (most novel first), so a reviewer can approve 50 turns in
10 minutes rather than 60.

---

## Summary of Rejected Alternatives

| Alternative | Reason rejected |
|---|---|
| Lambda architecture (stream + batch) | Complexity not justified at weekly fine-tuning cadence |
| Pseudonymization instead of PII deletion | The mapping table is itself PII under PDPL |
| LLM judge for quality gate | PDPL restriction on sending data to external APIs; cost |
| Automated eval set selection (BLEU-based) | Correlated with knowledge base, not model quality |
| Latest-value feature join (naive) | Leaks future catalog features into training rows |

---

## Conclusion

The core insight is that a flywheel for a PDPL-constrained Vietnamese chatbot must
make different choices than the English-language default: PII scrubs before Bronze
(not after), human reviewers for eval (not LLM judges), batch (not streaming), and
ASOF feature joins to prevent catalog-drift leakage. None of these decisions are
obvious from a generic tutorial; each comes from the specific constraint set.
The pipeline described here can be operated by a 3-person team on ~$400/month of
on-prem GPU and cloud storage, which is the actual binding constraint at this
company's scale.
