# PM-L01: Why AI Systems Need Different Quality Thinking

**Module 1: Why AI Evals Are Different** | Source: w1-l1 | ~10 min read + exercise

---

## Part 1: The Concepts

### The problem with testing AI like software

Traditional software is deterministic. Same input, same output, every time. A unit test runs once, passes, and you ship. If `calculate_tax(100)` returns `7.50` today, it returns `7.50` tomorrow, on every server, forever.

AI systems don't work that way. Run the same input through an LLM five times and you might get five different outputs — not because anything is broken, but because LLMs select tokens probabilistically. Each response involves rolling weighted dice. Same input, different output. This isn't a bug. It's a fundamental property.

This breaks every assumption traditional testing relies on: assert exact output equality, regression-test a single example, run tests once and ship. None of that works when the system can produce different answers each time.

### Two metrics that change everything

Traditional QA asks: **does this input produce the correct output?**
AI evaluation asks: **what percentage of the time does this input produce a correct output?**

Two metrics capture this distinction:

**pass@k** — "Can the system do this at all?" Run it k times. Did it succeed **at least once**? This is a binary yes/no per test case — it doesn't care whether the system succeeded 1 out of 5 times or 5 out of 5. Both count as pass.

**reliable@k** — "Does the system do this every time?" Were **all** k runs correct? Also binary per test case — anything less than 100% consistency is a fail.

Here's a concrete example. You run 4 test cases through an AI system, 5 times each:

| Test case | Runs correct | pass@5 | reliable@5 |
|-----------|-------------|--------|------------|
| A | 5 out of 5 | Yes (1) | Yes (1) |
| B | 3 out of 5 | Yes (1) | No (0) |
| C | 1 out of 5 | Yes (1) | No (0) |
| D | 0 out of 5 | No (0) | No (0) |

Notice that Test B (3/5) and Test C (1/5) both get the same pass@5 score — the metric doesn't distinguish between them. It only asks: "is the system *capable* of handling this case?" Both are capable. Neither is reliable.

The **aggregate** metrics are calculated across all test cases:
- **pass@5** = 3 out of 4 test cases passed = **0.75**
- **reliable@5** = 1 out of 4 test cases were reliable = **0.25**
- **The gap** = 0.75 − 0.25 = **0.50**

The gap is your consistency risk:

| Gap | Diagnosis | Action |
|-----|-----------|--------|
| Small (< 0.2) | Capable and consistent | Ship candidate |
| Large (0.3-0.6) | Capable but unreliable | Hold — fix reliability before shipping |
| Both metrics low | Not capable enough | Model quality problem — different fix needed |

A team that reports "pass@5 = 0.95" is telling the truth — the system almost always succeeds *eventually*. But if reliable@5 is 0.40, users will encounter incorrect or inconsistent results in roughly 60% of interactions. Both numbers are true. The gap between them is what the PM needs to see.

Different gaps point to different problems requiring different solutions. This makes the gap diagnostic, not just descriptive.

### "Just set temperature to zero"

Your engineer will suggest this. It helps — but even at temperature zero, different server hardware produces slightly different floating-point calculations, documented at 3-8% inconsistency. Traditional software runs identically on any hardware. AI systems don't. Variance can be reduced but not assumed away. It must be measured.

### Benchmarks screen. Product eval decides.

Google's Bard passed internal benchmarks and gave a factually wrong answer on live TV — costing Alphabet ~$100B in market cap in a single day. IBM Watson for Oncology showed promising results in controlled settings but produced unsafe treatment recommendations on diverse real patients. Both systems passed their internal evaluations. Both failed in production.

Standard benchmarks measure capability on clean, well-formed test problems. They don't represent the messy reality of user inputs — ambiguity, typos, edge cases, adversarial behavior. High benchmark scores are necessary but not sufficient. After passing benchmarks, you need product-level evaluation on representative real-world data.

### What this means for you as a PM

You can no longer ask "does the system work?" and expect a yes/no answer. Instead, you ask:
- **How often** does it get the right answer?
- **On which input types** does it struggle?
- **How confident** is it when it's wrong?
- **What's the cost** when it fails?

These questions require data. Which brings us to the exercise.

---

## Part 2: Exercise — Calculate pass@5 and reliable@5

Your engineering team ran a controlled test on the Delivery Hero menu verification system: they took 20 menu changes and ran each one through the LLM verifier **5 times**. Every run is logged with the LLM's decision, confidence score, and reasoning. The ground truth (correct answer) is also recorded.

**Your job:** Calculate pass@5 and reliable@5, identify where the system fails, and make a ship/ramp/hold decision.

### Setup

Open `exercises/dh-multirun-test.csv`. This file has 100 rows: 20 test cases × 5 runs each.

Key columns:
- `test_id` — identifies which menu change (T-01 through T-20)
- `run_number` — which of the 5 runs (1-5)
- `change_type` — what kind of change: price, description, allergen_dietary, image, or category
- `ground_truth` — the correct decision: **approve** or **reject** (what a human QC agent decided)
- `llm_decision` — what the LLM decided: approve, reject, or **flag_for_review**
- `llm_confidence` — how confident the LLM was (0.0-1.0)

Each change type has its own old/new columns (e.g., `old_price`/`new_price` for price changes, `old_allergen_dietary`/`new_allergen_dietary` for allergen changes). Only the relevant pair is populated per row.

**How to score correctness:** Ground truth is always binary — humans approve or reject. The LLM has a third option: flag_for_review (send to a human). Score each run like this:

| LLM decision | Ground truth | Correct? | Why |
|-------------|-------------|----------|-----|
| approve | approve | Yes | Right call |
| reject | reject | Yes | Right call |
| flag_for_review | reject | Yes | Rightly cautious — caught a bad change |
| flag_for_review | approve | No | Overcautious — wasted human reviewer time |
| approve | reject | **No** | **Dangerous — bad change went live** |
| reject | approve | No | Friction — good change blocked |

### Step 1: Score each test case

For each of the 20 test cases (T-01 through T-20), compare `llm_decision` to `ground_truth` across all 5 runs. Fill in:

| test_id | Item | Change type | Runs correct (out of 5) | pass@5 (≥1 correct?) | reliable@5 (all 5 correct?) |
|---------|------|-------------|------------------------|----------------------|---------------------------|
| T-01 | | | /5 | Yes / No | Yes / No |
| T-02 | | | /5 | Yes / No | Yes / No |
| ... | | | | | |
| T-20 | | | /5 | Yes / No | Yes / No |

### Step 2: Calculate the aggregate metrics

- **pass@5** = (number of test cases with at least 1 correct run) / 20 = ___
- **reliable@5** = (number of test cases with all 5 runs correct) / 20 = ___
- **The gap** = pass@5 − reliable@5 = ___

### Step 3: Diagnose the pattern

Look at the test cases that are *capable but unreliable* (pass@5 = Yes, reliable@5 = No). What do they have in common?

- What `change_type` are they? Is one type more unreliable than others?
- For price changes: what `price_markup_pct` range are they in? Do they cluster around the 25% policy threshold?
- For non-price changes (allergen, description, image): what makes these hard for the LLM?
- Are there cases where the system isn't even capable (0/5 correct)? What's different about those?

Write down: "The system is unreliable on ___ types of changes because ___."

### Step 4: Check the confidence signal

Compare `llm_confidence` for correct decisions vs incorrect ones:
- Average confidence when the LLM was correct: ___
- Average confidence when the LLM was wrong: ___

Is there a gap? If the system is less confident when it's wrong, you could use confidence as a routing signal — sending low-confidence decisions to humans instead of auto-actioning them.

---

## Part 3: PM Decision Point

Your VP asks: "Is the menu verifier ready to ship?"

Based on the pass@5, reliable@5, and gap you just calculated, write a 4-sentence memo:

1. **Capability:** "The system can handle ___% of test cases correctly (pass@5 = ___)."
2. **Reliability:** "However, it produces consistent correct results only ___% of the time (reliable@5 = ___)."
3. **Risk:** "The gap of ___ is concentrated in [describe the pattern you found] — meaning [business impact]."
4. **Recommendation:** "I recommend [ship / ramp / hold] with the condition that [specific mitigation based on what you found]."

Refer to the gap interpretation table from Part 1 to calibrate your recommendation.
