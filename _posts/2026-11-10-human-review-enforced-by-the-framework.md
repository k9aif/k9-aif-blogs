---
layout: post
title: "When a Hijacked Agent Gets Past the Shield"
date: 2026-11-10
author: Ravi Natarajan
categories: [architecture, ai, agent-systems, security]
tags: [K9-AIF, k9x Shield, HIL, human in the loop, tool calls, business rules, k9x SATAN]
---
<!-- DRAFT, local only. Do not push until 1.25.0 ships and the tests named below exist. -->

## What K9-AIF already does, out of the box

Every request in K9-AIF takes one path: Router → Orchestrator → Squad → Agent. Each layer already comes with its own protections:

![What K9-AIF provides out of the box, layer by layer](../assets/images/blogs/k9aif-oob-capabilities.png)

| Layer | Capability | Classes that ship today | What it does |
|---|---|---|---|
| Router | Identity | `OIDCAuthenticator` (`KeycloakAuthenticator`, `EntraIDAuthenticator`, `OktaAuthenticator`), `ApiKeyAuthenticator` | Authenticates the caller before anything runs |
| Orchestrator | Zero Trust | `DefaultZeroTrustGuard`, `RoleBasedAuthorizationGuard` | Checks the caller's role is allowed this action, scores risk, masks sensitive data |
| Orchestrator | Human review | `RequiresHIL`, `BaseHILOrchestrator` | Publishes a review to `hil.requests.<queue>`; the flow waits, for days if needed, and resumes exactly once on the decision |
| Squad | Business rules | `K9RulesAgent` with `K9RulesAdapter` (YAML) or `RestRulesAdapter` (a decision service such as IBM ODM or Drools) | Applies the solution's own rules. The rules belong to the solution; the framework supplies the adapters |
| Agent | k9x Shield | `ShieldGovernance` with 15 `BaseVulnerabilityCheck` checks | Screens input, documents, output and tool calls on every agent; the agent can't opt out. SATAN fires an attack at each check to prove it holds |
| Agent | One path to the model | `llm_invoke`, `K9ModelRouter` | Agents never call a model directly; the router picks the model by policy |
| Agent → tools | Governed tools | `MCPStreamableHttpConnector`, `@govern_tool_call`, `@govern_tool_result` | Calls vetted tools on an MCP server, with the call and the result screened |
| All layers | Secrets | `VaultSecretAdapter`, `IbmSecretAdapter`, `AwsSecretAdapter`, `EnvSecretAdapter` | Secrets come from a secret manager, never from config |

<!-- RAVI: your field incident goes here. -->

That covers a lot. This post is about the one case it doesn't fully cover yet: **an agent that has been hijacked and gets past Shield anyway.**

## The hijacked agent

![Today: the honest agent raises RequiresHIL; a hijacked agent goes straight to the tool](../assets/images/blogs/hil-review-today.png)

A supplier invoice carries an instruction worded so that no pattern in Shield's checks matches it. The model now wants to release a payment to a new account. A well-behaved agent would raise `RequiresHIL` here. A hijacked one simply doesn't. Shield's egress checks see an approved tool with well-formed arguments and pass it. The payment goes out, and nobody was asked.

Nothing in that path is broken. `release_payment` is a vetted tool on the MCP server; the attacker didn't need an unapproved one, only an approved one called with their account number. Every check did its job, but every check answers **allow or block**. "Ask a person first" was left to the agent's own judgement, which is exactly what the attacker took over. It doesn't matter how the tool is called, whether through an MCP client, the Claude Agent SDK, CrewAI or LangGraph: none of those paths can hold a call for a person unless the agent asks.

Shield lowers the odds of a hijack. No detector catches everything, so the next question is: **if an agent is hijacked anyway, what is the worst it can do?** For an irreversible action, the answer should be "propose it to a person", not "do it".

<!-- RAVI: your opinion here. -->

## How it gets caught (planned, 1.25.0)

The change takes that decision away from the agent. Agents stop executing tools themselves. They propose a call, and a dispatcher owned by the framework decides, using the solution's own business rules at the moment of the call.

![Planned 1.25.0: Shield, the registry and the solution's own rules decide; a review is published by the framework, and the tool runs only on approval](../assets/images/blogs/hil-review-planned.png)

1. **Shield** screens the call, as today.
2. The **tool registry** says whether the tool is irreversible. A tool nobody declared counts as irreversible.
3. The **solution's rules**, through the same `BaseRulesAdapter` it already uses, decide one of three things:
   - **allow**: the call runs;
   - **deny**: it's refused, for example any wire to an external account;
   - **HIL**: the flow pauses and a person decides. An irreversible call that no rule allows goes here by default.
4. On a HIL decision, only a well-formed approval, before the deadline, runs the call. Rejected, expired or malformed means it never runs. Every step is recorded in an audit log.

The framework gives the rules two facts they never had. `irreversible` comes from the registry. `untrusted_input` says whether outside content, such as a document, a web page or another agent's reply, entered this request. A clerk's intent never came from the invoice; an agent's can. So a rule can say: *an irreversible call in a request that carried outside content goes to a person.* That's the rule that catches our invoice.

Three details make the gate hold:

- **The framework writes the review, not the agent.** The reviewer sees the exact tool and arguments that were intercepted. The agent's explanation is shown separately, labelled untrusted.
- **The framework publishes the review itself.** If it only raised an exception, the exception would travel back up through the agent's own code, which could catch it and carry on.
- **An approved call runs at most once.** "Executing" is recorded before the call. If the process dies mid-call, the call goes back to a person instead of being retried; a duplicate payment is worse than a delay.

It doesn't slow the system down. The dispatcher runs inside each worker, as Shield does, with no central service to queue through, and a paused flow is a row in the state store, not a blocked thread. The rules decide which calls reach a reviewer, so reviewer effort follows risk, not traffic. The existing `RequiresHIL` path stays as it is.

## Where agent identity fits

IBM made [Agent Identity](https://www.ibm.com/new/announcements/ibm-agent-identity-is-now-generally-available-building-trust-for-the-agentic-era) generally available on 26 October 2026, with IBM Verify and HashiCorp Vault. It registers each agent with an owner, a risk class and its approved tool connections, issues on-behalf-of tokens that carry both the user and the agent, and uses Vault for short-lived credentials. It answers *is this the right agent, acting for the right person?* It decides when a token is requested, [as IBM documents it](https://www.ibm.com/docs/en/agent-identity?topic=overview), and our hijacked agent would pass: it is registered, holds a valid token and acts for a real clerk. Whether the account number came from the invoice is known only inside the agent runtime. So the two fit together: identity for *the right agent*, the dispatcher for *the right action*. Verify speaks OIDC, like the identity providers K9-AIF already supports.

## What it will and won't cover

<!-- Fill from the Step 1 report and the 1.25.0 tests. Every line must name the test that proves it. -->

| Covered (planned, each backed by a test) | Not covered |
|---|---|
| A hijacked agent that never raises `RequiresHIL` still has its irreversible call stopped for review | Agent code that opens its own network connection. Python can't block that from inside the process; the k9x Inspector will flag it at review time, and deployment isolation is the runtime answer. |
| Approved runs once; rejected, expired or malformed never run | A reviewer who approves a bad call. The review shows exactly what will happen; it can't make the decision. |
| A tool nobody declared is treated as irreversible | Reversible tools: they run without review unless a rule says otherwise. |
| A rules-engine failure means the call doesn't run | The quality of the solution's rules. The framework applies them to every call; it can't write them. |
| SATAN's `HILBypassAttack` fires the hijack above and is stopped | |

## For developers: the classes

The new classes on the left are planned for 1.25.0. Everything on the right exists today and is used as it is. The dispatcher is the only caller of `call_tool()`.

![Planned 1.25.0 classes and the existing K9-AIF ABBs they use](../assets/images/blogs/hil-review-classes.png)

| Class | Role |
|---|---|
| `BaseToolDispatcher` | Receives every proposed call; screens it, looks it up, applies the rules, then runs, refuses or sends it for review |
| `ToolRegistry` / `ToolSpec` | Declares each tool and whether it is irreversible; locked framework config |
| `ToolCall` | The proposed call, with the agent's rationale kept in a field labelled untrusted |
| `ToolCallApprovalRequest` | The review a person sees, built from the intercepted call |
| `ToolCallDecision` | Reads the reviewer's reply strictly; anything but a valid approval means no |
| `ToolCallAuditLog` | Records every call, decision, decider and time |

<!-- RAVI: closing thought in your words. -->
