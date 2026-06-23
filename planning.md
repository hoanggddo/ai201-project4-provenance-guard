# Provenance Guard — planning.md

## Architecture Narrative

A creator submits a piece of text to `POST /submit` along with their `creator_id`. The text
is passed through two independent detection signals: an LLM-based classifier (Groq,
llama-3.3-70b-versatile) that assesses semantic and stylistic coherence holistically, and a
stylometric heuristics function that computes measurable statistical properties of the text in
pure Python. Each signal returns a score between 0 and 1 (higher = more likely AI-generated).
The two scores are combined into a single confidence score via a weighted average. That
confidence score is mapped to one of three transparency label variants, which is returned to
the caller. Every submission writes a structured JSON entry to the audit log. The response
includes a unique `content_id`, the attribution result, the confidence score, and the label text.

If a creator believes their content was misclassified, they submit a `POST /appeal` with the
`content_id` and their reasoning. The system looks up the original log entry, updates its
status to `"under_review"`, appends the appeal reasoning, and returns a confirmation. No
automated re-classification occurs — the appeal is queued for human review.

---

## Architecture Diagram

```
SUBMISSION FLOW
===============

POST /submit
  { text, creator_id }
        |
        v
+-------------------+       +------------------------+
| Signal 1: LLM     |       | Signal 2: Stylometrics |
| (Groq)            |       | (Pure Python)          |
| → llm_score 0–1   |       | → stylo_score 0–1      |
+-------------------+       +------------------------+
        |                           |
        +----------+----------------+
                   |
                   v
        +---------------------+
        | Confidence Scoring  |
        | 0.65×llm +          |
        | 0.35×stylo          |
        | → confidence 0–1    |
        +---------------------+
                   |
                   v
        +---------------------+
        | Label Generation    |
        | ≤0.35 → human       |
        | 0.36–0.69 → uncert. |
        | ≥0.70 → AI          |
        +---------------------+
                   |
                   v
        +---------------------+
        | Audit Log (JSON)    |
        | write structured    |
        | entry               |
        +---------------------+
                   |
                   v
        Response:
        { content_id, attribution,
          confidence, label }


APPEAL FLOW
===========

POST /appeal
  { content_id, creator_reasoning }
        |
        v
  Look up entry by content_id
  → 404 if not found
  → 400 if already under_review
        |
        v
  Update status → "under_review"
  Append appeal_reasoning to log entry
        |
        v
  Return confirmation
```

---

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence holistically — whether the writing
"reads" like AI-generated prose in terms of structure, tone, hedging patterns, and narrative
flow.

**Why it differs between human and AI writing:** AI writing tends toward smooth transitions,
balanced sentence structure, hedged claims ("it is important to note that..."), and avoidance
of strong personal voice. Human writing is messier, more idiosyncratic, and more emotionally
variable.

**Output:** A JSON object `{ "ai_score": float, "reasoning": string }` where `ai_score` is
between 0 and 1. The model is prompted to return only this JSON with no preamble.

**Blind spot:** A skilled human writer who writes cleanly and formally can fool it. A heavily
edited AI draft can too. It is also a black box — we cannot inspect why it scored the way it
did, only that it did.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Statistical surface properties of the text that differ between human and
AI writing:

- **Sentence length variance:** Standard deviation of sentence lengths in words. AI output
  tends toward uniform sentence length; human writing varies more.
- **Type-token ratio (TTR):** Unique words / total words. AI tends toward moderate, consistent
  TTR. Human writing varies more — sometimes very repetitive, sometimes highly varied.
- **Punctuation density:** Punctuation characters / total characters. Human writing often
  contains more irregular punctuation (dashes, ellipses, informal marks); AI output is cleaner.

Each metric is normalized to a 0–1 "AI-likeness" score and averaged into a single
`stylo_score`.

**Output:** A float between 0 and 1.

**Blind spot:** Formal human writing (academic papers, legal prose, professional blog posts)
scores as AI-like because it shares surface properties with AI output — uniform sentence
length, controlled vocabulary, sparse punctuation. Short texts (under ~80 words) produce
unreliable metrics due to insufficient sample size.

---

### Combining Signals

```
confidence = (llm_score × 0.65) + (stylo_score × 0.35)
```

The LLM signal receives more weight because it captures meaning and stylistic intent, not just
surface statistics. Stylometrics serve as an independent structural check that cannot be gamed
by prompting.

---

## Uncertainty Representation

| Confidence range | Attribution label | Meaning |
|-----------------|-------------------|---------|
| 0.00 – 0.35 | `likely_human` | Strong signals of human authorship |
| 0.36 – 0.69 | `uncertain` | Insufficient signal to classify confidently |
| 0.70 – 1.00 | `likely_ai` | Strong signals of AI generation |

The uncertain band is intentionally wide. Most real-world content lives in the middle, and
being wrong in either direction in that range is costly. A score of 0.51 and a score of 0.95
produce meaningfully different labels and different label language.

**False positive asymmetry:** Labeling a human's work as AI-generated is worse than missing
an AI submission. The `likely_human` threshold (≤0.35) is therefore conservative — the system
must see strong evidence before confidently asserting human authorship, and the uncertain band
defaults to giving creators the benefit of the doubt.

---

## Transparency Label Variants

### 🟢 Likely human-written (confidence 0.00–0.35)

> **Appears to be human-written**
> Our system found no strong signals of AI-generated content. This is an automated assessment
> and not a guarantee of authenticity.

---

### 🟡 Uncertain (confidence 0.36–0.69)

> **Attribution unclear**
> Our system was unable to confidently determine whether this content was human-written or
> AI-generated. This does not mean the content is inauthentic.
> *If you're the creator and believe this is incorrect, you can submit an appeal.*

---

### 🔴 Likely AI-generated (confidence 0.70–1.00)

> **May contain AI-generated content**
> Our system found signals consistent with AI-generated writing. This is an automated
> assessment and may not be accurate.
> *If you're the creator and believe this is incorrect, you can submit an appeal.*

---

## Appeals Workflow

**Who can appeal:** Any caller who provides a valid `content_id`. Since the system has no
authentication layer, possession of the `content_id` is treated as proxy for creator identity.
This is a known limitation (see Edge Cases).

**What they provide:**
- `content_id` — the unique ID returned by `/submit`
- `creator_reasoning` — free text explaining why the classification is believed to be incorrect

**What the system does:**
1. Looks up the audit log entry by `content_id`
2. Returns `404` if the `content_id` does not exist
3. Returns `400` if the entry is already `"under_review"` (no duplicate appeals)
4. Updates the entry's `status` from `"classified"` to `"under_review"`
5. Appends `appeal_reasoning` and `appeal_timestamp` to the entry
6. Returns a confirmation response

**What a human reviewer sees:** The full audit log entry including original attribution,
confidence score, both individual signal scores, submission timestamp, and the creator's
reasoning — everything needed to make a judgment call.

---

## Anticipated Edge Cases

### Edge Case 1: Formal human writing scored as AI-generated

A non-native English speaker, academic, or legal professional writes in a precise, structured
style. Their sentence length variance is low, vocabulary is controlled, and punctuation is
sparse. The stylometric signal scores this as AI-like. The LLM signal may also flag it because
"formal and polished" correlates heavily with AI training data. Result: a false positive with
moderate-to-high confidence, which is exactly the worst outcome the system should minimize.

*Mitigation:* The wide uncertain band (0.36–0.69) catches many of these cases before they
reach a `likely_ai` label. The appeals workflow provides a path for affected creators.

### Edge Case 2: Short text produces unreliable stylometrics

A submission under ~80 words (a short poem, a caption, a brief note) does not give the
stylometric signal enough data to compute meaningful variance or TTR. The confidence score may
appear decisive (e.g., 0.71) but is based on very thin statistical evidence, with the LLM
signal carrying nearly all the weight — undermining the value of running two signals at all.

*Mitigation:* The system should note in the audit log when text is below a minimum word
threshold, and the confidence score should be treated as less reliable in those cases. A future
improvement would be to widen the uncertain band dynamically for short texts.

---

## API Surface

| Method | Endpoint | Accepts | Returns |
|--------|----------|---------|---------|
| POST | `/submit` | `{ text, creator_id }` | `{ content_id, attribution, confidence, label }` |
| POST | `/appeal` | `{ content_id, creator_reasoning }` | `{ status, message }` |
| GET | `/log` | — | `{ entries: [...] }` |

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1

**Spec sections to provide:** Detection Signals (Signal 1 only) + Architecture Diagram

**What to ask for:**
1. Flask app skeleton with `POST /submit` route stub that accepts `{ text, creator_id }` and
   returns a hardcoded response
2. A standalone `llm_signal(text)` function that calls Groq with a structured prompt and
   returns `{ "ai_score": float, "reasoning": string }`

**How to verify:** Call `llm_signal()` directly on 3–4 test inputs before wiring into the
route. Confirm it returns valid JSON with `ai_score` between 0 and 1. Test the route with
`curl` and confirm `content_id` appears in the response.

---

### Milestone 4 — Signal 2 + Confidence Scoring

**Spec sections to provide:** Detection Signals (Signal 2) + Uncertainty Representation +
Architecture Diagram

**What to ask for:**
1. A standalone `stylometric_signal(text)` function computing sentence length variance, TTR,
   and punctuation density, each normalized to 0–1 and averaged into a single score
2. A `compute_confidence(llm_score, stylo_score)` function implementing the weighted average
   and returning `{ confidence, attribution }` based on the threshold table

**How to verify:** Run both signals on the four test inputs from the spec. Do the scores vary
meaningfully? Does clearly AI-generated text score noticeably higher than clearly human text?
Print both signal scores separately to diagnose disagreements.

---

### Milestone 5 — Production layer

**Spec sections to provide:** Transparency Label Variants + Appeals Workflow + Architecture
Diagram

**What to ask for:**
1. A `generate_label(confidence, attribution)` function that returns the correct label text
   for all three variants
2. The `POST /appeal` endpoint implementing the full workflow: lookup, 404/400 guards, status
   update, log append, confirmation response

**How to verify:** Submit inputs that produce each of the three confidence bands and confirm
the correct label text is returned. Test the appeal endpoint with a valid `content_id`, a
nonexistent one (expect 404), and a re-appeal (expect 400). Check `GET /log` after each to
confirm the log reflects the correct state.
