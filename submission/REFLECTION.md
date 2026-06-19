# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

   The **split labeling** in traces (which marks rows as `'eval'` vs train) is the
   most silent failure point. If the upstream telemetry SDK stops emitting `split`
   attributes, `build_eval_set` returns zero rows and the eval set quietly empties
   — but the pipeline still runs. Detection: alert on `len(eval_set) == 0` after
   every flywheel run, and track eval-set size as a data-quality metric over time.

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

   The model memorizes the exact eval prompts during DPO training, so it scores
   perfectly on those examples — but the lift is measurement error, not capability.
   The lie shows up as a large gap between eval metrics and held-out production
   metrics (or A/B win rate): offline looks great, online is flat.

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

   **Cumulative purchase count** for a recommendation model. If a user's total
   purchase count at training time is joined as-of today rather than as-of the
   event being labeled, early sessions (where the user had 0 prior purchases)
   get credited with 50 purchases. The model learns "users who bought a lot buy
   more" — a tautology — and the feature is useless at serve time for new users.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

   **Graph wins:** "Where does a widget ship from?" — the answer requires joining
   two separate sentences (widget IS_A accessory; accessory SHIPS_FROM Hanoi).
   No single chunk contains both facts, so top-1 vector retrieval can only return
   half the chain regardless of embedding quality.

   **Graph is overkill:** "What is the widget return policy?" — a single sentence
   ("Customers may return widgets within 30 days") answers this completely. Vector
   retrieval finds the right chunk in one hop; building and traversing a graph adds
   complexity with no payoff.
