# Provenance Guard - Planning
## Architecture
Submission flow:
POST /submit → Signal 1 (Groq LLM) → Signal 2 (Stylometric) → Confidence Scoring → Transparency Label → Audit Log → Response
Appeal flow:
POST /appeal → Update status to "under_review" → Log appeal alongside original decision → Response
[Client] --text,creator_id--> [POST /submit]
|
+--------------+--------------+
|                             |
[Signal 1: Groq LLM]          [Signal 2: Stylometry]
(semantic/stylistic           (sentence length variance,
coherence, 0-1 score)         vocab diversity, punctuation)
|                             |
+--------------+--------------+
|
[Confidence Scoring]
(weighted combination -> 0-1 score)
|
[Transparency Label]
(maps score to one of 3 label variants)
|
[Audit Log Write]
(content_id, signals, confidence, label)
|
[Response]
[Client] --content_id,reasoning--> [POST /appeal]
|
[Lookup original entry by content_id]
|
[Update status -> under_review]
|
[Audit Log Write: append appeal_reasoning]
|
[Response: confirmation]

Narrative: A submission's raw text is passed independently to two detection signals — an LLM-based semantic judgment and a stylometric structural analysis — whose scores are combined with fixed weights into a single confidence value representing the probability the content is AI-generated. That confidence is mapped to one of three transparency labels and everything (both signal scores, the combined score, and the label) is written to a structured audit log keyed by a generated content_id. An appeal references that same content_id, updates its status to "under_review," and appends the creator's reasoning to the same log entry rather than creating a separate, disconnected record.

## Detection Signals

1. **Signal 1 — LLM-based classification (Groq, llama-3.3-70b-versatile)**
   - What it measures: Asks the model to holistically judge whether the text reads as human- or AI-written, returning a probability (0–1) that the content is AI-generated.
   - Why this property differs: LLMs are good at picking up on generic phrasing patterns, hedging language ("it is important to note"), and overly balanced/structured argumentation that's common in AI output but less common in unscripted human writing.
   - What it can't capture: It can be fooled by heavily edited AI text or by human writers whose natural style happens to be formal/structured (e.g., academic writing, non-native English speakers). It's also a black box — we don't know exactly which features it's keying on.

2. **Signal 2 — Stylometric heuristics (pure Python)**
   - What it measures: Three structural sub-metrics combined into one score — sentence length variance (uniformity), type-token ratio (vocabulary diversity), and punctuation density regularity.
   - Why this property differs: AI-generated text tends to have more uniform sentence lengths and more "average" vocabulary diversity; human writing tends to be more irregular, with bursts of short and long sentences and more idiosyncratic punctuation habits.
   - What it can't capture: It says nothing about meaning or content — a human could write in an unusually uniform style (technical writing, copyediting) and get flagged. It's also unreliable on very short texts where there isn't enough data to compute meaningful statistics.

These two signals are genuinely independent: one is semantic (what the text is "saying" and how it's framed), the other is purely structural (how the text is shaped, regardless of meaning). That independence is why combining them is more informative than either alone.

**Output format:** Both signals output a float in [0, 1], where higher = more AI-like. Combined via weighted average:
`confidence = 0.6 * llm_score + 0.4 * stylometric_score`
(LLM weighted higher because it has access to semantic context the stylometric signal lacks.)

## Uncertainty Representation

A confidence score is the system's estimated probability the content is AI-generated, not a verdict.

- `confidence >= 0.75` → **likely_ai**
- `confidence <= 0.35` → **likely_human**
- everything in between → **uncertain**

These thresholds are deliberately asymmetric and wide in the middle. On a creative-writing platform, a false positive (calling a human's work AI-generated) is more damaging to trust than a false negative, so the system is biased toward returning "uncertain" rather than confidently mislabeling borderline cases. A score of 0.51 and a score of 0.95 must never produce the same label — that's enforced directly by these three buckets and the % shown in the label text.

## Transparency Label Design

- **High-confidence AI:** "This content shows strong signals of AI generation. Our system is highly confident (X%) this was AI-generated."
- **High-confidence human:** "This content shows strong signals of human authorship. Our system is highly confident (X%) this was written by a person."
- **Uncertain:** "We're not confident either way. This content has mixed signals (X% AI-likelihood) — it could be human-written, AI-generated, or AI-assisted. Treat this as inconclusive."

X is the live confidence percentage substituted at runtime, so each label is specific to the actual submission, not a static string.

## Appeals Workflow

- **Who can appeal:** Any creator whose content received a content_id from `/submit`.
- **What they provide:** `content_id` and `creator_reasoning` (free text explaining why they believe the classification is wrong).
- **What happens on appeal:** The system looks up the original log entry by content_id, sets its status to `"under_review"`, and appends the `creator_reasoning` field to that same entry (not a new disconnected record) so the original signals/confidence/label remain visible alongside the appeal.
- **What a human reviewer would see:** Querying `/log` (or filtering the log by `status == "under_review"`) surfaces an entry containing the original text excerpt, both signal scores, the combined confidence, the label shown to the user, and the creator's appeal reasoning — everything needed to make a manual judgment call without re-running detection.

## Anticipated Edge Cases

1. **Very short submissions (under ~50 words):** The stylometric signal's sentence-length-variance and type-token-ratio calculations are statistically unreliable on small samples — a short, perfectly normal human sentence can look "uniform" simply because there's only one or two sentences to measure.
2. **Formal writing by non-native English speakers:** Simpler, more repetitive sentence structures and lower vocabulary diversity (common when writing in a non-native language) can resemble the structural patterns the stylometric signal associates with AI generation, risking a false "likely_ai" or "uncertain" result for entirely human work.

## AI Tool Plan

- **M3 (submission endpoint + first signal):** Provide the Detection Signals section + architecture diagram. Ask for: Flask app skeleton with `POST /submit` stub, and the Groq signal function. Verify by calling the signal function directly with 2–3 test strings before wiring into the endpoint.
- **M4 (second signal + confidence scoring):** Provide Detection Signals + Uncertainty Representation sections + diagram. Ask for: the stylometric signal function and the scoring/combination logic. Verify the generated thresholds match 0.75/0.35 exactly, and test against the 4 sample inputs from the project spec to confirm scores vary meaningfully and roughly match intuition.
- **M5 (production layer):** Provide Transparency Label + Appeals Workflow sections + diagram. Ask for: the label-generation function and the `/appeal` endpoint. Verify all three label variants are reachable by forcing different confidence values, and confirm an appeal call actually updates status to `under_review` and is visible via `/log`.






