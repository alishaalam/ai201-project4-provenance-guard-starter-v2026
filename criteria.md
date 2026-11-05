# Acceptance criteria — Provenance Guard

Five criteria that say what "working" means for this service, written in unit 7
**before** you built it.

**All five are yours.** None are given.

An acceptance criterion names a number or something plainly observable. *"The
detector is accurate"* is an opinion — and perfect AI detection isn't possible
anyway, so it's an opinion about something nobody can deliver. *"Of 10 pieces I
wrote myself, at most 1 is labelled high-confidence AI"* is a criterion.

Under each, write a sentence or two on **why that target**. *"I allow at most 1
false positive in 10 because on a writing site a wrong accusation costs more
than a missed detection"* is a real answer.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

**Cover at least three of these five areas.** They're prompts, not a form.

| Area | A question it could answer |
|---|---|
| False positives | How often may a human's writing get called AI? |
| Score spread | How far apart should clearly-AI and clearly-human text score? |
| Label coverage | Is every one of your three labels actually reachable? |
| The appeal path | Does an appeal always change status and always get recorded? |
| Bad input | What happens when `text` is missing, empty, or enormous? |

Two things worth knowing before you pick numbers:

- **The expensive mistake is one-sided.** Missing some AI text costs the site a
  little. Accusing a real writer costs that person a lot. A criterion that
  treats both errors as equally bad is describing a system nobody would ship.
- **Unit 8 tests across three trials, and the target has to hold across all
  three.** Your first signal runs a model, so identical input can score
  differently twice. A target of "at most 1 in 10" against trials of 2, 1, 2 is
  a **miss**.

---
## 1. False positives

Of 10 pieces of writing I wrote myself, at most 1 is labelled high-confidence AI, in each of three trials.

**Why this target:** A wrong accusation costs a real writer far more than a missed detection costs the site. I accept up to one in ten because the unsure band absorbs most borderline writing, and a flagged writer can appeal. I won't accept more, because each extra case is a person wrongly accused.


---

## 2. Score spread

The mean combined score of 5 clearly-AI pieces is at least 0.30 higher than the mean of 5 clearly-human pieces.

**Why this target:** If the two groups sit closer than 0.30, the thresholds at 0.35 and 0.65 can't separate them. The model signal alone is noisy, so I want room to spare.


---

## 3. Label coverage

Across 10 test submissions I can produce each of the three labels (human, unsure, AI) at least once, and text I judge borderline lands in the unsure band (0.35–0.65) in at least 3 of 5 cases.

**Why this target:** A label that can't be reached is a gap in the ranges. The unsure band is where the system admits it doesn't know, so it has to actually catch uncertain text.


---

## 4. The appeal path

In 5 of 5 appeals on real content_ids, the response says `under_review`, and `GET /log` shows an appeal entry for that content_id with the writer's reasoning. An unknown content_id returns 404 and writes no appeal.

**Why this target:** An appeal that sometimes vanishes is worse than no appeal. It has to be 5 of 5.


---

## 5. Bad input

For 6 bad inputs (missing text, empty text, whitespace only, 25,000 characters, 2 words, missing creator_id), 6 of 6 return a 4xx JSON error with an `error` field, and none returns a 500 or crashes the service.

**Why this target:** A service that throws a stack trace on bad input is easy to break, and unit 8 attacks start here.


---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 8 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Uncertain cases land in the unsure band

         The system handles uncertain cases well.

         **Why this target:** ...

         > **Revised in unit 8:** Text I judged borderline scores between
         > 0.4 and 0.6 in at least 4 of 5 cases.
         >
         > **Why revised:** "well" gave me nothing to check. I couldn't
         > reach a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "At most 1 of 10 of my own pieces flagged as AI" → "at most 3
            of 10", because 1 of 10 turned out to be hard.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
