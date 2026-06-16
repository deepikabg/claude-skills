# Eval Craft Reference

The full method behind the `eval-spec` skill. Read when a step needs depth. Sources: an AI-native eval playbook, Google's Agent Quality framework, and the Husain/Shankar error-analysis method.

## 1. Error analysis (the most important activity)

Evals are derived from real failures, not invented top-down. The flow:

- **Get traces.** Use real interactions if any exist. Otherwise generate **structured synthetic data via dimensions**: pick the dimensions that vary outcomes (e.g. recipe app: Dietary Restriction × Cuisine × Query Complexity), enumerate tuples, turn each into a natural-language query. Structured beats "give me 20 test queries" — it guarantees coverage of the corners.
- **Open coding.** One domain expert (a "benevolent dictator," not a committee) reviews 20–50 traces (50–100 for production) and writes free-form notes on what's wrong. No fixed categories yet — this is qualitative journaling.
- **Axial coding → taxonomy.** Cluster the notes into a failure taxonomy that is **mutually exclusive** (a failure fits exactly one category), **mechanism-distinct** (two categories can't be collapsed), and **root-cause not symptom** ("retrieved wrong context," not "hallucination").
- **Prioritize by frequency × impact.** Fix obvious one-line bugs immediately — don't write an eval for something a prompt tweak solves. Write evals for the high-frequency / high-impact categories that remain.

## 2. The three eval levels (classify before you instrument)

| Bucket | Nature | Instrument |
|---|---|---|
| Deterministic test | One correct answer | Unit/integration test, pass/fail |
| Probabilistic eval | Many valid outputs | Gold set + binary criteria + rubric / LLM-judge |
| Guardrail (must-never) | Catastrophic if ever true | Red-team suite, zero-tolerance pass/fail |

Mismatch is the classic trap: fuzzy evals on deterministic features = overhead; pass/fail tests on probabilistic features = false assurance.

## 3. Gold-standard datasets

50–200 hand-curated cases reflecting *real* scenarios (not sanitized demos): typical + edge + adversarial. The PM/domain expert writes them — the act of writing forces clarity on what "good" means. **Quality over quantity:** 50 sharp cases beat 10,000 synthetic ones.

## 4. Binary over Likert

Prefer clear pass/fail. 1–5 Likert scales bury disagreement and drift between raters. If a dimension feels scalar, decompose it into specific binary questions ("Includes the decision? Y/N", "Omits PII? Y/N"). A suspiciously high score (99%+) means the eval is too easy, not that the system is excellent.

## 5. LLM-as-Judge / Agent-as-Judge

- **LLM-as-Judge** for qualitative outputs: give the judge the input, output, golden reference (if any), and a detailed rubric; require written reasoning. Use it for hypothesis/pattern detection and scale — **never** as the final ship decision.
- **Agent-as-Judge** for process: plan quality, tool selection and arguments, context handling.
- Cheap code metrics are the **first filter** to catch obvious failures at scale before escalating to LLM-judge, then human.

## 6. Google Agent Quality — trajectory evaluation

"The trajectory is the truth" — an agent can produce a right-looking answer through broken reasoning.

- **Outside-In:** start end-to-end (black box) — did the final outcome meet the goal? Then go **Inside-Out** (glass box) — inspect the execution trace.
- **Four Pillars** to spec per agent:
  - **Effectiveness** — did it achieve the goal?
  - **Efficiency** — tokens/cost, latency, number of steps.
  - **Robustness** — graceful under API timeout, ambiguous input, missing data (retries/asks/reports vs. crashes/hallucinates).
  - **Safety** — the non-negotiable gate: stays in bounds, refuses harmful, resists prompt injection and data leakage.

## 7. Segment slices (bias is a product risk)

Global averages hide segment failures. For each eval, define slices by the segments that matter for this product (user type, geo, language, content type) and ask **"who is most likely to be harmed by a wrong output here?"** Add slices for those segments, not just an aggregate. Watch for distribution/representation/objective bias in the model, and selection/confirmation/attention bias where human reviewers amplify it.

## 8. Thresholds, HITL, cadence

- **Ship/no-ship threshold per dimension** (e.g. ≥90% recall on high-risk cases; 0 guardrail violations on the red-team suite; median expert pass on the gold set).
- **Tie HITL level to risk:** high → human approves (with an interruption/approval step before high-stakes tool calls); medium → AI acts, humans sample + handle escalations; low → AI acts with monitoring.
- **Error-analysis cadence:** new systems re-run weekly until patterns stabilize; mature systems monthly, plus after every incident, complaint spike, or drift alert.
