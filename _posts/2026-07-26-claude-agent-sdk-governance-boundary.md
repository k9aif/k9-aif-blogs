---
layout: post
title: "Claude Agent SDK in K9-AIF: Govern What It Does, Not How It Thinks"
date: 2026-07-26
author: Ravi Natarajan
---

I'd been reading Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) alongside the Claude Agent SDK docs, mostly just to understand how Anthropic itself thinks about agent design. Somewhere in there I noticed the obvious thing that's easy to skip past: the Agent SDK only works with Claude models. Not "Claude by default." Claude, full stop.

That's a reasonable design choice for Anthropic's own SDK. But it's exactly the kind of detail that matters if you're building on K9-AIF, where every other integration point (native agents, CrewAI crews) is provider-agnostic by construction. So the question I actually sat with was narrower than "should we use the Claude Agent SDK": it was, for an organization standardizing on Claude, migrating or modernizing onto it, how would K9-AIF blend this SDK in without quietly giving up what the framework is for?

That question turned out to have a real, verifiable answer, and a boundary I wasn't expecting to be this clean.

---

## What "blend it in" actually has to mean

K9-AIF's whole architecture rests on a small number of guarantees: every agent boundary is governed, every action is checked against policy before it executes, every routing decision is auditable, and every model call routes through `llm_invoke` so the same agent runs against Ollama, OpenAI, or Watsonx without touching its code. The framework already has an adapter proving that holds for an external agent framework, not just native agents.

I'd built that adapter myself, for CrewAI, so I already knew the pattern cold. The plan was straightforward: inherit the right framework base classes and build the equivalent for the Claude Agent SDK: `BaseOrchestrator`, `BaseAdapter`, a payload mapper, the whole shape.

But here, I would not be able to use the framework's Intelligent Model Router the same way. The parallel held for about half the adapter.

---

## The half that holds: action governance

The Claude Agent SDK gives you three real levers over what an agent is allowed to *do*, and they're stronger than I expected:

- **What tools it can see at all.** The adapter decides the full list up front, from its own internal registry. The SDK never gets handed a tool the adapter didn't put there itself.
- **What tool it can actually use, in the moment.** Before every single tool call runs, the SDK stops and asks the adapter for permission. The adapter checks that request against K9-AIF's own security layer, k9x_Shield, before answering yes or no.
- **What a spawned sub-agent is allowed to touch.** The Agent SDK can spin up its own helper agents mid-task. Each one only gets the tools the adapter explicitly hands it. Ask for anything outside that list, and the whole thing refuses to even start, rather than quietly leaving something out.

That last point is the one worth being careful about, because it's easy to claim and hard to actually prove. Does a helper agent's tool request get checked the same way the main agent's does, or does it get a separate, possibly weaker check? I didn't want to answer that from the docs alone, so I read the SDK's own source code directly. Every permission request, from the main agent or a helper, goes through the exact same check. There's no separate, weaker door.

That's the difference between a governance claim and a governance guarantee. One is a sentence in a README. The other is a fact you can point to a line of code for.

Here's the shape of it:

<a href="../assets/images/blogs/claude_agent_sdk_block_diagram.png" target="_blank" rel="noopener"><img src="../assets/images/blogs/claude_agent_sdk_block_diagram.png" alt="Claude Agent SDK adapter: simple block view showing pre-check and post-check around the agent loop"></a>

Read left to right: a request comes in from a K9-AIF caller (1) and gets checked before anything else happens. Only once it clears that check does Claude's own agent loop even start (2). Claude reasons and decides which tools to call, but every one of those calls comes straight back through the same check before it's allowed to run (4). The model call itself (3) is the one box in the middle nothing here touches. Once the loop finishes, the result goes through one last check before it's handed back (5). Five steps. The SDK owns exactly one of them.

---

## The half that doesn't: inference governance

Here's where it actually breaks, and no amount of adapter cleverness closes it.

A native K9-AIF agent works because there's a point where the framework can swap in its own model client wherever a model call happens: it substitutes itself in, so the call gets routed through K9-AIF's own model router instead of going straight to whichever LLM the code would otherwise reach.

The Claude Agent SDK has no equivalent swap-in point, and I didn't want to just assume that, so I checked it four separate ways:

- I read every option the SDK lets you configure. None of them is "plug in your own model." You can pick which Claude model to use, but you can't hand it a different brain.
- I checked the one setting that looked like it might be a loophole, a low-level connection option. It turned out to control *how* the SDK talks to the Claude process, not *what* answers the questions.
- I searched the SDK's own code for any hidden way to redirect it elsewhere. There isn't one.
- I checked Anthropic's own documentation, which says plainly that redirecting inference like this isn't a supported way to use the SDK.

So the honest statement is simple: the Claude Agent SDK's own reasoning can't be routed through K9-AIF's model router. Not because the adapter is incomplete, but because that's how the product is built. It's Anthropic's own harness, thinking end to end inside a piece of Claude's own software that K9-AIF doesn't own.

The tempting move here would have been to build something that *looked* like it solved this anyway, logging the model's output somewhere and calling it "governed." That would have been the wrong move. A fake safeguard is worse than an honest limitation, because it teaches the next person to trust something that isn't actually there.

---

## Naming the boundary instead of hiding it

So the adapter says plainly what it is: governed on actions, not on thinking. Every tool call, every helper agent it spawns, every check before and after: real. What the model reasons about, moment to moment: outside the framework's reach, permanently. That's written down in the adapter's own documentation, not left as a surprise for whoever reads the code next.

If a solution genuinely needs both, action governance and inference governance together, the answer isn't this adapter. It's a much simpler approach: call Claude's plain API directly, write the tool-use loop yourself in K9-AIF's own agent code, and let every model call route through K9-AIF's model router like anything else. That trades away Claude's autonomous reasoning loop in exchange for full governance. This adapter trades the other way: keep Claude's own loop, and contain everything it's allowed to act on.

Neither is more "correct." They're different tradeoffs for different needs, and the framework should let you pick, not blur the line between them.

<a href="../assets/images/blogs/claude_agent_sdk_class_diagram.png" target="_blank" rel="noopener"><img src="../assets/images/blogs/claude_agent_sdk_class_diagram.png" alt="Claude Agent SDK adapter: class diagram"></a>

---

## What this actually buys an organization

This isn't just an architecture exercise. It's the difference between two real paths an organization standardizing on Claude actually faces.

Without an adapter like this, adopting the Agent SDK's autonomous loop means one of two things. Either every tool call it makes runs on trust, with whatever guardrails a team manages to bolt on by hand for this one integration. Or the organization holds back from the Agent SDK entirely, and loses the real productivity gain of letting Claude plan and carry out multi-step work on its own.

With this adapter, neither tradeoff is necessary, and none of it depends on trusting the model to behave. A Zero Trust check runs before the session even starts. Every action the agent attempts afterward, including anything a spawned helper agent tries, passes through the same security checks every other agent in the framework already answers to. Nothing gets waved through just because it arrived from a different SDK. That's governance end to end on the one axis that's actually about what the system does: every action, checked, every time, not sampled or reviewed after the fact.

None of those checks disappear once the moment passes, either. They're events the same way every other K9-AIF decision is an event: written to the framework's own logging and reporting layer, not just printed to a console and lost. An audit trail an organization actually has to produce, for a regulator or an internal review, comes out of that logging, not out of someone's memory of what the agent probably did.

That matters most exactly where "the model decided" isn't an acceptable answer: financial approvals, claims decisions, anything touching regulated data. An organization gets to use Claude's own reasoning for the parts of a process that genuinely benefit from it, without quietly widening what "agent" means to include unaudited, unbounded actions.

And because this follows the same adapter shape the framework already uses elsewhere, the next agent framework an organization needs to adopt gets the same treatment: contained by default, governed the same way, logged and reportable the same way. That's the actual value here, not this one integration by itself, but a repeatable answer to a question every organization building agentic systems eventually has to ask again.

K9X Studio support for building and wiring these agents visually, the same way it already does for native K9-AIF squads, is next.

Beyond that, I've been turning over a bigger question too: what it would look like for the K9X ecosystem as a whole, not just this one adapter, to sit efficiently alongside the rest of Claude's own product suite. No firm answer yet, just a thread worth pulling.

---

## Where this lives

`k9_adapters/claude_agent_sdk/`: `ClaudeAgentSDKOrchestratorAdapter`, `ClaudeAgentSDKPayloadMapper`, `K9ClaudeAgentSDKAdapter`. `claude-agent-sdk==0.2.128` pinned in `requirements.txt`. The package's own `CLAUDE.md` records the verified facts above, including the exact `_internal/query.py` check for subagent routing, so the next person working in this folder doesn't have to re-derive any of it from scratch.

The goal was never to make the model reason the way K9-AIF wants. It's to make sure that whatever it decides to do next has to clear a gate first, and to be straight about the one place that gate doesn't reach.

---

## References

- K9-AIF Framework: [github.com/k9aif/k9-aif-framework](https://github.com/k9aif/k9-aif-framework)
- Anthropic, Building Effective Agents: [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)
- Claude Agent SDK docs: [code.claude.com/docs/en/agent-sdk/overview](https://code.claude.com/docs/en/agent-sdk/overview)
- `k9_adapters/claude_agent_sdk/`: in the framework repo above
- PyPI (k9-aif): [pypi.org/project/k9-aif](https://pypi.org/project/k9-aif/)
