---
layout: post
title: "The Validation Loop Pattern — How K9-AIF Agents Reason Until They're Sure"
date: 2026-05-25
author: Ravi Natarajan
---

OpenAI recently published details on **Aardvark** — their autonomous security research agent that can discover, exploit, and verify real-world vulnerabilities without human intervention.

If you read past the security headlines, the architectural insight buried in that paper is more interesting than any individual vulnerability it found.

Aardvark does not guess once and report. It hypothesizes, tests, observes, updates its understanding, and decides whether to keep going or stop. The loop is the agent. The one-shot `execute()` call is not sufficient for problems that require convergence.

That pattern — **hypothesis → tool/test → observation → re-reason → continue or finalize** — is domain-agnostic. The same structure that drives a security agent discovering exploits also drives a claims adjudication agent building evidence, a fraud detection agent correlating signals, or a compliance agent narrowing down gaps.

K9-AIF now ships this as a first-class Architecture Building Block: **`BaseValidationLoopAgent`**.

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

## Applicable domains

The pattern is domain-agnostic. The same ABB that drives `ClaimsEvidenceAgent` also drives:

- **Security** — commit scan → exploit attempt → confirm/deny (Aardvark-style)
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

## What this means for enterprise AI

Most enterprise AI projects hit a wall when the problem stops being one-shot. A document classifier is straightforward. A claims adjudication engine that must build evidence, handle uncertainty, escalate to humans, and maintain an audit trail is not. The gap between those two is where most frameworks leave you on your own.

`BaseValidationLoopAgent` is K9-AIF's answer to that gap. The loop skeleton is solved once. Every solution team that needs iterative reasoning inherits it — and only writes the five domain methods that are actually their problem to solve.

The pattern Aardvark demonstrated for security is now a reusable ABB for any domain. That is the point.

---

*`BaseValidationLoopAgent` is available in `k9_aif_abb/k9_agents/validation/`. The full usage guide is in [Skill 10 of SKILLS.md](https://github.com/k9aif/k9-aif-framework/blob/main/SKILLS.md). The framework is open source at [github.com/k9aif/k9-aif-framework](https://github.com/k9aif/k9-aif-framework).*
