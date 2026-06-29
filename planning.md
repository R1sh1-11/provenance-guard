# Provenance Guard — Planning Document

## Architecture Narrative

A piece of text enters the system through POST /submit. The endpoint validates the request, then passes the raw text to two independent detection signals running in sequence. Signal 1 sends the text to Groq (llama-3.3-70b-versatile) and gets back a probability that the content is AI-generated. Signal 2 runs stylometric heuristics directly in Python, computing three statistical properties of the text and combining them into a second probability. The confidence scorer takes both signal outputs and produces a single calibrated score between 0 and 1. The label generator maps that score to one of three plain-language transparency labels. The final response, along with both signal scores, the combined confidence, and the label, gets written to the audit log before being returned to the caller.

An appeal enters through POST /appeal. The endpoint looks up the original content_id, updates its status to "under_review", appends the appeal reasoning to the audit log entry, and returns a confirmation. No automated re-classification happens. A human reviewer would open GET /log and see the original decision alongside the creator's reasoning.

## Architecture Diagram

```
SUBMISSION FLOW
===============

POST /submit
(text, creator_id)
       |
       v
  LLM Signal (Groq)
  "Is this AI-generated?"
  output: float 0.0 to 1.0
       |
       v
  Stylometric Signal (Python)
  sentence variance + TTR + punctuation density
  output: float 0.0 to 1.0
       |
       v
  Confidence Scorer
  weighted average of both signals
  output: calibrated float 0.0 to 1.0
       |
       v
  Label Generator
  maps score to one of three label variants
  output: plain-language string
       |
       v
  Audit Log
  writes: content_id, creator_id, timestamp,
          llm_score, stylo_score, confidence, label, status
       |
       v
  JSON Response
  (content_id, attribution, confidence, label)


APPEAL FLOW
===========

POST /appeal
(content_id, creator_reasoning)
       |
       v
  Lookup content_id in audit log
       |
       v
  Update status to "under_review"
  Append creator_reasoning to log entry
       |
       v
  JSON Response
  (appeal received, status: under_review)
```

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Whether the text reads as AI-generated based on semantic coherence, phrasing patterns, structural predictability, and vocabulary choices. The model evaluates holistically, the same way a human reader might notice something "feels off."

**Output format:** A float between 0.0 and 1.0 representing probability of AI authorship. Extracted from a structured JSON response prompted from the model.

**What it misses:** Heavily edited AI output that a human has reworked substantially. Also, AI models prompted to write "casually" or "imperfectly" can fool this signal. The LLM signal is also not deterministic, so identical inputs can produce slightly different scores across calls.

### Signal 2: Stylometric Heuristics (Python)

**What it measures:** Three statistical properties that differ reliably between AI and human writing:

1. **Sentence length variance:** AI text tends to produce sentences of similar length. Human writing is irregular. Low variance = higher AI probability.
2. **Type-token ratio (TTR):** Vocabulary diversity. AI text reuses common words more than humans do in short passages. Low TTR = higher AI probability.
3. **Punctuation density:** Measured as punctuation marks per 100 words. AI text tends to use punctuation in predictable, grammatically correct patterns. Extremely low or formulaic density = higher AI probability.

Each metric is normalized to a 0.0 to 1.0 range and averaged into a single stylometric score.

**Output format:** A float between 0.0 and 1.0.

**What it misses:** Formal human writing (academic papers, legal documents, professional blog posts) shares statistical properties with AI text and will score high on this signal. Non-native English speakers who write in a deliberate, careful style may also score higher than native speakers with casual writing habits.

## Confidence Scoring and Uncertainty Representation

### Combining signals

The two signals are combined using a weighted average:

```
confidence = (llm_score * 0.65) + (stylo_score * 0.35)
```

The LLM signal gets more weight because it captures semantic and stylistic coherence holistically. The stylometric signal is a useful cross-check but is noisier on short texts and formal human writing.

### What the score means

| Score range | Interpretation |
|-------------|---------------|
| 0.00 to 0.35 | Likely human-written |
| 0.36 to 0.64 | Uncertain, cannot confidently classify |
| 0.65 to 1.00 | Likely AI-generated |

A score of 0.60 means the system leans slightly toward AI but does not have enough signal to say so confidently. The label for that score communicates uncertainty, not a verdict. A score of 0.90 means both signals agree strongly and the label reflects that.

### False positive priority

A false positive (labeling a human writer's work as AI) is treated as worse than a false negative. The uncertain band (0.36 to 0.64) is intentionally wide. Anything in that range gets the uncertain label rather than a verdict, giving creators the benefit of the doubt and directing them toward the appeals workflow.

## Transparency Label Variants

These are the exact strings the system returns in the label field:

**High-confidence AI (confidence >= 0.65):**
> "Our system found strong indicators that this content was likely generated by an AI writing tool. This does not mean your work has been removed. If you wrote this yourself, you can submit an appeal and a reviewer will take a closer look."

**Uncertain (0.36 <= confidence < 0.65):**
> "Our system was not able to confidently determine whether this content was written by a person or an AI tool. No action has been taken. If you believe this result is wrong, you can submit an appeal with context about your writing process."

**High-confidence human (confidence < 0.36):**
> "Our system found strong indicators that this content was written by a person. Thank you for sharing your work."

## Appeals Workflow

**Who can appeal:** Any creator who submitted content and received a content_id.

**What they provide:** The content_id of their submission and a plain-text explanation of their writing process or why they believe the classification is wrong.

**What the system does on appeal:**
1. Looks up the original audit log entry by content_id.
2. Updates the status field from "classified" to "under_review".
3. Appends the creator_reasoning and an appeal_timestamp to the log entry.
4. Returns a confirmation JSON response.

**What a reviewer sees in GET /log:**
The original entry with attribution result, confidence score, both signal scores, and the creator's appeal reasoning all in one place. The status field being "under_review" signals that this entry needs human attention.

No automated re-classification happens. The reviewer makes the final call.

## Anticipated Edge Cases

**1. Formal human writing scored as AI**
A graduate student submitting a carefully written personal essay with consistent sentence structure, low colloquial vocabulary, and grammatically correct punctuation will score high on the stylometric signal. The LLM signal may also flag it depending on topic and phrasing. This is the system's biggest reliability problem and the main reason the uncertain band is wide.

**2. Short texts producing unreliable scores**
A poem of four lines does not give the stylometric signal enough data to compute meaningful sentence length variance or a reliable type-token ratio. Both metrics normalize poorly on very short inputs. The system should note in its response when text is below a minimum length threshold (roughly under 50 words), but will still return a score rather than refusing.

**3. Non-native English speakers**
Deliberate, careful writing from someone who is not a native English speaker may pattern-match to AI writing on both signals. Formal vocabulary choices, consistent sentence length, and low punctuation variation are common in this population. The appeals workflow is the primary mitigation.

## AI Tool Plan

### M3: Submission endpoint and first signal
**Spec sections provided to AI:** Detection signals section (Signal 1 only), Architecture diagram (submission flow).
**What to ask for:** Flask app skeleton with POST /submit route stub, plus the Groq signal function that returns a float.
**Verification:** Call the Groq function directly with the four test inputs from the spec. Print the raw output and confirm it returns a parseable float, not a string or dict.

### M4: Second signal and confidence scoring
**Spec sections provided to AI:** Detection signals section (Signal 2), Confidence scoring section, Architecture diagram.
**What to ask for:** Stylometric signal function computing sentence variance, TTR, and punctuation density, plus the scoring logic combining both signals with the 0.65/0.35 weighting.
**Verification:** Run all four test inputs from the spec through both signals independently. Check that clearly AI text scores above 0.65 combined and clearly human text scores below 0.35. If not, print both signal scores separately and find which one is miscalibrated.

### M5: Production layer
**Spec sections provided to AI:** Transparency label variants section, Appeals workflow section, Architecture diagram (both flows).
**What to ask for:** Label generation function mapping confidence float to one of the three label strings, plus the POST /appeal endpoint.
**Verification:** Submit inputs that hit each of the three score ranges and confirm the label text matches exactly. Submit an appeal using a content_id from a prior submission, then call GET /log and confirm the status changed to "under_review" and creator_reasoning is present.