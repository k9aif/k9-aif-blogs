---
layout: post
title: "The Validation Loop Pattern — How K9-AIF Agents Reason Until They're Sure"
date: 2026-05-25
author: Ravi Natarajan
---

Most AI agents are one-shot: payload in, result out. That works when the answer is deterministic — classify this, extract that, format this response.

It breaks the moment the answer is uncertain and must be earned.

A fraud detection agent that correlates signals across multiple passes produces meaningfully better risk assessments than one that calls the LLM once and reports. A document extraction agent that checks its own output against a required-fields schema — and re-extracts when fields are missing — is more reliable than one that hopes the first parse was complete. A claims adjudication agent that returns 60% confidence needs somewhere to put that uncertainty: re-query, observe, decide again.

The pattern is **hypothesis → tool/test → observation → re-reason → continue or finalize**. It is domain-agnostic. Every team that builds iterative agents reinvents it. Most do it badly — ad-hoc while loops, uncapped retries, no structured step history, no clean escalation path.

K9-AIF ships this once, as a first-class Architecture Building Block: **`BaseValidationLoopAgent`**.

---

## The problem with one-shot agents

A standard `BaseAgent.execute()` is synchronous and single-pass: payload in, result out. That works for deterministic tasks — extract a field, classify a document, format a response.

It breaks down the moment the answer is uncertain and must be earned through iteration.

Consider a claims adjudication agent. The first evidence pass might return 60% confidence — not enough to approve, not enough to deny. The agent needs to re-query with a refined hypothesis, observe what comes back, and decide again. A one-shot agent has nowhere to put that reasoning loop. Every team reinvents it. Most do it badly — ad-hoc while loops, uncapped retries, no structured step history, no clean escalation path.

`BaseValidationLoopAgent` fixes this once, at the framework level.

---

## The loop

![K9-AIF Validation Loop Pattern](../assets/images/blogs/k9x-framework-validation-loop-pattern.png)

Every iteration of the loop runs the same four steps:

1. **Generate hypothesis** — form the next thing to test, using prior steps and the original payload
2. **Run validation** — invoke the tool, function, rule engine, or LLM that tests it
3. **Evaluate observation** — interpret the raw result into a structured observation with a confidence score
4. **Decide** — return one of four dispositions

The four dispositions are:

| Disposition | Meaning |
|---|---|
| `CONTINUE` | Confidence not yet sufficient — run another iteration |
| `FINALIZE` | Confidence sufficient — produce the validated output |
| `ESCALATE` | Unresolvable uncertainty — route to human-in-the-loop |
| `FAIL` | Definitive negative result |

The loop runs until a terminal disposition is reached or `max_iterations` is hit. If the cap is hit, the agent either finalizes with best-effort output or escalates — configurable per deployment.

---

## How it is built in K9-AIF

The ABB lives in `k9_aif_abb/k9_agents/validation/`. Five abstract methods define the contract. The loop skeleton is fixed.

```python
from k9_aif_abb.k9_agents.validation import (
    BaseValidationLoopAgent,
    ValidationDisposition,
    ValidationLoopContext,
    ValidationLoopResult,
)


class ClaimsEvidenceAgent(BaseValidationLoopAgent):

    layer = "ClaimsEvidenceAgent SBB"

    def generate_hypothesis(self, loop_ctx: ValidationLoopContext):
        # Use prior steps + payload to form the next evidence query
        prior = loop_ctx.steps[-1].observation if loop_ctx.steps else {}
        return {
            "query":    "policy_coverage",
            "claim_id": loop_ctx.payload["claim_id"],
            "focus":    prior.get("gaps", []),
        }

    def run_validation(self, hypothesis, loop_ctx: ValidationLoopContext):
        # Call your rule engine, database, or LLM here
        return policy_engine.check(hypothesis)

    def evaluate_observation(self, tool_result, loop_ctx: ValidationLoopContext):
        return {
            "covered":    tool_result.get("covered"),
            "gaps":       tool_result.get("gaps", []),
            "confidence": tool_result.get("match_score", 0.0),
        }

    def should_continue(self, observation, loop_ctx: ValidationLoopContext):
        threshold = self.config.get("confidence_threshold", 0.8)
        if observation["confidence"] >= threshold:
            return ValidationDisposition.FINALIZE
        if loop_ctx.iteration >= 3 and observation["confidence"] < 0.4:
            return ValidationDisposition.ESCALATE
        return ValidationDisposition.CONTINUE

    def finalize(self, loop_ctx: ValidationLoopContext) -> ValidationLoopResult:
        last = loop_ctx.steps[-1]
        return ValidationLoopResult(
            disposition      = ValidationDisposition.FINALIZE,
            output           = {"decision": "approved", "confidence": last.confidence},
            steps            = loop_ctx.steps,
            iterations       = loop_ctx.iteration,
            final_confidence = last.confidence,
            evidence         = [str(s.observation) for s in loop_ctx.steps],
        )
```

All domain logic is in those five methods. The loop, the step history, the telemetry, the error handling, the cap enforcement — all in the ABB.

---

## What the framework handles so you don't have to

**Structured step history.** Every iteration is recorded as a `ValidationLoopStep` — hypothesis, observation, disposition, confidence. The full history is available inside the loop (to inform the next hypothesis) and in the final output (for audit trails, Neo4j lineage, human review).

**Telemetry hooks.** The ABB emits structured events at every stage: `loop_started`, `hypothesis_generated`, `validation_tool_invoked`, `observation_evaluated`, `loop_continued`, `loop_finalized`, `loop_escalated`, `loop_failed`. These flow through `publish_event()` — wired to the monitor and message bus if configured.

**Tool error handling.** If `run_validation()` raises, the loop does not crash. The exception is caught, recorded as a step with confidence 0.0, and the agent returns `FAIL` — or `ESCALATE` if `escalate_on_tool_error: true` is set in config. No uncaught exceptions reach the Squad.

**Safe config parsing.** `finalize_on_max_iterations: false` is correctly read as `False`, not `True`. Python's `bool("false") == True` is a known trap — the ABB uses `_parse_bool()` to handle it.

**Confidence clamping.** Subclasses that return confidence values outside `[0.0, 1.0]` are silently clamped. Bad data from a tool never corrupts the confidence semantics.

**Sensitive data exclusion.** Raw tool results are never included in the step output dict. The `run_validation()` return value is available inside the loop (passed to `evaluate_observation()`) but stripped before the result leaves the agent. Tool responses often contain sensitive data — the ABB protects against accidental exposure.

---

## It is still just an agent

From the outside, `BaseValidationLoopAgent` is indistinguishable from any other `BaseAgent`. The Squad calls `execute(payload)` and receives a `dict`. It has no idea the agent iterated three times internally.

```yaml
flow:
  - agent: ClaimsTriageAgent       # one-shot
    result_key: triage
  - agent: ClaimsEvidenceAgent     # may iterate 1–5 times internally
    result_key: evidence
  - agent: AdjudicationAgent       # one-shot, resumes when evidence is done
    result_key: adjudication
```

The loop is an implementation detail. The squad contract is unchanged. The orchestrator contract is unchanged. The caller does not know and does not need to know.

This is the point of architecture building blocks — the complexity lives in the ABB, not scattered across every solution that needs iterative reasoning.

---

## Config

```yaml
max_iterations:             5      # hard cap — loop never runs forever
confidence_threshold:       0.8    # available to should_continue() via self.config
finalize_on_max_iterations: true   # true → finalize with best effort; false → escalate
escalate_on_tool_error:     false  # false → FAIL on tool exception; true → ESCALATE
```

All keys are optional. Defaults are reasonable for most domains. Override per agent deployment without touching code.

---

## K9ValidationLoopAgent — the OOB implementation

Just as `K9ModelRouter` is the ready-to-run OOB implementation of `BaseModelRouter`, **`K9ValidationLoopAgent`** is the OOB implementation of `BaseValidationLoopAgent`.

The LLM is the validation tool. Each iteration asks the LLM to assess the payload, return a confidence score, and signal whether another iteration would help. The loop continues until confidence reaches the threshold, the cap is hit, or the LLM says it can't improve further.

The `class:` field in agent YAML is a **config index key** — `AgentLoader` uses it to look up which YAML config belongs to which Python class. It must match the registered Python class name exactly.

Always create a named SBB subclass — the `class:` field, the Python class name, and the registry key all match:

```yaml
# agents/yaml/risk_assessment_agent.yaml
name: RiskAssessmentAgent
class: RiskAssessmentAgent       # ← matches the SBB Python class name

role: >
  You are a risk assessment specialist. Evaluate the input for financial and
  operational risk signals.

goal: >
  Assess overall risk level with a confidence score. Return JSON with
  conclusion, confidence, reasoning, and needs_more.

model: reasoning
max_iterations: 4
confidence_threshold: 0.85
finalize_on_max_iterations: true
```

```python
# agents/src/risk_assessment_agent.py
from k9_aif_abb.k9_agents.validation import K9ValidationLoopAgent

class RiskAssessmentAgent(K9ValidationLoopAgent):
    layer = "RiskAssessmentAgent SBB"
    # override only what differs from OOB
```

```python
# In _load_squad()
from agents.src.risk_assessment_agent import RiskAssessmentAgent

agent_registry.register(
    "RiskAssessmentAgent",
    lambda: RiskAssessmentAgent(config=loader.merge_with_global("RiskAssessmentAgent", self.config)),
)
```

### When to extend instead

If your validation tool is not the LLM — a rule engine, a database query, a sandbox — extend `K9ValidationLoopAgent` and override only `run_validation()`:

```python
from k9_aif_abb.k9_agents.validation import K9ValidationLoopAgent


class FraudValidationAgent(K9ValidationLoopAgent):
    """
    Extends K9ValidationLoopAgent — swaps in a rule engine as the validation
    tool.  Everything else (hypothesis generation, observation parsing,
    continuation logic, finalize) is inherited OOB.
    """

    layer = "FraudValidationAgent SBB"

    def run_validation(self, hypothesis, loop_ctx):
        # Replace LLM call with domain rule engine
        return fraud_rule_engine.evaluate(loop_ctx.payload)
```

One override. Everything else — step history, confidence clamping, telemetry, error handling, escalation — is inherited.

If your continuation logic also differs, override `should_continue()` too:

```python
    def should_continue(self, observation, loop_ctx):
        if observation["confidence"] >= 0.9:
            return ValidationDisposition.FINALIZE
        if observation["confidence"] < 0.2:
            return ValidationDisposition.FAIL   # ruled out definitively
        return ValidationDisposition.CONTINUE
```

Two overrides. Still no loop code, no step management, no telemetry wiring.

The hierarchy is:

```
BaseValidationLoopAgent   ← loop skeleton, error handling, telemetry (ABB)
  └── K9ValidationLoopAgent  ← LLM-driven OOB implementation
        └── FraudValidationAgent  ← domain SBB, overrides only what differs
```

`K9ValidationLoopAgent` ships with 16 tests covering OOB loop behaviour, JSON parsing, multi-iteration convergence, max-iteration caps, and subclass extension — all fully offline, no LLM or network required. See `k9_aif_abb/tests/test_k9_validation_loop_agent.py`.

---

## Applicable domains

The pattern is domain-agnostic. The same ABB that drives `ClaimsEvidenceAgent` also drives:

- **Security** — commit scan → exploit attempt → confirm/deny
- **Fraud** — signal correlation → rule check → risk confirmation
- **Compliance** — regulation lookup → clause match → gap assessment
- **Document extraction** — parse attempt → schema validation → confidence check
- **Diagnostics** — symptom query → test → differential narrowing

Different domains, different tools in `run_validation()`, same loop structure.

---

## State contracts in `models/`

The data classes (`ValidationLoopContext`, `ValidationLoopStep`, `ValidationLoopResult`, `ValidationDisposition`) live in `k9_agents/validation/models/` — separate from the execution logic. No loop code is in `models/`. No model code is in the agent.

This separation is intentional. Loop state can be persisted to PostgreSQL, written to the Neo4j knowledge graph, fed into telemetry dashboards, or serialized for human review — without touching the agent implementation. The contracts are stable; the execution logic can evolve independently.

---

## The Solutions Architect decision — and why it matters

The generator, intake, and Claude Code all scaffold agents extending `BaseAgent` by default. One-shot. That is the right default for most agents — triage, routing, guard, audit, graph sync. These produce their answer in a single pass and should stay one-shot.

The SA must make an explicit design-time decision for each agent:

> *"Does this agent need to test something, observe the result, and decide whether to try again — or does it produce its answer in one pass?"*

| One-pass → `BaseAgent` | Iterative convergence → `K9ValidationLoopAgent` |
|---|---|
| Triage, routing, audit, guard, graph sync | Fraud signal correlation, claims evidence, compliance gap, document confidence |

This is not an automatic upgrade. The generator cannot make this decision — only the SA can, because it requires domain knowledge about whether the problem requires convergence.

When the decision is made, the change is surgical:

```python
# Before — generated default
class FraudDetectionAgent(BaseAgent):
    def execute(self, payload): ...

# After — SA changes to iterative
class FraudDetectionAgent(K9ValidationLoopAgent):
    def generate_hypothesis(self, loop_ctx): ...
    def run_validation(self, hypothesis, loop_ctx): ...
    def evaluate_observation(self, tool_result, loop_ctx): ...
    def should_continue(self, observation, loop_ctx): ...
    def finalize(self, loop_ctx): ...
```

The squad YAML, the orchestrator, the agent YAML `class:` field — none of these change. The loop is internal to the agent.

**EOC reference agents identified for this migration:**
- `FraudDetectionAgent` — currently does rule signals + one LLM pass. Fraud correlation is the canonical iterative use case — multiple passes produce meaningfully better signal confidence.
- `DocumentExtractorAgent` — currently one-shot extract. Extraction confidence check + re-extraction on parse failure is a natural validation loop.

---

## What this means for enterprise AI

Most enterprise AI projects hit a wall when the problem stops being one-shot. A document classifier is straightforward. A claims adjudication engine that must build evidence, handle uncertainty, escalate to humans, and maintain an audit trail is not. The gap between those two is where most frameworks leave you on your own.

`BaseValidationLoopAgent` is K9-AIF's answer to that gap. The loop skeleton is solved once. Every solution team that needs iterative reasoning inherits it — and only writes the five domain methods that are actually their problem to solve.

The pattern is domain-agnostic. That is the point.

---

*`BaseValidationLoopAgent` and `K9ValidationLoopAgent` are available in `k9_aif_abb/k9_agents/validation/`. The full usage guide — including the SA decision table — is in [Skill 10 of SKILLS.md](https://github.com/k9aif/k9-aif-framework/blob/main/SKILLS.md). The framework is open source at [github.com/k9aif/k9-aif-framework](https://github.com/k9aif/k9-aif-framework).*
