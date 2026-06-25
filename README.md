# Provenance Guard

A backend content attribution system for creative sharing platforms. Provenance Guard classifies submitted text as likely human-written, likely AI-generated, or uncertain — returning a confidence score and a plain-language transparency label that platforms can display to readers. Creators who believe they've been misclassified can file an appeal through the same API.

---

## Architecture

A submitted piece of text travels through the following pipeline:

```
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
        | Audit Log (.jsonl)  |
        +---------------------+
                   |
                   v
        Response: { content_id, attribution,
                    confidence, label, signals }
```

**Appeal flow:**

```
POST /appeal { content_id, creator_reasoning }
  → lookup by content_id (404 if not found)
  → reject if already under_review (400)
  → update status to "under_review"
  → append appeal_reasoning + appeal_timestamp to log entry
  → return confirmation
```

The audit log is a `.jsonl` file (one JSON object per line). Every submission and appeal is captured there. `GET /log` returns all entries newest-first.

---

## Setup

**Requirements:** Python 3.10+, a free [Groq](https://console.groq.com) account.

```bash
git clone https://github.com/YOUR_USERNAME/ai201-project4-provenance-guard
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate        # Mac/Linux
.venv\Scripts\activate           # Windows
pip install -r requirements.txt
```

Create a `.env` file in the project root (never commit this):

```
GROQ_API_KEY=your_key_here
```

Start the server:

```bash
python app.py
```

---

## API Reference

### `POST /submit`

Submit a piece of text for attribution analysis.

**Request body:**
```json
{
  "text": "The content to analyze.",
  "creator_id": "any-string-identifying-the-creator"
}
```

**Response:**
```json
{
  "content_id": "uuid",
  "attribution": "likely_ai | uncertain | likely_human",
  "confidence": 0.7488,
  "label": {
    "headline": "May contain AI-generated content",
    "body": "Our system found signals consistent with AI-generated writing...",
    "appeal_note": "If you're the creator and believe this is incorrect, you can submit an appeal."
  },
  "signals": {
    "llm_score": 0.8,
    "llm_reasoning": "...",
    "stylo_score": 0.6536,
    "stylo_metrics": { ... }
  },
  "timestamp": "2026-06-25T04:01:09.813588+00:00"
}
```

Rate limited: **10 requests per minute, 100 per day** per IP address.

---

### `POST /appeal`

Contest a classification. Updates the submission's status to `under_review` and logs the creator's reasoning.

**Request body:**
```json
{
  "content_id": "uuid-from-submit-response",
  "creator_reasoning": "I wrote this myself. My formal writing style may appear AI-like."
}
```

**Response:**
```json
{
  "status": "appeal_received",
  "content_id": "uuid",
  "message": "Your appeal has been received and the content is now under review...",
  "appeal_timestamp": "2026-06-25T04:02:08.041226+00:00"
}
```

Returns `404` if the `content_id` does not exist. Returns `400` if an appeal has already been filed for this submission.

---

### `GET /log`

Returns all audit log entries, newest first. Includes individual signal scores, attribution results, confidence scores, and any appeal data.

---

## Detection Signals

### Signal 1: LLM Classification (Groq — `llama-3.3-70b-versatile`)

**What it measures:** Semantic and stylistic coherence holistically — whether the writing reads like AI-generated prose in terms of structure, tone, hedging patterns, and narrative flow.

**Why it differs:** AI writing tends toward smooth transitions, balanced sentence structure, hedged claims ("it is important to note that..."), and avoidance of strong personal voice. Human writing is messier, more idiosyncratic, and more emotionally variable.

**Output:** `ai_score` between 0–1 (higher = more likely AI) plus a one-sentence reasoning string. The model is prompted to return only JSON with no preamble. Parsing failures fall back to 0.5 (uncertain) to avoid silent errors causing false positives.

**Blind spot:** A skilled human writer who writes cleanly and formally can fool it. A heavily edited AI draft can too. It is also a black box — we cannot inspect why it scored as it did.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Three statistical surface properties of the text:

- **Sentence length variance** — standard deviation of sentence lengths in words. AI output tends toward uniform sentence length; human writing varies more. Low variance → higher AI-likeness score.
- **Type-token ratio (TTR)** — unique words / total words. AI tends toward a moderate, predictable TTR (~0.5). Human writing deviates more in both directions. A TTR close to 0.5 scores as more AI-like.
- **Punctuation density** — punctuation characters / total characters. Human writing often has denser, more irregular punctuation. Lower density → higher AI-likeness score.

Each metric is normalized to 0–1 and averaged into a single `stylo_score`.

**Why it complements Signal 1:** The LLM signal is semantic; stylometrics are structural. They are genuinely independent — a text can fool one without fooling the other. That independence is what makes combining them more informative than either alone.

**Blind spot:** Formal human writing (academic papers, legal prose) shares surface properties with AI output — uniform sentence length, controlled vocabulary, sparse punctuation — and scores as AI-like on this signal. Short texts (under ~80 words) produce unreliable metrics due to insufficient sample size.

---

## Confidence Scoring

```
confidence = (llm_score × 0.65) + (stylo_score × 0.35)
```

The LLM receives more weight because it captures meaning and stylistic intent, not just surface statistics. Stylometrics serve as an independent structural check.

| Confidence | Attribution | Label shown |
|------------|-------------|-------------|
| 0.00 – 0.35 | `likely_human` | Appears to be human-written |
| 0.36 – 0.69 | `uncertain` | Attribution unclear |
| 0.70 – 1.00 | `likely_ai` | May contain AI-generated content |

**On uncertainty:** The uncertain band is intentionally wide. Most real content lives in the middle, and being wrong in either direction in that range is costly. A confidence of 0.37 and a confidence of 0.68 both produce the "uncertain" label, but a 0.95 produces a different label than a 0.71 — both are `likely_ai`, but the score itself is surfaced to allow downstream systems to treat them differently if needed.

**False positive asymmetry:** Labeling a human's work as AI-generated is worse than missing an AI submission. The `likely_human` threshold (≤0.35) is conservative — the system must see strong evidence before asserting human authorship. When signals conflict, the wide uncertain band absorbs the disagreement rather than forcing a binary call.

---

## Transparency Label Variants

These are the exact strings returned in the `label` field of every `/submit` response.

### 🟢 Likely human-written (`confidence` 0.00 – 0.35)

> **Appears to be human-written**
> Our system found no strong signals of AI-generated content. This is an automated assessment and not a guarantee of authenticity.

*(No appeal note — the classification favors the creator.)*

---

### 🟡 Uncertain (`confidence` 0.36 – 0.69)

> **Attribution unclear**
> Our system was unable to confidently determine whether this content was human-written or AI-generated. This does not mean the content is inauthentic.
> *If you're the creator and believe this is incorrect, you can submit an appeal.*

---

### 🔴 Likely AI-generated (`confidence` 0.70 – 1.00)

> **May contain AI-generated content**
> Our system found signals consistent with AI-generated writing. This is an automated assessment and may not be accurate.
> *If you're the creator and believe this is incorrect, you can submit an appeal.*

**Design rationale:** None of the labels assert certainty. The AI label says "may contain," not "is." Both the uncertain and AI labels surface the appeals path. The human label includes a caveat too — no overclaiming in either direction. The language was chosen to be readable by a non-technical audience without being alarming.

---

## Rate Limiting

`POST /submit` is limited to **10 requests per minute** and **100 requests per day** per IP address.

**Reasoning:** A legitimate writer submits their own work infrequently — even a prolific creator is unlikely to submit more than a handful of pieces per session. 10 per minute is invisible to real usage while immediately stopping any automated script trying to flood the system or probe the classifier. 100 per day is generous enough for any individual user while limiting the blast radius of a compromised account or a sustained scripted attack. Exceeding either limit returns HTTP `429 Too Many Requests`.

---

## Known Limitations

**Formal human writing is misclassified as uncertain or AI-likely.** The stylometric signal measures properties that are genuinely shared between AI output and formal human prose — uniform sentence length, controlled vocabulary, low punctuation density. An academic, a non-native English speaker writing carefully, or a legal professional will score as AI-like on Signal 2 even when Signal 1 correctly identifies them as human. The wide uncertain band catches many of these cases, but the system will produce false positives on formal writing more than on casual writing. The appeals workflow is the primary mitigation.

**Short texts produce unreliable stylometric scores.** A submission under ~80 words does not give the variance and TTR calculations enough data to be meaningful. A three-sentence poem could produce any stylometric score. In those cases, the combined confidence score is effectively driven almost entirely by Signal 1, undermining the value of having two independent signals. A future improvement would dynamically widen the uncertain band for short texts and surface a low-confidence warning in the response.

**No authentication on appeals.** The system treats possession of a `content_id` as sufficient identity for filing an appeal. In production, appeals should require the creator to authenticate. As built, anyone who obtains a `content_id` (e.g., from a public API response) could file an appeal on someone else's behalf.

---

## Spec Reflection

**Where the spec helped:** Writing out the three transparency label variants in `planning.md` before writing any code meant that the label generation function had a concrete contract to implement against — no design decisions left for implementation time. The exact strings in the code match the exact strings in the spec, which made verification trivial.

**Where implementation diverged:** The punctuation density metric did not behave as expected. Both clearly AI-generated and clearly human-written test inputs produced a `punctuation_density_score` of 1.0, meaning the metric saturated at its ceiling and contributed no meaningful signal. The normalization range (anchored at 0.03–0.10 raw density) did not match the actual punctuation density of the test inputs, which were both denser than the upper bound. As a result, the stylometric signal's ability to distinguish the two test cases relied almost entirely on sentence length variance and TTR. This is documented as a known limitation rather than patched, because adjusting the normalization range to fit two test inputs would be overfitting rather than genuine calibration.

---

## AI Tool Usage

**Instance 1 — generating the Flask app skeleton and signal functions.**
I provided the detection signals section of `planning.md` and the architecture diagram, and asked for: (1) the Flask app skeleton with a `POST /submit` stub, (2) a standalone `llm_signal()` function returning structured JSON, and (3) a `stylometric_signal()` function computing the three specified metrics. The output included all three components. I reviewed the LLM prompt carefully and revised it to add explicit instructions to return no preamble and to use triple-quoted text delimiters around the input, which the generated version had omitted. I also added regex stripping of markdown fences as a defensive measure after noticing the model occasionally wraps JSON in backticks.

**Instance 2 — generating the confidence scoring and label logic.**
I provided the uncertainty representation section (threshold table) and the label variants section, and asked for: (1) a `compute_confidence()` function implementing the weighted average and threshold mapping, and (2) a `generate_label()` function returning the correct label dict for each attribution result. The generated threshold logic used `< 0.35` rather than `<= 0.35` as specified, which would have made a score of exactly 0.35 fall into the uncertain band instead of the human band. I corrected this to match the spec exactly before using the output.
