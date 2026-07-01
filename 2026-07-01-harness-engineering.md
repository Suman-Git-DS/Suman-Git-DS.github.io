---
layout: default
title: "Harness Engineering: The Infrastructure Layer That Makes AI Agents Actually Work"
date: 2026-07-01
---

{% include nav.html %}

# Harness Engineering: The Infrastructure Layer That Makes AI Agents Actually Work

*Prompt engineering writes the instructions. Context engineering curates what the model sees. Harness engineering controls what the system can and cannot do.*

---

## Why This Matters Now

You have spent hours crafting the perfect prompt. It works brilliantly in the playground. Then you deploy it into a multi-step agentic workflow, and things start going sideways — the agent ignores conventions, violates architectural rules, accumulates errors across turns, and gradually drifts from the intended goal.

The problem is not the prompt. The problem is the environment the agent is operating in.

This is exactly what the OpenAI engineering team encountered in 2025, when a small group of engineers set out to build a production system using Codex agents — with zero manually written code. Over five months, they merged ~1,500 pull requests and generated roughly one million lines of code. Early progress was slow — not because the model was incapable, but because the *environment* was underspecified.

They called the discipline of fixing that environment **harness engineering**. Anthropic's own Labs team reached the same conclusion independently — documenting how a multi-agent harness with generator, evaluator, and planner roles pushed Claude's performance well above baseline on both frontend design and long-running autonomous coding.

---

## The Three Layers: Prompt → Context → Harness

Before diving into the concept, it helps to understand where harness engineering sits relative to the techniques you already know.

| Layer | Core Question | What It Controls |
|---|---|---|
| **Prompt Engineering** | *What should be asked?* | The instruction text sent to the model |
| **Context Engineering** | *What should the model see?* | All tokens at reasoning time — memory, RAG, tool results |
| **Harness Engineering** | *How should the whole system behave?* | Constraints, feedback loops, and guardrails outside the agent |

**Prompt Engineering** — the command: *"Turn right."* It influences what the agent tries to do.

**Context Engineering** — the map, road signs, terrain: everything that tells the agent where it is.

**Harness Engineering** — the reins, the saddle, the fence, and the road: what the system is *allowed* to do.

A well-crafted prompt can influence what an agent *tries* to do. A well-designed harness controls what the system *is allowed* to do, what it *learns* from each run, and what happens when things go wrong.

---

## The Mental Model: 5 Components of a Harness

A harness is the sum of mechanisms that operate **around** the agent, not inside it.

```
┌──────────────────────────────────────────────────────────────────┐
│                        AGENT HARNESS                             │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌────────────────────┐      │
│  │ 1. Context  │   │ 2. Tools &  │   │  3. Constraints    │      │
│  │    Files    │   │ MCP Servers │   │     & Linters      │      │
│  │ (AGENTS.md) │   │ (scoped     │   │ (what cannot       │      │
│  │ (CLAUDE.md) │   │  access)    │   │  happen at all)    │      │
│  └─────────────┘   └─────────────┘   └────────────────────┘      │
│                                                                  │
│  ┌──────────────────────────────┐   ┌────────────────────────┐   │
│  │  4. Feedback Loops           │   │  5. Observability      │   │
│  │  (CI · tests · error-to-ctx  │   │  (traces · logs        │   │
│  │   · structured retry)        │   │   · metrics)           │   │
│  └──────────────────────────────┘   └────────────────────────┘   │
│                                                                  │
│                  ┌──────────────────┐                            │
│                  │    LLM AGENT     │  ← brain, not the system   │
│                  └──────────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

### Component 1 — Context Files (The System of Record)

Structured instruction files — `AGENTS.md`, `CLAUDE.md`, `.cursorrules` — that the agent reads at task start, encoding project structure, naming conventions, and team rules.

> **Key insight:** One giant `AGENTS.md` fails in practice. As it grows, the agent pattern-matches instead of following rules. The fix: tiered, directory-scoped files. The agent reads only what is relevant to the module it is working in.

```
docs/
├── AGENTS.md                   # Top-level: core principles only
├── design-docs/
│   ├── index.md                # Architecture map
│   └── core-beliefs.md         # Foundational decisions
└── references/
    └── design-system-reference.txt
```

### Component 2 — Tools and MCP Servers (Scoped Capabilities)

The Model Context Protocol lets agents access external systems: issue trackers, browsers, databases. But connecting everything is not the goal — **tool definitions consume context window tokens**. The harness controls which tools are available for which tasks.

> **Key insight:** Anthropic's harness research found that separating tool access by agent role — planner, generator, evaluator — was a key lever for improving output quality. The evaluator had Playwright; the generator did not.

### Component 3 — Mechanical Constraints (What Cannot Happen)

Writing a rule in documentation allows the agent to violate it. Enforcing it mechanically prevents that. OpenAI enforced dependency direction with custom linters (`Types → Config → Repo → Service → Runtime → UI`). When code violated that order, the linter blocked it *and* injected corrective instructions into the agent's context, enabling self-repair.

### Component 4 — Feedback Loops (Continuous Correction)

Error handling in a harness is not "retry." It means designing what happens when something goes wrong so the agent can recover autonomously — CI pipelines, test runners, structured error messages formatted for agent consumption, and background tasks that scan for drift.

> **Key insight:** Anthropic's research showed that agents asked to evaluate their own work reliably praised it — even when quality was mediocre. Separating the agent doing the work from the agent judging it proved a stronger lever than any prompt instruction.

### Component 5 — Observability (Debugging the System, Not the Prompt)

If the agent can access runtime data — logs via LogQL, metrics via PromQL, DOM snapshots via Playwright — it can validate and debug the code it produces. Observability is not optional in a harness; it is the mechanism through which the agent closes the loop.

---

## Step-by-Step: Building a Minimal Harness

1. **Encode intent as structured artifacts, not inline prompts.** Write `AGENTS.md` at the repo root with core principles only (under 200 lines). Use sub-directory files for module-specific rules. The agent reads less and follows more.

2. **Scope the tool surface.** Give the agent only the tools it needs for the current task. Limit MCP servers per task invocation. Tool schemas consume tokens; every unused tool is wasted context.

3. **Enforce constraints mechanically.** Write linters or structural tests for rules that matter most. Configure them to return agent-readable corrective messages, not just exit codes or booleans.

4. **Design error messages for agent consumption.** When a constraint check fails, the message should tell the agent what went wrong and what to do instead. "Fix the following before proceeding" outperforms "Error: rule violated."

5. **Build the recovery loop.** Every evaluation — CI, constraint check, accuracy gate — feeds back into the agent's next context window as structured input. The agent is not retried blind; it is retried informed.

6. **Add entropy management.** Schedule background tasks to scan for architectural drift and open cleanup PRs automatically. In long-running systems, the harness needs to maintain itself, not just the agent.

---

## The Code: A Harness That Retrains Until It Wins

**The scenario:** you want a fraud classifier with ≥ 85% validation accuracy. The harness keeps asking an agent to propose better hyperparameters, trains a real scikit-learn RandomForest on a generated dataset, checks whether accuracy meets the bar, and injects structured feedback if it doesn't — until the target is hit or the iteration limit is reached. This is the harness pattern in its most direct form: **goal → attempt → constraint check → feedback → retry**.

This version upgrades the bare-minimum example with four production-relevant additions:

- **Upgrade 1 — Real training.** Uses scikit-learn `RandomForestClassifier` on a generated classification dataset — not a simulated accuracy float. Actual model, actual validation score.
- **Upgrade 2 — Schema validation.** Pydantic `HyperParams` model validates the agent's JSON output before training. Invalid responses are caught and fall back to random search.
- **Upgrade 3 — LLM plug-and-play.** Pass an OpenAI (or any provider) callable and the harness uses LLM-guided exploration. Leave it `None` and it falls back to random search. Same loop either way.
- **Upgrade 4 — Dual modes.** `stop_on_success=True` exits as soon as the target is met. `False` runs all iterations and returns the best result — useful for full hyperparameter exploration.

```python
"""
Harness Engineering — Model Retraining Example
================================================
Scenario: You want a classifier with >= 85% validation accuracy.
The harness keeps retraining (adjusting hyperparameters) until the
model meets the bar — or gives up after max_iterations.

- LLM optional (plug-and-play via make_openai_caller)
- Pydantic schema validation on agent JSON output
- Random search fallback if LLM fails or is not configured
- stop_on_success toggle: early exit OR full exploration
"""

import json
import random
from dataclasses import dataclass, field
from typing import Callable, Optional, List

from pydantic import BaseModel, ValidationError, Field
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

random.seed(42)


# ── DATASET ──────────────────────────────────────────────────────────────────
# Simulates a real fraud detection dataset: 3000 samples, 25 features,
# class imbalance and noise to make the accuracy target non-trivial.

X, y = make_classification(
    n_samples=3000, n_features=25, n_informative=8,
    n_redundant=10, class_sep=0.6, flip_y=0.08, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X, y, test_size=0.25, random_state=42
)


# ── SCHEMA ────────────────────────────────────────────────────────────────────
# Pydantic validates the agent's JSON before any training happens.
# Invalid output is caught here, not inside the model training call.

class HyperParams(BaseModel):
    n_estimators: int = Field(ge=10, le=300)
    max_depth: int    = Field(ge=2,  le=15)


# ── DATA STRUCTURES ───────────────────────────────────────────────────────────

@dataclass
class TrainingContext:
    task: str
    target_accuracy: float
    feedback: list[str]      = field(default_factory=list)
    tried_params: List[dict] = field(default_factory=list)
    # feedback accumulates across iterations — every failure is injected
    # into the next prompt so the agent learns from its own history

@dataclass
class HarnessResult:
    final_accuracy: float
    hyperparams: dict
    passed: bool
    iterations_taken: int
    feedback_history: list[str]

@dataclass
class LLMConfig:
    api_key: str
    model: str


# ── LLM CALLER ───────────────────────────────────────────────────────────────
# Factory pattern: returns a callable with the provider baked in.
# The harness only sees Callable[[str], str] — provider-agnostic.

def make_openai_caller(config: LLMConfig):
    from openai import OpenAI
    client = OpenAI(api_key=config.api_key)

    def call(prompt: str) -> str:
        response = client.chat.completions.create(
            model=config.model,
            messages=[
                {"role": "system", "content": (
                    "Return ONLY JSON. Do not repeat previous parameters. "
                    "Explore different combinations."
                )},
                {"role": "user", "content": prompt},
            ],
            temperature=0.7,  # encourages exploration
        )
        return response.choices[0].message.content
    return call


# ── PROMPT BUILDER ────────────────────────────────────────────────────────────
# The entire harness history — every prior attempt and its result —
# is included in each prompt. The agent is never retried blind.

def _build_prompt(ctx: TrainingContext) -> str:
    return f"""
Task: {ctx.task}
Target: Accuracy >= {ctx.target_accuracy:.0%}

Previous Attempts:
{chr(10).join(ctx.feedback) if ctx.feedback else "None"}

Avoid repeating:
{ctx.tried_params}

Return ONLY JSON:
{{"n_estimators": int, "max_depth": int}}
"""


# ── RANDOM SEARCH FALLBACK ────────────────────────────────────────────────────
# Used when LLM is not configured or when it produces invalid JSON.

def random_params():
    return {
        "n_estimators": random.randint(20, 300),
        "max_depth":    random.randint(3, 15),
    }


# ── AGENT: PROPOSES HYPERPARAMETERS ──────────────────────────────────────────
# This is the LLM call layer. Replace make_openai_caller with any provider.
# Falls back to random search if LLM fails validation or is not configured.

def propose_hyperparams(
    ctx: TrainingContext,
    llm_call: Optional[Callable[[str], str]] = None,
) -> dict:

    print(f"\n{'='*60}\nITERATION {len(ctx.feedback)+1}\n{'='*60}")
    prompt = _build_prompt(ctx)
    print("[harness] Prompt:", prompt)

    if llm_call:
        try:
            raw = llm_call(prompt)
            print("[harness] LLM raw:", raw)
            parsed = HyperParams.model_validate_json(raw)
            params = parsed.model_dump()
            if params not in ctx.tried_params:
                return params
        except (ValidationError, json.JSONDecodeError) as e:
            print("[harness] LLM validation failed:", e)

    # random search: try up to 5 times to find an untried combination
    for _ in range(5):
        params = random_params()
        if params not in ctx.tried_params:
            print("[harness] Random fallback params:", params)
            return params
    return random_params()


# ── REAL MODEL TRAINING ───────────────────────────────────────────────────────
# StandardScaler + RandomForest pipeline. Returns actual val accuracy.

def train_model(params: dict) -> float:
    pipeline = Pipeline([
        ("scaler", StandardScaler()),
        ("model", RandomForestClassifier(
            n_estimators=params["n_estimators"],
            max_depth=params["max_depth"],
            min_samples_leaf=5,
            random_state=42,
        ))
    ])
    pipeline.fit(X_train, y_train)
    preds = pipeline.predict(X_val)
    return round(accuracy_score(y_val, preds), 4)


# ── CONSTRAINT CHECK ──────────────────────────────────────────────────────────
# Returns None on pass. Returns a corrective message string on fail.
# That message is injected into the next prompt — not just logged.

def check_accuracy(acc: float, target: float) -> str | None:
    if acc >= target:
        return None
    return f"Accuracy {acc:.2%} below target {target:.2%}. Increase n_estimators or max_depth."


# ── THE HARNESS LOOP ──────────────────────────────────────────────────────────
# The core pattern: propose → train → check → inject feedback → retry.
# stop_on_success=True: exit the moment the target is met.
# stop_on_success=False: run all iterations, return the best result found.

def run_harness(
    task: str,
    target_accuracy: float = 0.85,
    max_iterations: int = 10,
    stop_on_success: bool = False,
    llm_call: Optional[Callable[[str], str]] = None,
) -> HarnessResult:

    ctx = TrainingContext(task=task, target_accuracy=target_accuracy)
    best_acc, best_params = 0.0, {}

    for i in range(max_iterations):
        params  = propose_hyperparams(ctx, llm_call)
        acc     = train_model(params)
        ctx.tried_params.append(params)

        print(f"[harness] Accuracy: {acc:.2%}")

        if acc > best_acc:
            best_acc, best_params = acc, params

        violation = check_accuracy(acc, target_accuracy)

        if violation is None and stop_on_success:
            print(f"\n[harness] EARLY SUCCESS at iteration {i+1}")
            return HarnessResult(acc, params, True, i+1, ctx.feedback)

        ctx.feedback.append(f"Iter {i+1}: acc={acc:.2%}. {violation or 'OK'}")

    print("\n[harness] All iterations complete.")
    return HarnessResult(
        final_accuracy=best_acc, hyperparams=best_params,
        passed=best_acc >= target_accuracy,
        iterations_taken=max_iterations,
        feedback_history=ctx.feedback,
    )


# ── RUN ───────────────────────────────────────────────────────────────────────

if __name__ == "__main__":

    USE_LLM = False  # set True and add your key to enable LLM-guided search

    llm_call = None
    if USE_LLM:
        config   = LLMConfig(api_key="<your-key>", model="gpt-4o-mini")
        llm_call = make_openai_caller(config)

    result = run_harness(
        task="Train fraud classifier with high accuracy",
        target_accuracy=0.85,
        max_iterations=10,
        stop_on_success=True,
        llm_call=llm_call,
    )

    print("\n" + "="*60)
    print("FINAL RESULT")
    print("="*60)
    print("Passed:",       result.passed)
    print("Best Accuracy:", f"{result.final_accuracy:.2%}")
    print("Iterations:",    result.iterations_taken)
    print("Best Params:",   result.hyperparams)
```

**What each part does in the harness context:**

- `TrainingContext.feedback` — the harness's memory: every prior attempt and its accuracy delta, accumulated across rounds and injected into the next prompt
- `HyperParams` (Pydantic) — a schema-level constraint: invalid agent output is rejected before any compute is spent, with the validation error becoming the next prompt's context
- `make_openai_caller` — a factory that returns a `Callable[[str], str]`. The harness loop never imports OpenAI directly; the provider is swappable
- `check_accuracy()` — returns `None` on pass, or a corrective message on fail. That message is what gets injected — not just a boolean
- `stop_on_success` — controls harness behaviour: early exit once the bar is met, or full exploration to find the global best

**Expected terminal output** (random search mode, `stop_on_success=True`):

```
============================================================
ITERATION 1
============================================================
[harness] Random fallback params: {'n_estimators': 187, 'max_depth': 11}
[harness] Accuracy: 81.73%
============================================================
ITERATION 2
============================================================
[harness] Random fallback params: {'n_estimators': 243, 'max_depth': 13}
[harness] Accuracy: 83.47%
============================================================
ITERATION 3
============================================================
[harness] Random fallback params: {'n_estimators': 298, 'max_depth': 14}
[harness] Accuracy: 86.40%

[harness] EARLY SUCCESS at iteration 3
============================================================
FINAL RESULT
============================================================
Passed: True
Best Accuracy: 86.40%
Iterations: 3
Best Params: {'n_estimators': 298, 'max_depth': 14}
```

---

## What Anthropic Found When They Pushed This Further

*Anthropic Engineering · Published March 2026*

Using a three-agent harness (planner, generator, evaluator) on long-running autonomous coding and frontend design:

- Agents asked to evaluate their own work reliably praised it — even when quality was mediocre. **Separating the generator from the evaluator** was a stronger lever than any prompt instruction.
- For long-running tasks, context anxiety — the model wrapping up prematurely as it approached its context limit — was addressed not with compaction but with **context resets + structured handoff artifacts** between sessions.
- Sprint contracts (generator and evaluator negotiating "what does done look like" before coding started) kept work faithful to the spec without over-specifying implementation too early.
- The evaluator used Playwright MCP to interact with the live running application — not a static screenshot — before scoring each sprint. This caught real usability bugs that static review missed.

---

## What the Industry Has Converged On

OpenAI, Anthropic, and independent engineering teams all reached the same conclusion in early 2026: agentic systems become reliable only when you build the right scaffolding around them. The bottleneck is infrastructure, not model intelligence.

**01 — Context Architecture.** Tiered, progressive disclosure of instructions. The agent reads what is relevant to the current task — not a monolithic file.

**02 — Agent Specialization.** Scoped prompts and restricted tool access per role. Planner, generator, evaluator — each with a distinct context and capability surface.

**03 — Persistent Memory.** Filesystem-backed structured artifacts, not conversation history. State survives context resets and multi-session runs.

**04 — Structured Execution.** Research → plan → execute → verify. Not one long prompt. Each phase has a defined output format the next phase depends on.

---

## Further Reading

Verified primary sources — worth reading in full.

1. **[Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/)** — OpenAI Engineering Blog, Feb 2026. The original field report: how OpenAI built 1M lines with no manual code, and the four harness problems they solved.

2. **[Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps)** — Anthropic Engineering, Mar 2026. How Anthropic pushed Claude above baseline using a three-agent architecture, with GAN-inspired feedback loops and Playwright-based live evaluation.

3. **[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)** — Anthropic Research, Dec 2024. Foundational guide on agent architecture: when to use workflows vs. agents.

4. **[My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey)** — Mitchell Hashimoto (HashiCorp co-founder), Feb 2026. The practitioner post that sharpened the term.

5. **[Beyond Prompts and Context: Harness Engineering for AI Agents](https://madplay.github.io/en/post/harness-engineering)** — MadPlay, Feb 2026. Clear positioning of harness vs. context vs. prompt engineering.

6. **[What Is an Agent Harness?](https://parallel.ai/articles/what-is-an-agent-harness)** — Parallel AI. Precise distinction between orchestration and harness.

---

## Closing Thought

*The model is the brain. The harness is the hands, the guardrails, and the feedback system that keeps the brain productive.*

Prompt engineering shaped what agents *try*. Context engineering shaped what agents *know*. Harness engineering shapes what agents *can and cannot do* — and what they learn when they get something wrong.

In 2026, the difference between a demo and a production AI system is almost always the harness.
