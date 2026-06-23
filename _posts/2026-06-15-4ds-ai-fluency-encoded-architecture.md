---
layout: post
title: "The 4Ds of AI Fluency — Encoded Into Architecture"
date: 2026-06-15
author: Ravi Natarajan
---

I've been working through the AI Fluency course from Rick Dakan and Joseph Feller — developed in collaboration with Anthropic. It's excellent. The core of it is a framework of four competencies for working effectively with AI, called the 4Ds: Delegation, Description, Discernment, and Diligence.

Halfway through the Delegation module, I stopped.

_We already do this. Every single one of them. But not at runtime — at design time._

That's the claim I want to unpack here.

---

## The 4Ds, Briefly

Dakan and Feller frame AI engagement across three modes: Automation (AI executes specific tasks), Augmentation (you and AI collaborate), and Agency (you guide AI to work independently, shaping its knowledge and behavior rather than specific actions). The 4Ds apply across all three, but they matter most at the Agency tier — where the stakes, the complexity, and the surface area for error are highest.

| Competency | What it means |
|---|---|
| **Delegation** | Deciding what work AI should do, what you should do, and what you do together |
| **Description** | Communicating your goals, process, and expectations clearly to AI systems |
| **Discernment** | Evaluating AI outputs critically before acting on them |
| **Diligence** | Taking responsibility for AI collaboration — ethically, transparently, safely |

Most frameworks leave these to the developer. Understand the task. Write a good prompt. Check the output. Use AI responsibly. Good advice. Hard to enforce.

K9-AIF takes a different position: these aren't runtime behaviors — they're design-time decisions, encoded into the architecture before a single line runs.

---

## Delegation → K9X Studio Canvas

Dakan and Feller break Delegation into three components: Problem Awareness (understanding your goals and the work involved), Platform Awareness (knowing what different AI systems can do), and Task Delegation (strategically distributing work between human and AI).

In K9-AIF, you do all three before you write a line of code — on the Studio canvas.

When you design an architecture in K9X Studio, you're making explicit delegation decisions: which concerns belong to the Router, which to the Orchestrator, which Squad owns a domain, which Agent executes a specific task. The hierarchy — Router → Orchestrator → Squad → Agent — is a delegation structure. Every node is an explicit answer to the question: *who does this work?*

I've seen developers put routing logic inside an agent. It makes intuitive sense to them — the agent sees the output, so surely it decides what happens next. The canvas makes the mistake visible immediately: an Agent sits at the bottom of a four-tier hierarchy with no upward visibility. It doesn't know about Squads, Orchestrators, or routing decisions. That's not a constraint — that's the delegation encoded into the structure.

The Studio's Platforms palette also handles Platform Awareness directly: you declare which frameworks (CrewAI, LangChain, Watsonx), messaging systems, databases, and deployment targets are in scope. That selection flows into the generated `config.yaml`. Awareness made concrete, before any code is written.

---

## Description → ABB Contracts

Description, per the framework, covers three things: what you want the AI to produce (Product), how you want it to approach the work (Process), and how you want it to behave (Performance).

In K9-AIF, that's the Architecture Building Block (ABB).

An ABB isn't a prompt. It's a contract — a formal, machine-checkable specification of what a component produces (`execute(payload: dict) -> dict`), how it approaches its work (the governance pipeline, lifecycle hooks, squad flow), and how it should behave (synchronous, stateless, governance-enforced). Every SBB you build is an answer to a description that was written at the ABB layer before you started.

The distinction matters: natural language prompts are interpreted at runtime, with all the ambiguity that entails. ABB contracts are validated structurally. Description isn't something you do when you invoke the agent — it's something the architecture establishes before the agent exists.

---

## Discernment → k9aif inspect

Discernment is critical evaluation of AI outputs. Did the system do what you intended? Is the output trustworthy? Does the behavior match the design?

The `k9aif inspect` tool addresses a specific slice of this: at design time, it checks whether your SBB implementation correctly extends and complies with its ABB contract. Did you build what you described? Are the layer boundaries intact? Are you calling governance where the contract requires it?

Worth being precise here: this is structural compliance, not runtime output evaluation. It doesn't assess whether the LLM's answer was good. It assesses whether the component was built correctly — before it runs. That's a narrower claim than the full scope of Discernment, but it's an important one: most discernment failures in complex agentic systems start with architectural drift, not bad outputs. Catching the drift before anything runs is the design-time version of the competency.

---

## Diligence → The Full K9-AIF Discipline

Diligence in the 4Ds framework has three components: Creation Diligence (being thoughtful about which AI systems you use and how), Transparency Diligence (being honest about AI's role in your work), and Deployment Diligence (taking responsibility for what you ship).

This is where K9-AIF's position is strongest — and most different from a behavioral guideline.

Creation Diligence is handled by Zero Trust and the GovernanceAgent: every component operates under the assumption that inputs must be verified, permissions must be explicit, and no agent is trusted by default. You can't accidentally skip it in production — the architecture blocks you.

Transparency Diligence is the audit trail: every routing decision, model invocation, and governance check is persisted to the routing state store. The system's role in every output is logged structurally, not declared after the fact.

Deployment Diligence is the PolicyEngine and layer boundary enforcement: the architecture prevents patterns that violate its contracts from reaching deployment. The responsibility isn't assumed from the developer — it's enforced by the structure.

The 4Ds frame Diligence as personal responsibility. In K9-AIF it's architectural enforcement. You still need personal responsibility at adoption time — choosing what to deploy, what data it touches, what governance policies it enforces. But once it's running, the system doesn't rely on the developer remembering to be diligent.

---

## What This Means

The 4Ds describe what a skilled human does with AI. Rick Dakan and Joseph Feller built a framework that should be taught widely, because most people working with AI aren't doing these things intentionally.

K9-AIF encodes them into the architecture so the system does them by default — at design time, before anything runs.

That's not a replacement for AI Fluency. You still need Delegation thinking to design the canvas well, Description precision to write good ABBs, Discernment to interpret what `k9aif inspect` flags, and Diligence to set the right governance policies. The competencies still live in the architect.

What changes is the floor. A developer working within K9-AIF inherits better defaults than one working with a framework that leaves the 4Ds to runtime judgment. The architecture raises the minimum.

That seems worth saying out loud.

---

*The AI Fluency Framework and the 4Ds were developed by Rick Dakan and Joseph Feller in collaboration with Anthropic. Copyright 2025 Rick Dakan, Joseph Feller, and Anthropic. Released under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) license. This post references The AI Fluency Framework by Dakan and Feller. Supported in part by the Higher Education Authority, Ireland, through the National Forum for the Enhancement of Teaching and Learning.*
