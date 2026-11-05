# Milestone 1: Trace

## The path of a submission
1. **Client** POSTs `{text, creator_id}` to `/submit`.
2. **Validation (app.py)** rejects missing, blank, non-string or oversized input with a 4xx JSON error and logs a rejection.
3. **Signal one (detector.py)** gpt2 reads the text and returns `model_score` (0–1, higher = more AI-like) from perplexity.
4. **Signal two (stylometry.py)** measures sentence-length spread, word variety and punctuation density and returns `style_score` (0–1).
5. **Combining (scoring.py::combine_signals)** merges the two into `combined_score`, pulling toward 0.5 when they disagree.
6. **Label (scoring.py::score_to_label)** maps the score to `guess` (human / unsure / ai) and a plain-language label.
7. **Audit (audit.py)** appends a JSON line with timestamp, content_id, creator_id, guess, all three scores, label, status. The response goes back to the client.

## Signals
| | Measures | Why it might differ | Blind spot |
|---|---|---|---|
| Model signal | How predictable the text is to gpt2 (perplexity) | AI text tends to pick high-probability words, so it is more predictable | Formal or boilerplate human writing is also predictable; lightly edited AI is not; very short text is noisy |
| Style signal | Sentence-length spread, unique-word ratio, punctuation density | AI tends to write evenly sized sentences with steady vocabulary | Terse or technical humans look uniform; AI prompted to vary its style looks human; the unique-word ratio shifts with text length |

## False-positive trace
A student submits an essay they wrote in formal academic prose. gpt2 finds it predictable (model_score ~0.7), and the style signal sees even sentences (~0.6). Combined ~0.65, so the label says "probably AI-generated." They see the label, not the numbers. They can POST `/appeal` with their reasoning; the status becomes `under_review` and the appeal is logged next to the original decision. Nothing else changes the label, and no human reviewer exists in this service yet. That is why `AI_THRESHOLD` should sit high and the label wording should be soft.

## Endpoint contracts
### POST /submit
- In: `{"text": string (non-blank, <= 20000 chars), "creator_id": string (non-blank)}`
- Out 200: `{"content_id": str, "guess": "ai"|"human"|"unsure", "confidence": float 0–1, "label": str, "model_score": float, "style_score": float, "combined_score": float, "status": "decided"}`
- Errors (JSON `{"error": code, "message": str}`): 400 missing or invalid field, 413 too long, 422 too short to score, 503 detector unavailable.

### POST /appeal
- In: `{"content_id": string, "reasoning": string (non-blank)}`
- Out 200: `{"content_id": str, "status": "under_review", "message": str}`
- Errors: 400 missing field, 404 unknown content_id.
