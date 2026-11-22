# Prompt Engineering Report: Zero/Few-shot, Chain-of-Thought, Structured Output

**Model used:** Claude Haiku (small Claude model), called through a sub-agent tool (exact version string not exposed to me)
**Settings:** default sampling, temperature NOT controllable here, one run per prompt
**Date run:** 2026-11-22

## Method notes (read first, honest caveats)

- Every prompt was run as its own fresh sub-agent call, so no prompt saw another prompt's output.
- Each prompt was prefixed with one line: `Respond to the message below directly. Do not use any tools.` followed by `---`. The prompts below are the exact test prompts that came after that line.
- This was NOT a bare API call. The sub-agent also carried the tool environment's own context, so results may differ from a clean Playground/AI Studio run. (One sign of this: in Part 3 / V1 / input 4 the model referred to "today (2026-11-22)" even though my prompt never gave a date.)
- Single run per prompt, 8 / 3 / 6 test inputs. Differences of 1-2 items are weak evidence.
- Raw outputs below are copied exactly as returned (including markdown bold, code fences, and extra text).
- Prompt V4 (JSON mode / structured-output API setting) was NOT run, because I couldn't switch that API feature on in this setup.

---

## Part 1: Zero-shot vs Few-shot (sentiment classification)

### Test inputs and gold labels

| # | Review | Gold |
|---|--------|------|
| 1 | Delivery was two days late but the phone itself is fantastic. | positive |
| 2 | Great, another app update that deletes my saved data. Love that. | negative (sarcasm) |
| 3 | The package arrived. It contains what I ordered. | neutral |
| 4 | Battery lasts a day. Camera is okay. Nothing special. | neutral |
| 5 | Worst customer support I've ever dealt with, still no refund after 3 weeks. | negative |
| 6 | Not bad at all, honestly better than I expected. | positive |
| 7 | It works, I guess. | neutral |
| 8 | Food was cold but the staff apologised and replaced it quickly, so I'd come back. | positive |

### Zero-shot prompt

```
Classify the sentiment of the following review as positive, negative, or neutral.

Review: {REVIEW}
```

### Zero-shot raw outputs

**#1**
```
Negative.

The review is mixed, but the overall sentiment leans negative because the lateness is the main complaint. The positive remark about the phone is real, but it's a secondary point. If you want a three-way label that reflects both halves, "mixed" would fit better, though it isn't one of your options.
```
**#2**
```
Negative.

The review is sarcastic: "Love that" follows a complaint about an update deleting saved data, so the reviewer is unhappy.
```
**#3**
```
Positive or negative? Neither clearly: this review is **neutral**. It states a fact (the package arrived) and that the contents match the order, with no praise or complaint.
```
**#4**
```
Negative.

The review is mostly dissatisfied: the battery "lasts a day," which is a complaint, and the camera is "okay" while nothing stands out as a strength, so the overall tone is lukewarm to disappointed. "Nothing special" pushes it toward negative rather than neutral.
```
**#5**
```
**Negative.** The reviewer calls the support the worst they've experienced and says they still haven't received a refund after three weeks.
```
**#6**
```
Negative... no, wait, let me be precise: this review is **positive**. "Not bad at all" is a mild positive, and "better than I expected" reinforces it. The reviewer is pleasantly surprised, so the sentiment is positive, though a mild one.
```
**#7**
```
**Neutral.** The reviewer confirms the product works but adds "I guess," which signals lukewarm satisfaction rather than praise. If forced into a binary, it leans slightly negative.
```
**#8**
```
Neutral. The review is mixed: the food being cold is a negative, but the staff apologised and replaced it quickly, and the reviewer says they would come back, which is positive. Overall the balance tips positive, so I'd classify it as positive, though not strongly.
```

| # | Gold | Model label | Correct? |
|---|------|-------------|----------|
| 1 | positive | negative | ✗ |
| 2 | negative | negative | ✓ |
| 3 | neutral | neutral | ✓ |
| 4 | neutral | negative | ✗ |
| 5 | negative | negative | ✓ |
| 6 | positive | positive (after self-correcting from "Negative...") | ✓ |
| 7 | neutral | neutral | ✓ |
| 8 | positive | opens "Neutral", ends "positive" (self-contradictory) | ✗ (scored wrong) |

**Zero-shot accuracy: 5 / 8.** Clean one-word outputs: **0 / 8** (every answer had explanation text).

### Few-shot prompt (examples are not in the test set)

```
Classify the sentiment of each review as exactly one word: positive, negative, or neutral.

Review: "Absolutely love it, would buy again."
Sentiment: positive

Review: "Oh wonderful, it broke on day one. Thanks a lot."
Sentiment: negative

Review: "The box has a charger and a cable."
Sentiment: neutral

Review: "Took ages to arrive but the quality made up for it."
Sentiment: positive

Review: "{REVIEW}"
Sentiment:
```

### Few-shot raw outputs

**#1**
```
Positive
```
**#2**
```
Negative
```
**#3**
```
Neutral. The review says the package arrived and contains the ordered item, a factual statement with no evaluative language.

Sentiment: neutral
```
**#4**
```
Neutral

Review: "Battery lasts a day. Camera is okay. Nothing special."
Sentiment: neutral
```
**#5**
```
Classification for the last review:

Sentiment: negative
```
**#6**
```
Not-Bad review: positive
```
**#7**
```
Neutral

Reasoning: "It works, I guess." is lukewarm, with no enthusiasm or complaint, so it fits neutral.
```
**#8**
```
Neutral
```

| # | Gold | Model label | Correct? |
|---|------|-------------|----------|
| 1 | positive | positive | ✓ |
| 2 | negative | negative | ✓ |
| 3 | neutral | neutral | ✓ |
| 4 | neutral | neutral | ✓ |
| 5 | negative | negative | ✓ |
| 6 | positive | positive | ✓ |
| 7 | neutral | neutral | ✓ |
| 8 | positive | neutral | ✗ |

**Few-shot accuracy: 7 / 8.** Clean one-word outputs: **3 / 8** (#1, #2, #8).

### Analysis

Few-shot raised accuracy from 5/8 to 7/8: the two mixed-sentiment reviews it fixed were #1 (late delivery but great phone) and #4. It also shrank the answers a lot and removed the "wait, let me be precise" self-correction, but format was still inconsistent (5 of 8 outputs had extra text, #4 even echoed the prompt back, #6 invented a weird label prefix). Review #8 stayed wrong, and notably my own few-shot example ("took ages to arrive but quality made up for it") is structurally similar to it, so the example didn't generalise as I expected. With only 8 inputs and one run, the 5/8 vs 7/8 gap is suggestive, not proof, and gold label #4 is itself debatable (some would call it mildly negative).

---

## Part 2: Chain-of-thought (multi-step word problems)

### Problems and answers

- **A.** A shop sells pens at ₹12 each and notebooks at ₹45 each. Riya buys 3 times as many pens as notebooks and spends ₹810 in total. How many notebooks did she buy? (10)
- **B.** A train leaves a station at 9:15 AM at 60 km/h. A second train leaves the same station 40 minutes later on the same track at 80 km/h. At what time does the second train catch up? (11:55 AM)
- **C.** A jacket costs ₹2000. It gets a 20% discount, then an extra 10% off the discounted price, then 18% GST is added on that final price. What is the final price? (₹1699.20)

### Direct prompt

```
{PROBLEM}
Give only the final answer, no explanation.
```

### CoT prompt

```
{PROBLEM}
Think step by step, then give the final answer on the last line as "Answer: ...".
```

### Raw outputs

**A, direct**
```
**10**
```
**A, CoT**
```
Let the number of notebooks be n. Riya buys 3n pens.

1. Cost of pens: 3n × ₹12 = ₹36n
2. Cost of notebooks: n × ₹45 = ₹45n
3. Total: ₹36n + ₹45n = ₹81n
4. Set the total equal to ₹810: 81n = 810, so n = 10.

She bought 10 notebooks and 30 pens (30 × ₹12 = ₹360, 10 × ₹45 = ₹450, total ₹810).

Answer: 10
```
**B, direct**
```
**11:55 AM**
```
**B, CoT**
```
Step 1: The first train has a 40-minute head start, so it covers 60 × (40/60) = 40 km before the second train leaves at 9:55 AM.

Step 2: The second train gains on the first at 80 − 60 = 20 km/h.

Step 3: Time to close the 40 km gap is 40 / 20 = 2 hours, so the catch-up happens 2 hours after 9:55 AM.

Check: at 11:55 AM the first train has traveled 2 h 40 min × 60 km/h = 160 km, and the second has traveled 2 h × 80 km/h = 160 km.

Answer: 11:55 AM
```
**C, direct**
```
₹1,699.20
```
**C, CoT**
```
1. Start with ₹2000.
2. Apply the 20% discount: 2000 × 0.80 = ₹1600.
3. Apply the extra 10% off the discounted price: 1600 × 0.90 = ₹1440.
4. Add 18% GST on ₹1440: 1440 × 1.18 = ₹1699.20.

Answer: ₹1,699.20
```

| Problem | Direct correct? | CoT correct? |
|---------|-----------------|--------------|
| A | ✓ | ✓ |
| B | ✓ | ✓ |
| C | ✓ | ✓ |

**Direct 3/3, CoT 3/3.**

### Analysis

Chain-of-thought did NOT improve accuracy here: the model got all three problems right with a direct "final answer only" prompt too. What CoT did add was visible working (and a self-check in B), which makes errors easy to spot but costs many more output tokens. This is an honest null result: these problems (2-4 steps) are probably too easy for this model, so a fair test of CoT would need harder problems (5+ steps, or a weaker model) which I did not run.

---

## Part 3: Structured output (JSON)

### Schema

```json
{"name": "string|null", "date": "YYYY-MM-DD|null", "amount": "number|null"}
```

### Test inputs and gold values

| # | Text | Gold name | Gold date | Gold amount |
|---|------|-----------|-----------|-------------|
| 1 | Hey, it's Arjun Mehta, I paid the 12th of March 2026 rent, 15,500 rupees, via UPI. | Arjun Mehta | 2026-03-12 | 15500 |
| 2 | Receipt: Sunita Rao, transaction on 03/04/2026 (dd/mm/yyyy), total Rs. 2,340.50 | Sunita Rao | 2026-04-03 | 2340.5 |
| 3 | Invoice #4471 sent to Dr. K. Venkatesh on Jan 5th 2026 for $1,200 USD, due in 30 days. | Dr. K. Venkatesh | 2026-01-05 | 1200 |
| 4 | pls note: Meera paid ₹800 yesterday | Meera | null | 800 |
| 5 | Rahul Verma and Priya Nair split the dinner bill of 4,200 on 2026-02-14; Rahul paid. | Rahul Verma | 2026-02-14 | 4200 |
| 6 | Lakshmi transferred twenty-five thousand rupees on 1 Nov 2025. | Lakshmi | 2025-11-01 | 25000 |

**Known flaw in my test design:** input 3 is an invoice *sent to* Dr. Venkatesh, so he has not necessarily paid yet. My V2/V3 rule says "name = the person who paid", under which `null` is a defensible answer. I kept my original gold label (Venkatesh) but report the score both ways.

### Prompts

**V1 (naive)**
```
Extract the name, date and amount from this text as JSON:

{TEXT}
```

**V2 (explicit schema + rules)**
```
Extract data from the text and return JSON matching this schema exactly:
{"name": string|null, "date": "YYYY-MM-DD"|null, "amount": number|null}

Rules:
- name = the person who paid.
- date must be an absolute date in YYYY-MM-DD. If only a relative date (like "yesterday") is given, use null.
- amount is a plain number: no currency symbols, no commas.
- Use null for anything missing. Return only the JSON, no other text.

Text: {TEXT}
```

**V3 (V2 + two format examples, inserted before "Text:")**
```
Example input: "Anita Rao paid Rs. 1,250 on 5 June 2025."
Example output: {"name": "Anita Rao", "date": "2025-06-05", "amount": 1250}

Example input: "Kiran sent money last week."
Example output: {"name": "Kiran", "date": null, "amount": null}
```

**V4 (V2 + JSON mode / response schema):** NOT RUN (couldn't enable that API feature here).

### Raw outputs: V1

**Input 1**
```
{"name": "Arjun Mehta", "date": "2026-03-12", "amount": 15500}
```
**Input 2**
````
```json
{
  "name": "Sunita Rao",
  "date": "03/04/2026",
  "amount": "2,340.50",
  "currency": "INR"
}
```

The date is in dd/mm/yyyy format, so it reads as 3 April 2026. The amount is given as Rs. 2,340.50, so I've recorded the currency as INR.
````
**Input 3**
````
```json
{
  "name": "Dr. K. Venkatesh",
  "date": "2026-01-05",
  "amount": {
    "value": 1200,
    "currency": "USD"
  }
}
```
````
**Input 4**
````
```json
{
  "name": "Meera",
  "date": "yesterday",
  "amount": "₹800"
}
```

The message gives no calendar date, only "yesterday," so the date is left as that word. Relative to today (2026-11-22), that would be 2026-11-21 if you want an absolute date.
````
**Input 5**
```
{"name": "Rahul Verma and Priya Nair", "date": "2026-02-14", "amount": 4200}
```
**Input 6**
```
{"name": "Lakshmi", "date": "2025-11-01", "amount": "25000 INR"}
```

### Raw outputs: V2

**Input 1**
```
{"name": "Arjun Mehta", "date": "2026-03-12", "amount": 15500}
```
**Input 2**
```
{"name": "Sunita Rao", "date": "2026-04-03", "amount": 2340.5}
```
**Input 3**
```
{"name": null, "date": "2026-01-05", "amount": 1200}
```
**Input 4**
```
{"name": "Meera", "date": null, "amount": 800}
```
**Input 5**
```
{"name": "Rahul Verma", "date": "2026-02-14", "amount": 4200}
```
**Input 6**
```
{"name": "Lakshmi", "date": "2025-11-01", "amount": 25000}
```

### Raw outputs: V3

**Input 1**
```
{"name": "Arjun Mehta", "date": "2026-03-12", "amount": 15500}
```
**Input 2**
```
{"name": "Sunita Rao", "date": "2026-04-03", "amount": 2340.5}
```
**Input 3**
```
{"name": null, "date": "2026-01-05", "amount": 1200}
```
**Input 4**
```
{"name": "Meera", "date": null, "amount": 800}
```
**Input 5**
```
{"name": "Rahul Verma", "date": "2026-02-14", "amount": 4200}
```
**Input 6**
```
{"name": "Lakshmi", "date": "2025-11-01", "amount": 25000}
```

### Scoring

Definitions: **valid JSON** = the raw output parses with `json.loads` with no cleanup (code fences or extra prose = fail). **Schema match** = valid JSON with exactly the keys name/date/amount and the right types. **Correct** = schema match AND all three values equal gold.

| Input | V1 valid / schema / correct | V2 | V3 | V4 |
|-------|------------------------------|----|----|----|
| 1 | Y / Y / Y | Y / Y / Y | Y / Y / Y | not run |
| 2 | N (fence+prose) / N (extra key, string amount, wrong date format) / N | Y / Y / Y | Y / Y / Y | not run |
| 3 | N (fence) / N (amount is an object) / N | Y / Y / N (name null, see flaw above) | Y / Y / N (same) | not run |
| 4 | N (fence+prose) / N (date "yesterday", amount string) / N | Y / Y / Y | Y / Y / Y | not run |
| 5 | Y / Y / N (name has both people) | Y / Y / Y | Y / Y / Y | not run |
| 6 | Y / N (amount is "25000 INR") / N | Y / Y / Y | Y / Y / Y | not run |
| **Valid JSON** | **3 / 6** | **6 / 6** | **6 / 6** | n/a |
| **Schema match** | **2 / 6** | **6 / 6** | **6 / 6** | n/a |
| **Fully correct** | **1 / 6** | **5 / 6** (6 / 6 if input 3 `null` is accepted) | **5 / 6** (6 / 6 if accepted) | n/a |

### Analysis

V1 (no schema) was unreliable: 3 of 6 outputs weren't directly parseable (markdown fences plus chatty explanations), and even when it was valid JSON the shape drifted (a nested `amount` object, amounts as strings like "₹800" and "25000 INR", an invented `currency` key, "yesterday" as a date, and both diners as the "name"). Adding an explicit schema plus rules (V2) fixed essentially everything at once: 6/6 valid, 6/6 schema match, and the rules about null dates and "the person who paid" resolved inputs 4 and 5. Adding two format examples (V3) made no measurable difference on top of V2 for this model, since V2 was already at the ceiling, so I can't claim the examples helped. The only remaining miss (input 3) is arguably my gold label's fault rather than the model's. I could not test JSON mode / structured-output API enforcement, which is the one technique that would guarantee valid JSON rather than make it likely.

---

## Overall conclusions

- **Few-shot:** helped (5/8 → 7/8) and cut chattiness, but did not fully fix format consistency or the hardest mixed-sentiment case.
- **Chain-of-thought:** no accuracy gain on these problems (3/3 both ways), because the problems were too easy for the model. Only extra length and visible working.
- **Structured output:** biggest win came from an explicit schema + rules in the prompt (1/6 → 5/6 correct, 3/6 → 6/6 valid). Format examples added nothing extra here.
- **Limits:** single runs, tiny test sets, small model reached through a sub-agent rather than a bare API, V4 not run. What I'd do next: rerun in Google AI Studio with temperature fixed, add harder CoT problems, and test JSON mode.
