# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- DRAFT - edit me into your own words. -->

1. A client POSTs `{text, creator_id}` to `/submit`.
2. `app.py` validates it. Missing, blank or oversized input gets a 4xx JSON error and a logged rejection.
3. `detector.py` (gpt2) reads the text and returns `model_score`, based on how predictable the text is.
4. `stylometry.py` measures the shape of the text and returns `style_score`.
5. `scoring.py::combine_signals` merges the two into `combined_score`.
6. `scoring.py::score_to_label` turns that score into a guess and a plain-language label.
7. `audit.py` appends a line to `logs/audit.jsonl` with all three scores and the status, and the response goes back to the writer. If they disagree, `/appeal` records their reasoning and sets the status to `under_review`.


---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:** How predictable the text is to gpt2 (perplexity), turned into a 0–1 score where higher means more AI-like.

**Why that might differ between human and AI writing:** Generated text picks likely next words, so a language model finds it easy to predict. Human writing surprises it more.

**What it can't see:** Why the text is predictable. Formal or plain human writing also scores as AI-like, lightly edited AI doesn't, and very short text is noisy.

### Signal two — the style signal

**What it measures:** The shape of the text: how much sentence length varies, the ratio of unique words, and how much of the less common punctuation (`; : — ( ) " ! ?`) it uses.

**Why that might differ between human and AI writing:** Generated prose tends toward even sentences and steady vocabulary. Human writing is lumpier.

**What it can't see:** Meaning. Anyone can change the shape on purpose (break up sentences, add dashes or typos). Terse or technical human writing looks uniform, and the unique-word ratio shifts with text length.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:** An equal-weight average of the two scores. If the signals differ by more than 0.6, the result is pulled halfway back toward 0.5, so one signal on its own can't produce a confident label.

**The numbers:** Weights 0.5 / 0.5, disagreement gap 0.6, shrink 0.5, human threshold 0.35, AI threshold 0.65 (all in `config.py`). I first tried 0.6 / 0.4 with a gap of 0.4, but clearly human text then landed in "unsure" and the human label was unreachable.

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:**

My own casual Monday-morning paragraph: the model signal said 0.557 (leaning AI) while the style signal said 0.000 (very human). The gap of 0.557 is under 0.6, so the plain average applies: 0.278, which is the human label. The trade-off: a lower gap would have made this "unsure", but it would also have blocked the human label on text like this. On a formal human paragraph the signals agreed more (0.713 vs 0.442), giving 0.578, which is "unsure" and not a false accusation.

| Input | model | style | combined | label |
|---|---|---|---|---|
| clearly AI | 0.850 | 0.628 | 0.739 | ai |
| clearly human | 0.557 | 0.000 | 0.278 | human |
| formal human | 0.713 | 0.442 | 0.578 | unsure |
| lightly edited AI | 0.524 | 0.484 | 0.504 | unsure |

The last row is a miss: edited AI gets through as "unsure". That is a known cost of keeping false accusations low.


---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00 – 0.35 | We think this was probably written by a person. |
| unsure | 0.35 – 0.65 | We couldn't tell either way. Nothing here suggests a problem, and no action is needed. |
| high-confidence AI | 0.65 – 1.00 | This looks like it may have been written with the help of an AI tool. Our check can be wrong, so if you wrote it yourself, you can ask for a review. |

**Why I worded the "unsure" one this way:**
<!-- DRAFT - edit me. Run the "answer as the writer" prompt on all three first. -->

It says plainly that the system doesn't know, and tells the writer there is nothing to do. Most people who see it wrote every word themselves, so it avoids the word "AI" entirely.


---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

<!-- The response needs content_id, guess, confidence, label, AND both
     signal scores - model_score and style_score. Unit 8's tools read those
     two off the response, so leaving them out costs you next unit's
     diagnostics. -->

```bash
$ curl -X POST http://127.0.0.1:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"creator_id": "w3", "text": "Artificial intelligence is a transformative technology that offers numerous benefits across many industries. It enables organizations to improve efficiency, enhance decision-making, and drive innovation. Furthermore, it provides valuable insights that support strategic planning. Moreover, it helps teams to streamline workflows and reduce operational costs. In conclusion, artificial intelligence represents a significant opportunity for organizations of all sizes."}'
```

```json
{
  "combined_score": 0.7389,
  "confidence": 0.4778,
  "content_id": "d888b763248246858bf8b05cc9359919",
  "guess": "ai",
  "label": "This looks like it may have been written with the help of an AI tool. Our check can be wrong, so if you wrote it yourself, you can ask for a review.",
  "model_score": 0.8498,
  "status": "decided",
  "style_score": 0.6279
}
```

**An appeal**

```bash
$ curl -X POST http://127.0.0.1:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "d888b763248246858bf8b05cc9359919", "reasoning": "I wrote this myself for a class; I tend to write in a formal style."}'
```

```json
{
  "content_id": "d888b763248246858bf8b05cc9359919",
  "message": "Thanks. Your appeal has been recorded and this item is now under review.",
  "status": "under_review"
}
```

**What the log shows afterwards**

Entries for that content_id, from `GET /log`:

```bash
$ curl "http://127.0.0.1:5000/log"
```

```json
[
  {
    "combined_score": 0.7389,
    "content_id": "d888b763248246858bf8b05cc9359919",
    "creator_id": "w3",
    "guess": "ai",
    "label": "This looks like it may have been written with the help of an AI tool. Our check can be wrong, so if you wrote it yourself, you can ask for a review.",
    "model_score": 0.8498,
    "status": "decided",
    "style_score": 0.6279,
    "timestamp": "2026-11-05T00:34:36+00:00"
  },
  {
    "content_id": "d888b763248246858bf8b05cc9359919",
    "creator_id": "w3",
    "event": "appeal",
    "reasoning": "I wrote this myself for a class; I tend to write in a formal style.",
    "status": "under_review",
    "timestamp": "2026-11-05T00:35:01+00:00"
  }
]
```

---

## How I Used AI

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
