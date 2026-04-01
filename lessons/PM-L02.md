# PM-L02: Mapping Every Way Your AI System Can Fail

**Module 1: Why AI Evals Are Different** | Source: w1-l2 | ~12 min read + exercise

---

## Part 1: The Concepts

### Why you need a failure map

In PM-L01, you calculated a reliability gap — the system is unreliable on certain inputs. But *where* in the pipeline is it breaking? When an LLM produces a wrong output, you can't tell from that output alone whether:

- The system failed to retrieve the right context (retrieval problem)
- It retrieved the right context but ignored it (reasoning problem)
- It applied the wrong rules (policy interpretation problem)

All three produce the same symptom — a wrong answer — but require completely different fixes. You can't fix what you can't locate.

### The Evaluation Surface Map

An evaluation surface map is a structured inventory of **everything that can go wrong** with your AI system. It has three layers:

**Layer 1: Functional failures** — What breaks during normal use?

Map these by pipeline stage. Most LLM-powered systems follow a pipeline: input parsing → context retrieval → LLM reasoning → output generation → (optional) human review. Each stage has its own failure modes:

- **Input parsing** breaks when the system misinterprets what the user is asking
- **Context retrieval** breaks when relevant information isn't found or irrelevant information is pulled in
- **LLM reasoning** breaks when the model ignores context, hallucinates facts, or applies wrong logic
- **Output generation** breaks when confidence is miscalibrated or explanations don't match decisions

Each stage breaks in its own way. A retrieval fix won't help if the LLM is ignoring the context it retrieves.

**Layer 2: Adversarial surface** — How can users exploit the system?

This isn't a "security review to do later." It belongs on Day 1. Any system with external users has adversarial risk. Research on indirect prompt injection (Greshake et al., 2023) demonstrated success rates above 80% in controlled studies. Categories to consider:

- **Prompt injection:** Crafted inputs that manipulate LLM behavior
- **Policy circumvention:** Finding input patterns that consistently bypass rules
- **Incremental gaming:** Small changes that individually pass checks but collectively violate intent
- **Context poisoning:** Exploiting what gets retrieved to influence the LLM's reasoning

These aren't theoretical. They work. Any system shipped without adversarial surface mapping can be exploited.

**Layer 3: Coverage gaps** — What can't you measure yet?

This is the layer teams skip, and it's the most important. Three types:

- **Unmeasured failure modes:** Failures you suspect exist but can't confirm. Example: you think the LLM sometimes fabricates its reasoning, but you're not systematically cross-checking explanations against actual data.
- **Rare scenarios:** Edge cases not in your test data. Multilingual inputs, unusual formatting, novel input categories. They haven't been tested because they haven't been seen yet.
- **Instrumentation blind spots:** Behaviors you literally cannot observe with current logging. If you don't log what context was retrieved, you can't tell whether wrong outputs stem from bad retrieval or bad reasoning.

**A surface map with zero coverage gaps listed is almost certainly overconfident, not thorough.** Every real system has blind spots. Name them.

### Two discovery methods

You need both:

1. **Architecture-driven prediction** — Examine each pipeline stage's inputs, outputs, and dependencies. Predict what can go wrong. No data required. This catches failures that haven't happened yet.

2. **Trace-driven validation** — Inspect actual production data to confirm predictions and find surprises. A clinical text-to-SQL system found that 78% of failures originated in the generation step, not retrieval — the opposite of what the team predicted. Traces surface surprises that architecture analysis misses.

Prediction first, always. Going straight to traces means you only find what's already broken, not what's about to break.

### The "can evaluate now" vs "needs engineering" split

Everything in your surface map falls into one of two buckets: things you can measure with current instrumentation, and things that need engineering work before evaluation can begin.

The "needs engineering" column becomes your instrumentation roadmap. This is how a surface map becomes an action plan — not just a list of risks, but a prioritized engineering backlog.

---

## Part 2: Exercise — Build Your Evaluation Surface Map

You're the AI PM at Delivery Hero, preparing for a launch readiness review of the menu verification system. Your VP will ask: "What are all the ways this system can fail?" and "What haven't we tested yet?" You need to answer both.

The system's pipeline: **Change parsing → Context retrieval → LLM verification → Decision output → Human review (flagged items only)**

### Setup

Open `exercises/dh-menu-verification-dataset.csv` (200 rows of production shadow-mode data).

Relevant columns:
- `change_type` — price, description, allergen_dietary, image, category
- `llm_decision` — approve, reject, flag_for_review
- `human_decision` — approve or reject (ground truth)
- `human_agrees_with_llm` — TRUE or FALSE
- `failure_type` — when the LLM was wrong: false_approval, context_miss, hallucination, policy_miss, price_error

### Step 1: Predict failure modes (architecture-driven)

Before looking at the data, fill in this table. For each pipeline stage, predict at least 2 ways it could fail for a menu change verifier:

| Pipeline stage | Predicted failure mode 1 | Predicted failure mode 2 |
|---------------|------------------------|------------------------|
| Change parsing | ___ | ___ |
| Context retrieval | ___ | ___ |
| LLM verification | ___ | ___ |
| Decision output | ___ | ___ |

### Step 2: Validate with data (trace-driven)

Now look at the dataset. Filter to rows where `human_agrees_with_llm` = `FALSE`.

- How many failures are there total? ___
- Count by `failure_type`:

| failure_type | Count | Which pipeline stage does this map to? |
|-------------|-------|---------------------------------------|
| false_approval | ___ | ___ |
| context_miss | ___ | ___ |
| hallucination | ___ | ___ |
| policy_miss | ___ | ___ |
| price_error | ___ | ___ |

- Did any failure type surprise you vs your predictions?
- Which pipeline stage accounts for the most failures?

### Step 3: Map the adversarial surface

List 3 ways a restaurant owner could exploit the verifier to get policy-violating changes approved. For each, note which pipeline stage it targets:

| Attack | How it works | Stage targeted |
|--------|-------------|---------------|
| 1. ___ | ___ | ___ |
| 2. ___ | ___ | ___ |
| 3. ___ | ___ | ___ |

Hint: Look at the data for patterns. Are there restaurants that appear multiple times with changes that individually pass but collectively look suspicious?

### Step 4: Identify coverage gaps

List at least 3 things you **cannot** evaluate with the current dataset:

| Gap type | What's missing | What you'd need |
|----------|---------------|----------------|
| Unmeasured | ___ | ___ |
| Rare scenario | ___ | ___ |
| Instrumentation blind spot | ___ | ___ |

---

## Part 3: PM Decision Point

Compile your work into a one-page **Evaluation Surface Map** for the launch readiness review:

> **Functional failures:** [X] failure types identified across [Y] pipeline stages. The highest-risk stage is ___ because ___.
>
> **Adversarial risks:** [3] exploitation patterns identified. The most concerning is ___ because ___.
>
> **Coverage gaps:** [3] gaps identified. The most critical is ___ because without it we cannot measure ___.
>
> **Recommendation:** Before launch, we need to close [specific gap] by [specific instrumentation change]. Current evaluation covers ___% of the failure surface.

This is the artifact that transforms "I think it mostly works" into "here's exactly where it can break and what we can't measure yet."
