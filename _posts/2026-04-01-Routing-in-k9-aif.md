---
layout: post
title: "Routing in K9-AIF: Deterministic and Non-Deterministic Paths"
---

**Date:** 2026-04-01 — *Updated 2026-05-31 with OOB implementation in v1.2.x*  
**Author:** Ravi Natarajan

## Why Routing Matters

In enterprise AI systems, not every request should be handled the same way.

Some requests are already structured and predictable. Others are ambiguous and require interpretation before the system can decide what to do next.

This is where routing becomes important.

In many discussions around agentic AI, routing is often assumed to be an intelligent, AI-driven step. In practice, however, routing in enterprise systems is often much simpler and much earlier than that.

A user selecting a known option from a screen, a workflow step, or a predefined business path is already a routing decision.

K9-AIF treats routing as an **architectural concern**, not merely as a prompt-classification problem.

> **Routing should be as deterministic as possible, and only as intelligent as necessary.**

---

## Routing Already Exists in Enterprise Systems

Routing is not new.

Most enterprise applications already route users and requests every day through:

- menus
- forms
- workflow steps
- navigation paths
- application rules
- backend dispatch logic

For example, when a user selects one of the following on a portal:

- Claims
- Billing
- Policy Inquiry
- New Policy

…the system has already gained a strong signal about where the request belongs.

In other words, routing often begins **before AI is ever invoked**.

This is an important design principle, because it means not every route should require an LLM, intent classifier, or semantic router.

---

## What K9-AIF Adds

K9-AIF does not invent routing.

What it adds is an **architectural structure** for routing inside enterprise AI systems.

Without a framework, routing logic often gets scattered across:

- UI code
- controllers
- service layers
- orchestrator glue code
- hardcoded conditional branches

Over time, this becomes difficult to reason about, test, govern, and extend.

K9-AIF helps by making routing a **first-class architectural concern**.

This provides several benefits:

- routing logic becomes more explicit
- deterministic and non-deterministic routing can coexist
- routing remains separated from orchestration and agent execution
- new routes and handlers can be added more cleanly
- routing decisions become easier to inspect, test, and evolve

This is where K9-AIF helps: not by making routing “magical,” but by making it **structured**.

---

## Routing in K9-AIF

In K9-AIF, the **Router** is a first-class architectural component.

Its responsibility is to evaluate an incoming request and determine the appropriate processing path.

However, that does **not** mean every route must be AI-driven.

The router can support multiple routing strategies, including:

- deterministic routing
- rule-based routing
- non-deterministic routing
- fallback routing

This is important because different use cases require different levels of interpretation.

---

## Deterministic Routing

Deterministic routing is used when the correct route is already known or strongly constrained.

This usually happens when:

- the user has selected a structured option
- the business flow is predefined
- the application already knows the domain
- the path is governed by business rules

Examples include:

- Claims → Claims Orchestrator
- Billing → Billing Orchestrator
- Policy Inquiry → Policy Service Orchestrator
- New Policy → Policy Sales Orchestrator

In these cases, introducing AI-based routing may actually make the system:

- slower
- more expensive
- less predictable
- harder to test

Deterministic routing is often the better architectural choice when the path is already clear.

---

## Non-Deterministic Routing

Non-deterministic routing is useful when the path is **not yet known**.

This usually applies when:

- the user enters a free-form request
- the intent is ambiguous
- multiple orchestrators may be relevant
- interpretation is required before dispatch

Examples include requests such as:

- “I had an accident yesterday and need help”
- “I got a document in the mail and I’m not sure what to do”
- “I need help with something related to my policy”
- “My issue doesn’t fit any of the options above”

In these situations, the router may need to apply:

- rule-based evaluation
- semantic matching
- intent classification
- retrieval-assisted routing
- optional LLM-assisted interpretation

This is where a non-deterministic path becomes useful.

---

## K9-AIF Routing Model

The following high-level view shows how routing can work inside K9-AIF.

![K9-AIF Routing Model](/assets/images/blogs/k9-aif-routing-model.png)

---

## ACME Insurance Routing Example

The following example shows how deterministic and non-deterministic routing can coexist within the same application.

![ACME Insurance Routing Example](/assets/images/blogs/acme-insurance-routing.png)

---

---

## What This Looks Like in K9-AIF 1.2.x

The concepts above are now fully implemented as OOB components in `k9-aif 1.2.x`.

The **Router** is the single entry point for all events — always. It never sits behind a pre-classification step.

```
Event → K9EventRouter (single entry point)
    ├── event_type in routing table ──────────────────► domain topic
    └── event_type unknown ──────────► intent.in
                                            │
                              IntentOrchestrator (consumes intent.in)
                                  → IntentSquad → K9IntentAgent
                                      ├── intent resolved ──► domain topic
                                      └── intent unclear  ──► responses.out
```

There are exactly **three routing outcomes**:

1. **Deterministic** — `event_type` is in the routing table. The Router publishes directly to the domain topic. No LLM, no latency.
2. **Non-deterministic, resolved** — `event_type` is unknown. The Router publishes to `intent.in`. The `IntentOrchestrator` picks it up, runs intent classification, and re-publishes to the correct domain topic.
3. **Clarification required** — intent classification cannot determine the path with sufficient confidence. The `IntentOrchestrator` publishes a "please clarify" response. Nothing is silently dropped.

The `IntentOrchestrator` is a **Kafka-decoupled process** — it consumes `intent.in` independently of the Router. The Router does not know it exists. This is the same topology used throughout K9-AIF: Router publishes, Orchestrators consume.

---

## The IntentOrchestrator is OOB

The `IntentOrchestrator` self-bootstraps with `K9IntentAgent` and `IntentSquad` out of the box.

`K9IntentAgent` follows a three-step classification order:

1. **`intent_map` rule lookup** — zero latency, no LLM, configured in YAML
2. **LLM via `llm_invoke`** — for truly ambiguous free-text input
3. **Fallback** — `event_type` verbatim or `"unknown"`

Configure the routing table and intent map in `config.yaml`:

```yaml
routing:
  intent_topic:         intent.in
  response_topic:       responses.out
  confidence_threshold: 0.6
  table:
    fraud:    fraud.in
    claims:   claims.in
    document: documents.in
  intent_map:
    fraud_report: fraud
    claim_form:   claims
```

No Python code required for the common case.

---

## SBB Extension Points

Solutions override exactly what differs — nothing more.

**Replace the intent agent** (e.g. keyword matching, Drools rules, NLP pipeline):

```python
class ConfigListIntentAgent(BaseIntentAgent):
    def classify(self, payload):
        text = payload.get("message", "").lower()
        for intent, keywords in self._keywords.items():
            if any(kw in text for kw in keywords):
                return intent
        return ""
```

**Add domain logic around classification** (pre-processing, audit trail, custom clarification message):

```python
class AcmeIntentOrchestrator(IntentOrchestrator):
    def execute_flow(self, payload):
        payload = self._enrich(payload)         # domain pre-processing
        result  = super().execute_flow(payload) # OOB classification + routing
        self._audit(result)                     # domain audit trail
        return result

    def _clarification_message(self, intent, confidence, payload):
        return "Please choose: Report a Claim, Report Fraud, or Upload a Document."
```

The topology — Router → `intent.in` → IntentOrchestrator → domain topic — does not change regardless of which strategy is used inside the agent.

---

## Available Now

```bash
pip install k9-aif==1.2.1
```

The working example with all three routing outcomes and both SBB override patterns is in `examples/k9routing/` in the repository. It runs without Kafka or a live LLM.

---

## Learn More

K9-AIF is an architecture-first framework for modular and governed
agentic AI systems.

More at:

<a href="https://k9aif.com" target="_blank" rel="noopener noreferrer">
  Visit K9-AIF
</a>

