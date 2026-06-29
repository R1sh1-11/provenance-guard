# Provenance Guard

A backend attribution system for creative platforms. Provenance Guard classifies submitted text as likely AI-generated or likely human-written, returns a plain-language transparency label, and gives creators a path to appeal a classification they believe is wrong.

---

## Architecture Overview

A submission enters through `POST /submit`. The endpoint passes the raw text to two independent detection signals running in sequence. Signal 1 sends the text to Groq (llama-3.3-70b-versatile) and extracts a probability that the content is AI-generated. Signal 2 runs stylometric heuristics in pure Python, computing sentence length variance, type-token ratio, and punctuation density, then averaging them into a second probability. The confidence scorer combines both signals using a weighted average (LLM 65%, stylometrics 35%) into a single calibrated score between 0.0 and 1.0. The label generator maps that score to one of three plain-language transparency labels. The full result, including both individual signal scores, is written to the audit log before the response is returned.

An appeal enters through `POST /appeal`. The endpoint looks up the original `content_id` in the audit log, updates its status to `under_review`, appends the creator's reasoning, and returns a confirmation. No automated re-classification happens. A human reviewer opens `GET /log` and sees the original decision alongside the appeal reasoning in one structured entry.

```
SUBMISSION FLOW

POST /submit (text, creator_id)
       |
       v
  LLM Signal (Groq)          output: float 0.0 to 1.0
       |
       v
  Stylometric Signal (Python) output: float 0.0 to 1.0
       |
       v
  Confidence Scorer           weighted average, 65/35
       |
       v
  Label Generator             maps score to plain-language string
       |
       v
  Audit Log                   writes full entry to audit_log.json
       |
       v
  JSON Response               content_id, attribution, confidence, both scores, label

APPEAL FLOW

POST /appeal (content_id, creator_reasoning)
       |
       v
  Lookup content_id in audit log
       |
       v
  Update status to "under_review", append reasoning
       |
       v
  JSON Response               confirmation + updated status
```

---

## Detection Signals

### Signal 1: LLM Classification via Groq

**What it measures:** Whether the text reads as AI-generated based on semantic coherence, phrasing predictability, hedging language patterns, overly balanced structure, and whether the writing feels personally grounded or templated. The model evaluates holistically, the way a careful human reader would.

**Output:** A float between 0.0 and 1.0 extracted from a structured JSON response. The model is prompted at temperature 0.1 to minimize variability.

**What it misses:** Heavily edited AI output that a human has substantially reworked. AI models prompted to write casually or imperfectly can also fool this signal. The score is not fully deterministic, so identical inputs can produce slightly different results across calls.

### Signal 2: Stylometric Heuristics

**What it measures:** Three statistical properties that differ reliably between AI and human writing:

- **Sentence length variance:** AI text produces sentences of similar length. Human writing is irregular. Low standard deviation scores higher on the AI scale.
- **Type-token ratio (TTR):** Vocabulary diversity measured as unique words divided by total words. AI text reuses common words more than humans do in short passages. Low TTR scores higher on the AI scale.
- **Punctuation density:** Punctuation marks per 100 words. AI text uses punctuation in predictable, grammatically correct patterns. Very low density scores higher on the AI scale.

Each metric is normalized to 0.0 to 1.0 and averaged into a single stylometric score.

**Output:** A float between 0.0 and 1.0, computed entirely in Python with no external dependencies.

**What it misses:** Formal human writing (academic papers, professional blog posts, legal documents) shares statistical properties with AI text and will score high on this signal. Non-native English speakers who write deliberately and carefully may also score higher than native speakers writing casually.

---

## Confidence Scoring

### Combining signals

```
confidence = (llm_score * 0.65) + (stylo_score * 0.35)
```

The LLM signal receives more weight because it captures semantic and stylistic coherence holistically. The stylometric signal is a useful structural cross-check but is noisier on short texts and formal writing.

### Score ranges and label mapping

| Score range | Attribution | Label shown |
|-------------|-------------|-------------|
| 0.65 to 1.00 | likely_ai | High-confidence AI label |
| 0.36 to 0.64 | uncertain | Uncertain label |
| 0.00 to 0.35 | likely_human | High-confidence human label |

The uncertain band is intentionally wide. A score of 0.60 leans slightly toward AI but does not meet the threshold for a verdict. Anything in the uncertain range gets the benefit of the doubt and is directed toward the appeals workflow rather than flagged.

### Validation: two example submissions with different confidence scores

**High-confidence AI (confidence: 0.7096)**
```
Text: "Artificial intelligence represents a transformative paradigm shift in modern
society. It is important to note that while the benefits of AI are numerous, it is
equally essential to consider the ethical implications."

llm_score:   0.8
stylo_score: 0.5417
confidence:  0.7096
attribution: likely_ai
```

**High-confidence human (confidence: 0.2833)**
```
Text: "ok so i finally tried that new ramen place downtown and honestly?
underwhelming. the broth was fine but they put WAY too much sodium in it and i was
thirsty for like three hours after."

llm_score:   0.2
stylo_score: 0.4379
confidence:  0.2833
attribution: likely_human
```

The AI text scored 0.71 and the human text scored 0.28, a clear separation across both signals. The stylometric signal alone shows the AI text is more uniform (0.54 vs 0.44), and the LLM signal shows the strongest separation (0.8 vs 0.2).

---

## Transparency Label

All three variants are returned verbatim in the `label` field of the API response.

**High-confidence AI (confidence >= 0.65):**
> "Our system found strong indicators that this content was likely generated by an AI writing tool. This does not mean your work has been removed. If you wrote this yourself, you can submit an appeal and a reviewer will take a closer look."

**Uncertain (0.36 <= confidence < 0.65):**
> "Our system was not able to confidently determine whether this content was written by a person or an AI tool. No action has been taken. If you believe this result is wrong, you can submit an appeal with context about your writing process."

**High-confidence human (confidence < 0.36):**
> "Our system found strong indicators that this content was written by a person. Thank you for sharing your work."

---

## Rate Limiting

Rate limiting is applied to `POST /submit` via Flask-Limiter:

```
10 requests per minute
100 requests per day
```

**Reasoning:** A working writer submitting their own content would rarely exceed 2 to 3 submissions in a short window. The 10 per minute limit is generous enough to accommodate legitimate bursts (submitting a few pieces in a session) while stopping any scripted flood of submissions. The 100 per day limit reflects a realistic upper bound for a prolific user on a creative platform. Together these limits make automated scraping or abuse expensive without affecting normal use at all.

**Rate limit behavior (429 responses after limit is hit):**

```
200
200
200
200
200
200
200
200
200
200
429
429
```

---

## Audit Log

Every attribution decision is written to `audit_log.json` as a structured JSON entry. The log captures: timestamp, content_id, creator_id, attribution result, combined confidence score, individual signal scores, transparency label, status, and any appeal reasoning.

Sample entries (three representative entries from live testing):

**Entry 1: AI-generated content classified**
```json
{
  "content_id": "b72153b1-8f17-4675-a9a4-7e75009a17fc",
  "creator_id": "test-user-1",
  "timestamp": "2026-06-29T03:34:57.206299+00:00",
  "attribution": "likely_ai",
  "confidence": 0.7096,
  "llm_score": 0.8,
  "stylo_score": 0.5417,
  "label": "Our system found strong indicators that this content was likely generated by an AI writing tool. This does not mean your work has been removed. If you wrote this yourself, you can submit an appeal and a reviewer will take a closer look.",
  "status": "classified",
  "appeal_reasoning": null,
  "appeal_timestamp": null
}
```

**Entry 2: Human-written content classified**
```json
{
  "content_id": "f7c80f6d-bfeb-413d-83df-6efa6d0a701b",
  "creator_id": "test-user-2",
  "timestamp": "2026-06-29T03:35:07.312361+00:00",
  "attribution": "likely_human",
  "confidence": 0.2833,
  "llm_score": 0.2,
  "stylo_score": 0.4379,
  "label": "Our system found strong indicators that this content was written by a person. Thank you for sharing your work.",
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
  "appeal_timestamp": "2026-06-29T03:38:11.448734+00:00"
}
```

**Entry 3: Rate limit test submission**
```json
{
  "content_id": "2ea8ee1c-735e-4d3d-a1f6-0d1bcfcbb9ec",
  "creator_id": "ratelimit-test",
  "timestamp": "2026-06-29T03:38:42.400977+00:00",
  "attribution": "likely_human",
  "confidence": 0.305,
  "llm_score": 0.2,
  "stylo_score": 0.5,
  "label": "Our system found strong indicators that this content was written by a person. Thank you for sharing your work.",
  "status": "classified",
  "appeal_reasoning": null,
  "appeal_timestamp": null
}
```

Entry 2 shows an appeal in the log: the status changed from `classified` to `under_review` and the creator's reasoning is stored alongside the original classification decision.

---

## Known Limitations

**Formal human writing is the system's biggest reliability problem.** Academic essays, professional blog posts, and legal writing share statistical properties with AI text: consistent sentence length, broad but not diverse vocabulary, and low punctuation density outside of commas. A graduate student submitting a carefully written personal statement could score 0.55 to 0.65 on this system, landing in the uncertain band or potentially crossing into the AI threshold. The stylometric signal is the main driver of this problem since it cannot distinguish deliberate formality from AI uniformity. The LLM signal partially compensates by detecting personal grounding and genuine specificity, but not reliably for all formal writing styles. The wide uncertain band (0.36 to 0.64) exists specifically to catch these cases and route them toward human review rather than an automated verdict.

---

## Spec Reflection

**Where the spec helped:** Writing out the three label variants in `planning.md` before any implementation code was the most useful thing in the spec. When it came time to write the label generator function in M5, the exact strings were already decided. There was no guessing at what threshold should say what or how to phrase uncertainty for a non-technical reader. The spec turned a UX decision into a lookup table.

**Where implementation diverged:** The spec defined stylometric scoring with three metrics averaged equally. During testing, very short texts (under 20 words) produced unreliable scores because sentence length variance and TTR don't normalize well with so few data points. The implementation added an early return of 0.5 for texts below a minimum length threshold, which was not in the original plan. This was the right call since returning a false confident score on short text is worse than returning a neutral one, but it was not anticipated during planning.

---

## AI Usage

**Instance 1: Flask app skeleton and Groq signal function**

I provided the detection signals section of `planning.md` and the architecture diagram to Claude and asked it to generate the Flask app skeleton with a `POST /submit` route stub and a Groq signal function that returned a float. The output gave me a working skeleton but the prompt it generated for Groq asked for a plain text response rather than structured JSON, which meant parsing was fragile. I revised the prompt to require a JSON object with an `ai_probability` key and added a code fence stripping step before `json.loads()` since the model sometimes wraps responses in markdown backticks even when told not to.

**Instance 2: Stylometric signal and confidence scorer**

I provided the Signal 2 description and the uncertainty representation section from `planning.md` and asked Claude to generate the stylometric function computing sentence variance, TTR, and punctuation density, plus the weighted scoring logic. The generated scorer used a simple 0.5 threshold to assign a binary attribution label rather than preserving the three-band structure I had defined. I overrode this and rewrote the attribution logic to use the 0.35 and 0.65 thresholds from the spec, keeping the score as a continuous float and only mapping to attribution and label at the response layer.