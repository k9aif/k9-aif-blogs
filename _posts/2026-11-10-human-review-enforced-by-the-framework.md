---
layout: post
title: "Human Review the Agent Can't Skip"
date: 2026-11-10
author: Ravi Natarajan
categories: [architecture, ai, agent-systems, security]
tags: [K9-AIF, k9x Shield, HIL, human in the loop, tool calls, k9x SATAN]
---
<!-- DRAFT, local only. Do not push until 1.25.0 ships and the tests named below exist. -->

## Shield is on. So where is the gap?

In K9-AIF every agent extends `BaseAgent`, and every agent gets governance before and after it runs. With k9x Shield on, the incoming prompt and any attached document go through the ingress chain: prompt injection, PII boundary, memory poisoning, input size, request frequency and the rest. The output and any tool call go through the egress chain: tool authorization, tool arguments, execution guard, semantic drift, payment diversion, output sanitization. The framework ships 15 of these checks today, mapped to the OWASP LLM Top 10 and the Zscaler ThreatLabz 2026 findings.

<!-- RAVI: your field incident goes here. A case where an agent "asking" for approval was the only control. -->

That is a lot of screening. It still leaves one decision with the agent: **whether a human ever sees the call.**

Today, human review in K9-AIF starts when an agent or squad raises `RequiresHIL`. The Orchestrator catches it, publishes the request to `hil.requests.<queue>`, and the flow pauses until a reviewer replies. It works well when the agent is behaving. The question is what happens when it isn't.

![Today: the honest agent raises RequiresHIL; a hijacked agent goes straight to the tool](../assets/images/blogs/hil-review-today.png)

*An agent never calls a model directly: `llm_invoke` goes through `K9ModelRouter` to whichever model the routing policy picks.*

Follow the lower half of the diagram. A supplier invoice carries an instruction worded so that no pattern in the ingress chain matches it. Shield passes it. The model now wants to release a payment to a new account. A well-behaved agent would raise `RequiresHIL` here. A hijacked one simply doesn't. The egress checks then look at the tool call and see an approved tool with well-formed arguments, so they pass it too. The payment goes out, and nobody was ever asked.

Most tool calls in K9-AIF go through MCP, and the MCP server only exposes tools that have already been vetted. That holds here too: `release_payment` is a legitimate, approved tool. Vetting answers "is this tool allowed to exist?" It doesn't answer "should this particular call, with these arguments, happen right now?" The attack doesn't need an unapproved tool. It needs an approved one, called with the attacker's account number.

And it doesn't matter how the call is made. An MCP client, the Claude Agent SDK adapter, a CrewAI or LangGraph tool, or a function wrapped in `@govern_tool_call` all end the same way: the call is allowed or blocked. None of them has a third outcome, "hold this until a person says yes," unless the agent itself asks for one. So whichever path the call takes to the vetted tools on the MCP server, a hijacked agent still gets through.

Nothing in that path is broken. Each check did its job. The gap is that the checks answer **allow or block**, and "ask a human first" was left to the agent's own judgement, which is exactly what the attacker took over.

## Detection lowers the odds. The gate limits the damage.

Shield's checks are deterministic, and that is a strength: the same input always gets the same answer, and SATAN proves each one holds. But a deterministic check can only catch what it was written to catch. An attacker who words an injection differently gets past a pattern list. That is not a flaw in Shield; it is the nature of detection.

So the design question is not "how do we catch every injection?" It is: **if an agent is hijacked anyway, what is the worst it can do?** For an irreversible action, the answer should be "propose it to a human." Not "do it."

<!-- RAVI: your opinion here. Is agent-raised HIL useless, or just insufficient on its own? Is irreversible-by-default worth the approval load? -->

## The plan for 1.25.0

The change moves the decision from the agent to the framework. Agents stop executing tools. They propose calls, and one dispatcher, owned by the framework, decides what happens next.

![Planned 1.25.0: the dispatcher intercepts the call, publishes the review itself, and executes only on approval](../assets/images/blogs/hil-review-planned.png)

| Piece | What it does |
|---|---|
| `ToolRegistry` | Adds one fact to the MCP tools you have already vetted: `irreversible: true \| false`. A tool that isn't declared is treated as irreversible. The registry is a locked framework setting, so a solution's config can't quietly mark a payment tool as reversible. |
| `ToolCall` | What the agent proposes: tool name, arguments, originating request and task, agent identity, correlation ID. The agent's explanation travels in a field labelled `rationale_untrusted`. |
| `BaseToolDispatcher` | The only component that executes a tool. It runs the Shield egress checks, looks the tool up, and for an irreversible tool it pauses the flow instead of executing. |
| `ToolCallApprovalRequest` | The review the human sees, built by the framework from the intercepted call, never from the agent's description of it. |
| `ToolCallDecision` | Reads the reply strictly. Only a well-formed `approved` from an authorized reviewer, before the deadline, executes. Rejected, expired or malformed all mean the tool is not called. |
| `ToolCallAuditLog` | Records every intercepted call, the decision, who made it and when. |

Two design choices matter more than the class list.

**The dispatcher publishes the review itself.** It would be simpler to have the dispatcher raise `RequiresHIL` and let the Orchestrator publish, as it does today. But that exception would pass back up through the agent's own code on the way, and a hijacked agent could catch it and carry on. The tool still wouldn't run, but no human would see the request either. So the dispatcher publishes first, then signals the pause.

**At most once, not exactly once.** On approval the dispatcher writes "executing" to the audit log before calling the tool. If the process dies mid-call, it does not retry; the call goes back to a person. For a payment, a duplicate is worse than a delay.

The existing `RequiresHIL` path stays. An agent can still ask for review on its own judgement, for a fraud score or a low-confidence answer. The new gate sits underneath it for the cases the agent should not get to decide.

## What it will and won't cover

<!-- Fill from the Step 1 report and the 1.25.0 tests. Every line here must name the test that proves it. -->

| Covered (planned, each backed by a test) | Not covered |
|---|---|
| A hijacked agent that never raises `RequiresHIL` still has its irreversible call sent for review, and not executed | Agent code that opens its own network connection or starts its own process. Python can't stop that from inside the process. The k9x Inspector rule `K9-TOOL-001` will flag it at review time, and isolation at deploy time is the runtime answer. |
| Approved executes once; rejected, expired or malformed never execute | A reviewer who approves a bad call. The review shows the exact tool and arguments; it can't make the decision. |
| A tool nobody declared is treated as irreversible | Reversible tools. They run without review, by design, under the same Shield checks. |
| SATAN's `HILBypassAttack` fires the hijack above and is stopped | |

<!-- RAVI: closing thought in your words. -->
