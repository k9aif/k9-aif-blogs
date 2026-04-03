---
layout: post
title: "K9-AIF Model Router vs. NotDiamond: An Architectural Comparison"
date: 2026-04-02
author: Ravi Natarajan
---

Model routing is becoming a first-class concern in enterprise AI systems. As teams move beyond single-model experiments into governed, multi-model architectures, the question of *how* to select the right model for each request matters more than ever.

Two approaches sit at different ends of the design spectrum: the **K9-AIF Model Router**, which treats routing as an architectural concern inside a governed framework, and **NotDiamond**, a predictive routing service that uses a trained meta-model to dynamically select the best LLM for each query.

This post compares both from an architectural and operational perspective.

---

## Core Philosophy

The K9-AIF Model Router is an **Architecture Building Block (ABB)** — a governed slot in the inference layer of a K9-AIF application. Routing decisions are deterministic by default, driven by configuration, policy rules, and enterprise constraints. The router is part of your architecture, not a dependency on it.

NotDiamond is a **predictive routing service** — a trained meta-model that dynamically selects the best LLM for each query based on quality, cost, or latency signals. Its routing intelligence is learned from data, not configured as rules. It operates primarily as a cloud service, with VPC deployment available for enterprise customers.

---

## Routing Strategies

| Strategy | K9-AIF Model Router | NotDiamond |
|---|---|---|
| Decision model | Deterministic — rule-based, config-driven via YAML | Dynamic — ML meta-model trained on evaluation data |
| Capability-based routing | Route by model class, size, task type, or provider priority | Automatically infers task complexity and routes accordingly |
| Sensitivity-aware routing | Enterprise allow/deny lists, compliance-driven provider filtering | Not natively supported |
| Cost / latency tradeoffs | Configurable as routing criteria in YAML | Built-in tradeoff modes: cost, latency, or quality |
| Fallback rules | Configurable provider fallback chains | Not explicit — relies on learned model preferences |
| Custom routing logic | Extend `BaseModelRouter` as an SBB — full control | Train a custom router on your own evaluation dataset |
| Environment awareness | Route differently per deployment environment | Not supported |

---

## Session and State Persistence

This is one of the clearest differentiators between the two approaches.

The K9-AIF Model Router supports persistence out of the box:

- **SQLite** by default — no configuration required
- **PostgreSQL** for enterprise deployments
- **In-memory** storage for lightweight or test scenarios

The persistence backend is selected via `config.yaml`. What gets persisted:

- Full sessions and user turns
- Routing decisions per request
- Model affinity across sessions

This allows the router to evolve from simple one-shot selection into a **session-aware routing component** — where future decisions can consider prior interactions and model affinity.

NotDiamond, by contrast, returns a session ID per routing call. That ID can be used to submit feedback and track decisions, but there is no server-side session persistence, model affinity tracking across sessions, or configurable backend. State management is left entirely to the caller.

---

## Extensibility

K9-AIF treats `BaseModelRouter` as an ABB contract. Any routing strategy — including NotDiamond — can be introduced as a **Solution Building Block (SBB)** by extending this class.

This means you can:

- Start with the default rule-based router with zero setup
- Swap in a **NotDiamond-backed SBB** for intelligent, ML-driven routing
- Build a hybrid that applies governance filters before delegating to NotDiamond
- Do all of this without changing the rest of your application architecture

| Extensibility point | **K9-AIF Model Router** | **NotDiamond** |
|---|---|---|
| Custom strategy | Extend `BaseModelRouter` in Python | Train a custom router on your evaluation data |
| Use NotDiamond inside K9-AIF | Yes — wrap as an SBB, plug into the ABB slot | N/A |
| OOB default router | Yes — ships with a working default router | Yes — pre-trained cross-domain router available immediately |
| Swap without app changes | Yes — router is isolated behind the ABB contract | No — requires code changes at call sites |

---

## Decision Guidance

To help teams choose the right routing strategy, here’s a scenario-based guide:

| Scenario | Best Choice | Why |
|---|---|---|
| Strict governance, compliance, audit | K9-AIF Router | Deterministic, policy-aware, auditable |
| Max quality across diverse tasks | NotDiamond | ML meta-model selects optimal model per query |
| Hybrid enterprise routing | Both | Governance + intelligence combined |
| Air-gapped or on-prem deployment | K9-AIF | Self-hosted, no external API required |

---

## Why NotDiamond Matters

NotDiamond solves a difficult problem: predicting the best model for each query dynamically. Key benefits:

- Reduces cost by avoiding overuse of large models
- Improves quality by learning from evaluation data
- Complements K9-AIF governance with ML-driven decision-making
- Can be integrated as a hybrid SBB inside K9-AIF

---

## Hybrid Routing Flow (Visualized)

For hybrid setups combining governance and dynamic intelligence:
### Example: Hybrid Router in `config.yaml`

K9-AIF supports hybrid routing — combining **rule-based governance** with **dynamic intelligence** from NotDiamond.ai:

```yaml
router:
  type: hybrid
  base: rule_based
  intelligence: notdiamond
  governance:
    allowed_providers: [openai, anthropic, groq]
    compliance: enterprise_policy
```

### Routing Flows (Visualized)

**NotDiamond Flow:**  
User Query → Meta-Model → Provider Recommendation → LLM Call

**K9-AIF Flow:**  
User Query → Governance Rules → Router → LLMFactory → Selected LLM

**Hybrid Flow:**  
User Query → Governance Rules → NotDiamond Intelligence → LLMFactory → Selected LLM

---

## Integration and Governance

| Concern | **K9-AIF Model Router** | **NotDiamond** |
|---|---|---|
| Governance integration | First-class — sits inside K9-AIF’s governance layer | None — standalone tool, no governance framework |
| Policy enforcement | Allow/deny lists and compliance rules enforced at routing time | Not supported |
| Data privacy | All data stays inside your environment | Query content sent to NotDiamond’s external API |
| Auditability | Routing decisions persisted and auditable | Session IDs available, but no audit trail in your system |
| Enterprise deployment | Self-hosted, air-gap compatible, PostgreSQL-backed | SaaS, with VPC deployment available for enterprise customers |
| Orchestration awareness | Aware of agent, squad, and orchestration context | Stateless per call — no orchestration context |
| LLMFactory integration | Routing decisions feed directly into LLMFactory | Returns a provider recommendation — caller handles instantiation |

---

## How They Relate

NotDiamond and the K9-AIF Model Router are not competing solutions. They solve different problems.

- **NotDiamond** is excellent at the routing decision itself — its ML-trained intelligence is difficult to replicate with hand-written rules, especially across diverse query types and model providers.  
- **K9-AIF Model Router** is excellent at **where routing fits** in a governed enterprise architecture — persisted, policy-aware, session-aware, and architecturally isolated behind a stable ABB contract.

The right enterprise design uses both: **K9-AIF provides the governed inference layer**, and a NotDiamond-backed SBB provides the intelligent routing strategy inside it.

---

## Summary

K9-AIF treats model routing as a first-class architectural concern — not an implementation detail. The default router works out of the box with SQLite persistence and rule-based configuration. For teams that need ML-driven routing intelligence, NotDiamond can be introduced as an SBB without changing anything else in the architecture.

That separation — between *where routing lives* and *how routing decisions are made* — is the key architectural insight.

---

### Diagram: K9 Model Router in K9-AIF

The following diagram shows how the Model Router fits within the K9-AIF inference layer and interacts with `LLMFactory` and providers:

![K9-AIF Model Router](/assets/images/blogs/k9-aif-model-router.png)

---

Want to see a NotDiamond-backed router inside K9-AIF in action? 
A demo showcasing the hybrid configuration and session-aware routing 
will be released soon — stay tuned!

---

*K9-AIF is an architecture-first framework for modular and governed agentic AI systems. Learn more at [k9aif.com](https://k9aif.com).*
    allowed_providers: [openai, anthropic, groq]
    compliance: enterprise_policy
