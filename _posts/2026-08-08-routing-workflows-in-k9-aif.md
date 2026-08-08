---
layout: post
title: "Routing Workflows: The Pattern Anthropic Named, Already Governed in K9-AIF"
date: 2026-08-08
author: Ravi Natarajan
---

I'd been going through Anthropic's own workflow patterns material again, the same [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) writeup behind an [earlier post here](https://blog.k9x.ai/claude-agent-sdk-governance-boundary/) on the Claude Agent SDK: Chaining, Routing, Parallelization, Orchestrator-workers, Evaluator-optimizer, mostly to see how Anthropic itself draws the boundaries between these shapes. The "Routing Workflows" diagram stopped me. Its description: use an initial call to categorize the user's query or task, forward it to a dedicated pipeline for handling that category, and the chosen path can be a workflow, a prompt, a set of tools, whatever the category needs. User input only ever goes to one path.

I'd seen that shape before. Not in a diagram, in `k9_aif_abb/k9_core/router/`.

The question I actually wanted answered wasn't "does K9-AIF do routing." I already knew it did, there's a whole OOB `K9EventRouter` and `IntentOrchestrator` pair for exactly this. The real question was narrower: if I implemented Anthropic's Routing pattern exactly as described, using K9-AIF, what would I get that the pattern itself doesn't specify? Where does the framework add something on top of the shape, rather than just relabeling it?

> **Claude says:** Routing workflows solve a common problem in AI applications: different types of user requests need different handling approaches. Instead of using a one-size-fits-all prompt, you can categorize incoming requests and route them to specialized processing pipelines.
>
> **K9-AIF Framework provides this:** `K9EventRouter`, the single entry point for every request in a K9-AIF solution. It makes the same categorize-then-route decision, except categorization isn't a blanket LLM call by default, a deterministic routing table is checked first, `IntentOrchestrator`'s LLM-backed classification only runs when the request type is genuinely unknown, and a structured clarification response replaces a forced wrong guess when confidence isn't there. All of it under the same governance and audit trail as everything else in the framework.

---

## Anthropic's Routing pattern, as described

Three steps, no more:

1. An initial call categorizes the incoming query or task.
2. The query is forwarded to a dedicated pipeline for that category.
3. That pipeline can be a workflow, a prompt, a set of tools, or anything else suited to the category.

It's a clean, useful shape. It's also, deliberately, a pattern description, not an implementation. It doesn't say what "an initial call" costs, what happens when categorization is uncertain, or how a routing decision gets audited. That's not a gap in Anthropic's writeup, it's not what a pattern description is for.

## K9-AIF's Router, the same shape with three concrete outcomes

`K9EventRouter` is the single entry point for every event in a K9-AIF solution, always. It never sits behind a pre-classification step of its own. From there, exactly three outcomes:

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

1. **Deterministic.** `event_type` is already in the routing table (configured in YAML, not code). Straight to the domain topic. No LLM, no categorization latency at all.
2. **Non-deterministic, resolved.** `event_type` is unknown. The Router publishes to `intent.in`; the `IntentOrchestrator` picks it up independently and runs a [Squad](https://blog.k9x.ai/agent-squads-in-k9-aif/), `IntentSquad`, wrapping `K9IntentAgent`, which itself checks a rule-based `intent_map` before ever reaching for an LLM.
3. **Clarification required.** Confidence comes back below threshold. The `IntentOrchestrator` publishes a structured "please clarify" response. Nothing gets silently dropped, and nothing gets a wrong guess forced through.

*(Worth a quick disambiguation: this is the **Event Router**, `K9EventRouter`, deciding which orchestrator handles a request. It's a different component from the [Model Router](https://blog.k9x.ai/k9-model-router-in-k9-aif/), which decides which LLM handles a given inference call once an agent is already running. Same word, two different jobs, both deterministic-first for the same underlying reason.)*

That's not a reinterpretation of Anthropic's Routing pattern. It's the same shape, Anthropic's "initial call" is the categorization step, Anthropic's "dedicated pipeline" is the domain topic and its orchestrator. What's different is what got added to make it something you can run in production.

## Where K9-AIF adds to the pattern

**Deterministic first, by construction, not convention.** Anthropic's diagram implies a categorization call happens up front, every time. K9-AIF's Router refuses that default: check the table first, only reach for an LLM when the event type genuinely isn't known. `K9IntentAgent` repeats the same discipline one layer deeper, an `intent_map` YAML lookup before it ever calls `llm_invoke`. This is the same argument I made in [Not Every Agent Needs an LLM](https://blog.k9x.ai/not-every-agent-needs-an-llm/): most routing decisions in a real system are already knowable, and every unnecessary categorization call is compute spent proving something that was never actually in question.

**Governed and auditable, not just described.** Anthropic's pattern is silent on what happens to a routing decision afterward, and it should be, that's outside a pattern description's job. K9-AIF's Router isn't silent about it: every routing decision runs through the [same governance chain as everything else in the framework](https://blog.k9x.ai/how-k9-aif-enforces-governance/), inspectable rather than assumed.

**Decoupled topology, not a direct call.** The diagram draws a straight line from the categorization step to each destination pipeline. The real implementation is asynchronous: the Router publishes to `intent.in` and moves on, the `IntentOrchestrator` picks it up as a separate, Kafka-decoupled process the Router doesn't even know exists. If the classifier goes down, the Router keeps running, but `intent.in` just backs up until it recovers. Nothing crashes. Nothing gets decided either, until it's back. That's the honest tradeoff, not hidden, not free.

**A third outcome, not two.** The pattern's implicit choices are categorize, then forward. K9-AIF adds a real third path: confidence too low to act on becomes an explicit clarification response, not a forced guess and not a silent failure.

**Config-driven, not a new code path per category.** The routing table and the intent map are both YAML, this is the actual `config.yaml` from the working example, not a paraphrase:

```yaml
routing:
  intent_topic:         intent.in
  response_topic:       responses.out
  confidence_threshold: 0.6

  table:
    claims_submitted: claims.in
    fraud_alert:      fraud.in
    doc_uploaded:     documents.in

  intent_map:
    claims_submitted: claims
    fraud_report:     fraud
    doc_uploaded:     document
```

Adding a category is a config change. No Python required for the common case.

## Complementary, not competitive

Anthropic defined a pattern that's genuinely useful at the level it's aimed at: anyone structuring calls to Claude directly benefits from thinking in terms of Chaining, Routing, Parallelization, Orchestrator-workers, Evaluator-optimizer. That's not a claim K9-AIF has any reason to argue with.

What K9-AIF adds is what happens when that same pattern has to survive contact with a production enterprise system: governance that doesn't depend on remembering to add it, an audit trail that isn't optional, a topology that doesn't fall over when one downstream service is slow, and a third outcome for the case the two-outcome version of the pattern doesn't name. Someone building directly against Claude gets the pattern. Someone building on K9-AIF gets it already hardened.

---

## Author's note: how would you actually use this framework for Claude?

Two different questions here, worth keeping apart. Can the Claude Agent SDK's own reasoning be routed through `K9ModelRouter`? No, [that boundary is explicit](https://blog.k9x.ai/claude-agent-sdk-governance-boundary/): Anthropic's SDK gives no swap-in point for its own model call, verified against the SDK's own source, not just its docs. Can `K9IntentAgent`'s classification call, the one LLM touchpoint in this post's whole routing flow, be routed to Claude through `K9ModelRouter`? Not yet, but for a different reason. `K9IntentAgent` never touches the SDK at all, it's a native K9-AIF agent calling `llm_invoke()` → `K9ModelRouter` → `LLMFactory` like any other model call. Today, `LLMFactory` just has nothing to point at Claude with. That's not a boundary like the SDK's, it's an adapter that hasn't been built yet.

That's exactly the plain-API path the SDK post names as the answer for anyone who needs real inference governance with Claude, not the SDK adapter, a direct API call through K9-AIF's own model router. It's also exactly what `SKILLS.md` Skill 13 documents the recipe for: a `ClaudeLLM(BaseLLM)` adapter, registered in `LLMFactory`, credentials from `ANTHROPIC_API_KEY`. I checked before writing this: it doesn't exist in the framework yet. Only `ollama_llm.py`, `openai_llm.py`, `watsonx_llm.py`, and `mock_llm.py` do. So the honest answer isn't "just flip a config flag", it's that this is a contained, well-scoped addition following a pattern already proven three times over, not a change to anything this post describes.

---

Full routing mechanics, config reference, and the SBB extension points for replacing the intent agent or wrapping the orchestrator: [Routing in K9-AIF: Deterministic and Non-Deterministic Paths](https://blog.k9x.ai/routing-in-k9-aif/). Anthropic's own pattern writeup: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents).

```bash
pip install k9-aif==1.10.0
```

Working example with all three routing outcomes, both SBB override patterns, runs without Kafka or a live LLM: `examples/k9routing/` in the repository.

---

## References

- K9-AIF Framework: [github.com/k9aif/k9-aif-framework](https://github.com/k9aif/k9-aif-framework)
- Anthropic, Building Effective Agents: [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)
- K9-AIF Patterns, Event Router Pattern: [patterns.k9x.ai/event-router-pattern.html](https://patterns.k9x.ai/event-router-pattern.html)
- K9-AIF Patterns, Model Router Pattern: [patterns.k9x.ai/model-router-pattern.html](https://patterns.k9x.ai/model-router-pattern.html)
- PyPI (k9-aif): [pypi.org/project/k9-aif](https://pypi.org/project/k9-aif/)

**Related posts on this blog:**

- [Routing in K9-AIF: Deterministic and Non-Deterministic Paths](https://blog.k9x.ai/routing-in-k9-aif/)
- [Not Every Agent Needs an LLM](https://blog.k9x.ai/not-every-agent-needs-an-llm/)
- [How K9-AIF Enforces Governance](https://blog.k9x.ai/how-k9-aif-enforces-governance/)
- [Agent Squads in K9-AIF](https://blog.k9x.ai/agent-squads-in-k9-aif/)
- [K9 Model Router in K9-AIF](https://blog.k9x.ai/k9-model-router-in-k9-aif/)
- [Claude Agent SDK in K9-AIF: Govern What It Does, Not How It Thinks](https://blog.k9x.ai/claude-agent-sdk-governance-boundary/)

---

<small><em>Wrote this blog, refined it using Claude Code in VS Code. It knows the framework and patterns well, thanks to the CLAUDE.md files at both the main and component levels giving it enough context to work with.</em></small>
