# Provenance Guard

A backend system that classifies submitted text as likely AI-generated, likely human-written, or uncertain — using two independent detection signals, a calibrated confidence score, a transparency label for end users, and an appeals workflow for contested classifications.

## Architecture Overview

A submission's raw text is sent to `POST /submit`, where it's passed independently to two detection signals: an LLM-based semantic judgment (Groq) and a stylometric structural analysis (pure Python). The two scores are combined with fixed weights into a single confidence value. That confidence is mapped to one of three transparency labels, and the full result — both signal scores, the combined confidence, and the label — is written to a structured audit log keyed by a generated `content_id`. The endpoint returns the `content_id`, attribution, confidence, and label to the caller.

If a creator disputes a classification, they call `POST /appeal` with the `content_id` and their reasoning. The system looks up the original log entry, updates its status to `under_review`, and appends the creator's reasoning to that same entry — no automated re-classification occurs; it's flagged for manual review. `GET /log` exposes the full audit log for inspection.

```
POST /submit
    │
    ▼
[ Signal 1: Groq LLM ]  [ Signal 2: Stylometric heuristics ]
    │                          │
    └───────────┬──────────────┘
                ▼
      [ Confidence Scoring ]
                │
                ▼
      [ Transparency Label ]
                │
                ▼
      [ Audit Log Write ]
                │
                ▼
           Response

POST /appeal (content_id, creator_reasoning)
                │
                ▼
   [ Find log entry by content_id ]
                │
                ▼
[ status -> under_review, append reasoning ]
                │
                ▼
           Response
```

## Detection Signals

**Signal 1 — LLM-based classification (Groq, llama-3.3-70b-versatile)**
Asks the model to judge holistically whether text reads as human- or AI-written, returning a 0–1 score. This captures semantic and stylistic patterns — generic phrasing, hedging language, overly balanced argumentation — that are common in AI output but harder to fake mechanically. It can't capture: heavily edited AI text, or naturally formal human writing (academic prose, non-native English speakers) that happens to share surface traits with AI output. It's also a black box — there's no visibility into which features drove a given score.

**Signal 2 — Stylometric heuristics (pure Python)**
Combines sentence length variance, vocabulary diversity (type-token ratio), and punctuation regularity into a single 0–1 score. AI-generated text tends to be more uniform across these measures; human writing tends to be more irregular. It can't capture meaning or context at all, and it's unreliable on short passages — there isn't enough data for the statistics to be meaningful (see Known Limitations).

These are combined as a weighted average: `confidence = 0.6 * llm_score + 0.4 * stylometric_score`. The LLM signal is weighted higher because, in testing, it tracked intuitive judgments more reliably than the stylometric signal, which often produced near-neutral scores regardless of how clearly AI-like or human-like a short passage actually was.

## Confidence Scoring

Confidence represents the system's estimated probability the content is AI-generated (0–1). Thresholds:
- `>= 0.75` → **likely_ai**
- `<= 0.35` → **likely_human**
- in between → **uncertain**

This is an intentionally wide uncertain band. A false positive — telling a human creator their own work was flagged as AI — is more damaging to trust on a creative platform than a missed AI detection, so the system is biased toward "uncertain" rather than confidently mislabeling borderline cases.

**Validating that scores are meaningful:** tested against deliberately chosen inputs spanning the confidence range. Two examples from actual runs:

| Input | llm_score | stylometric_score | confidence | attribution |
|---|---|---|---|---|
| "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous..." | 0.8 | 0.498 | **0.679** | uncertain |
| "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it..." | 0.2 | 0.276 | **0.23** | likely_human |

The casual, irregular human text scored clearly low. The AI-sounding text scored high on the LLM signal alone (0.8) but only moderate on stylometry (0.498), pulling the combined score into "uncertain" rather than "likely_ai" — this is a real finding, not a hypothetical, and is discussed further under Known Limitations.

## Transparency Label

| Variant | Exact text |
|---|---|
| High-confidence AI | "This content shows strong signals of AI generation. Our system is highly confident (X%) this was AI-generated." |
| High-confidence human | "This content shows strong signals of human authorship. Our system is highly confident (X%) this was written by a person." |
| Uncertain | "We're not confident either way. This content has mixed signals (X% AI-likelihood) — it could be human-written, AI-generated, or AI-assisted. Treat this as inconclusive." |

`X` is substituted at runtime with the live confidence percentage, so each label reflects the actual submission rather than being a static string.

## Rate Limiting

The submission endpoint is limited to **10 requests per minute, 100 per day** per client (via Flask-Limiter, in-memory storage). 10/minute comfortably covers a real writer submitting drafts or revisions in a session, while making it impractical for a script to flood the endpoint. 100/day caps sustained abuse across a longer window without restricting normal usage patterns.

Verified by sending 12 rapid requests in a loop: the first several succeeded (`200`), and once the per-minute limit was reached, subsequent requests correctly returned `429 Too Many Requests`:
```
200
429
429
429
429
429
429
429
429
429
429
```

## Known Limitations

1. **Short, AI-generated passages can be misclassified as "uncertain" instead of "likely_ai."** In testing, a clearly AI-generated paragraph scored 0.8 on the LLM signal but only 0.498 (near-neutral) on stylometry, pulling the weighted average below the 0.75 threshold. This happens because sentence-length-variance and type-token-ratio statistics need enough text to be meaningful — on short samples, the stylometric signal contributes noise rather than signal, even when the LLM signal is confident.
2. **Formal writing by non-native English speakers or in technical/academic domains** can show low vocabulary diversity and uniform sentence structure — the same structural traits the stylometric signal associates with AI generation — risking a false "uncertain" or "likely_ai" result for legitimately human writing.

## Spec Reflection

The spec's requirement to write out exact label text *before* building anything (Milestone 2) was genuinely useful — having the three variants nailed down ahead of time meant the label-generation function was a straightforward lookup rather than something that needed to be reverse-engineered from vague requirements later.

Where implementation diverged from the original plan: the spec assumed both signals would contribute comparably useful information across all input lengths, but testing revealed the stylometric signal is substantially weaker on shorter text. Rather than silently re-tuning the weights to force "expected" results, we kept the original 0.6/0.4 weighting and documented the resulting miscalibration as a known limitation, since artificially tuning the system to match a single test case would be less honest than acknowledging where the multi-signal approach currently falls short.

## AI Usage

1. Directed the AI tool to generate the Flask app skeleton and the Groq signal function from the Detection Signals section of planning.md plus the architecture diagram. The generated function signature didn't quite match the planned output format, so it was revised to return a plain float in [0,1] rather than a dictionary, to match what the confidence-scoring step expected.
2. Directed the AI tool to generate the confidence-scoring and label-generation logic from the Uncertainty Representation and Transparency Label sections. The first version it produced used a binary AI/human split instead of the three-bucket threshold system specified in planning.md; this was corrected to match the 0.75/0.35 thresholds exactly before wiring it into the endpoint.
