---
layout: post
title: "Routing Workflows: The Pattern Anthropic Named, Already Governed in K9-AIF"
date: 2026-08-08
author: Ravi Natarajan
---

I'd been going through Anthropic's own workflow patterns material again, the same [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) writeup behind an [earlier post here](https://blog.k9x.ai/claude-agent-sdk-governance-boundary/) on the Claude Agent SDK: Chaining, Routing, Parallelization, Orchestrator-workers, Evaluator-optimizer, mostly to see how Anthropic itself draws the boundaries between these shapes. The "Routing Workflows" diagram stopped me. Its description: use an initial call to categorize the user's query or task, forward it to a dedicated pipeline for handling that category, and the chosen path can be a workflow, a prompt, a set of tools, whatever the category needs. User input only ever goes to one path.

I'd seen that shape before. Not in a diagram, in K9-AIF, a framework I've built for running AI systems where every decision, not just every model call, stays traceable and governed. One of its core pieces is a Router: the one place every incoming request has to pass through before anything else happens.

The question I actually wanted answered wasn't "does K9-AIF do routing." It already does, that piece is built and tested. The real question was narrower: if I implemented Anthropic's Routing pattern exactly as described, using K9-AIF, what would I get that the pattern description itself doesn't specify? Where does the framework add something on top of the shape, rather than just relabeling it?

> **Claude says:** Routing workflows solve a common problem in AI applications: different types of user requests need different handling approaches. Instead of using a one-size-fits-all prompt, you can categorize incoming requests and route them to specialized processing pipelines.
>
> **K9-AIF Framework provides this:** a Router that every request passes through first. It makes the same categorize-then-route decision, except categorizing isn't a blanket "ask an LLM" step by default, a simple table gets checked first, an LLM only gets involved when the request type is genuinely unknown, and if even that isn't confident enough, the system asks for clarification instead of guessing. All of it tracked and auditable, the same as everything else in the framework.

---

## Anthropic's Routing pattern, as described

Three steps, no more:

1. An initial call categorizes the incoming query or task.
2. The query is forwarded to a dedicated pipeline for that category.
3. That pipeline can be a workflow, a prompt, a set of tools, or anything else suited to the category.

It's a clean, useful shape. It's also, deliberately, a pattern description, not an implementation. It doesn't say what "an initial call" costs, what happens when categorization is uncertain, or how a routing decision gets audited. That's not a gap in Anthropic's writeup, it's not what a pattern description is for.

## K9-AIF's Router, the same shape with three concrete outcomes

K9-AIF's Router is the single entry point for every request, always. It never sits behind some other classification step of its own. From there, exactly three things can happen:

1. **Deterministic.** The request type is already known, listed in a simple config table, not buried in code. Straight to the right place. No LLM involved, no delay.
2. **Resolved by asking an LLM, only when needed.** The request type genuinely isn't recognized. Before anything expensive happens, a cheap rule-based check runs first, essentially a lookup list. Only if that also comes up empty does the system actually ask an LLM to classify the request.
3. **Clarification required.** Even the LLM's answer isn't confident enough to act on. Instead of guessing and risking a wrong answer, the system asks the user to clarify. Nothing gets silently dropped, and nothing gets forced through on a bad guess.

Here's the flow, for anyone who wants to see the actual shape of it:

```
Request → Router (single entry point)
    ├── request type already known ─────────────────► handled directly
    └── request type unknown ────────► classification step
                                            │
                              (a separate process picks this up)
                                  → cheap rule check first, then LLM if needed
                                      ├── resolved   ──► handled
                                      └── still unclear ──► ask for clarification
```

(For anyone curious about the exact code: this is `K9EventRouter` publishing to a queue, picked up independently by `IntentOrchestrator`, which runs `K9IntentAgent`. None of those names matter for the argument here, they're just proof this is real, tested code, not a simplified retelling for the blog.)

K9-AIF also ships a separate piece called the Intelligent Model Router, for choosing which LLM answers a call once something's already running. That's a different part of the framework and not what this post is about.

That's not a reinterpretation of Anthropic's Routing pattern. It's the same shape — Anthropic's "initial call" is the categorization step, Anthropic's "dedicated pipeline" is wherever the request ends up. What's different is what got added to make it something you can actually run in production.

## Where K9-AIF adds to the pattern

**Deterministic first, enforced by the architecture, not left as a choice.** Worth being precise here: Anthropic's own writeup is neutral on this — it says classification "can be handled accurately, either by an LLM or a more traditional classification model/algorithm," and leaves picking between them to whoever builds the system. K9-AIF doesn't leave it open. The cheap check always runs first, the LLM only gets called as a last resort, in that order, every time, because the framework enforces the order rather than trusting each team to choose well on their own. This is the same argument I made in [Not Every Agent Needs an LLM](https://blog.k9x.ai/not-every-agent-needs-an-llm/): most routing decisions in a real system are already knowable, and every unnecessary categorization call is compute spent proving something that was never actually in question.

**Governed and auditable, not just described.** Anthropic's pattern is silent on what happens to a routing decision afterward, and it should be, that's outside a pattern description's job. K9-AIF's Router isn't silent about it: every routing decision runs through the [same governance chain as everything else in the framework](https://blog.k9x.ai/how-k9-aif-enforces-governance/), inspectable rather than assumed.

**Decoupled, not a direct call.** The pattern's diagram draws a straight line from the categorization step to each destination. The real implementation isn't a direct call at all — the Router hands off the request and moves on; a completely separate process picks it up on its own time. If that separate process goes down, the Router keeps running, but those unresolved requests just wait until it's back. Nothing crashes. Nothing gets decided either, until it's back. That's the honest tradeoff, not hidden, not free.

**A third outcome, not two.** The pattern's implicit choices are categorize, then forward. K9-AIF adds a real third path: confidence too low to act on becomes an explicit request for clarification, not a forced guess and not a silent failure.

**Config-driven, not a new code path per category.** Adding a new request category is a configuration change, not new code. As proof this isn't hand-waved, here's the real configuration from a working example:

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

## Complementary, not competitive

Anthropic defined a pattern that's genuinely useful at the level it's aimed at: anyone structuring calls to Claude directly benefits from thinking in terms of Chaining, Routing, Parallelization, Orchestrator-workers, Evaluator-optimizer. That's not a claim K9-AIF has any reason to argue with.

What K9-AIF adds is what happens when that same pattern has to survive contact with a production enterprise system: governance that doesn't depend on remembering to add it, an audit trail that isn't optional, a system that doesn't fall over when one downstream piece is slow, and a third outcome for the case the two-outcome version of the pattern doesn't name. Someone building directly against Claude gets the pattern. Someone building on K9-AIF gets it already hardened.

---

Full routing mechanics, config reference, and the extension points for replacing the intent step or wrapping the orchestrator: [Routing in K9-AIF: Deterministic and Non-Deterministic Paths](https://blog.k9x.ai/routing-in-k9-aif/). Anthropic's own pattern writeup: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents).

```bash
pip install k9-aif==1.10.0
```

Working example with all three routing outcomes, both override patterns, runs without Kafka or a live LLM: `examples/k9routing/` in the repository.

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
