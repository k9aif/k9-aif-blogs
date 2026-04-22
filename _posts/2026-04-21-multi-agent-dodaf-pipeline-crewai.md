---
title: "A Multi-Agent DoDAF 2.0 Pipeline — An Architecture-First Approach"
date: 2026-04-21
author: Ravi Natarajan
tags: [CrewAI, DoDAF, Agentic AI, Systems Engineering, Architecture, Neo4j]
---
Modern systems engineering—especially under frameworks like DoDAF 2.0—requires processing large volumes of structured and unstructured inputs, generating multiple architectural views, and maintaining traceability across the entire lifecycle.

Traditionally, this process is manual, time-consuming, and difficult to scale.

This post presents an architecture-first approach to solving that problem using a **multi-agent pipeline built with CrewAI**, aligned with DoDAF 2.0 principles.

---

# The Core Idea

Instead of treating architecture generation as a monolithic workflow, we structure it as:

> **An orchestrated system of specialized agents, grouped into stages, each responsible for a well-defined architectural function.**

This enables:

- modular execution
- clear separation of concerns
- traceability across architectural artifacts
- governed and repeatable processing

---
The following diagram illustrates the end-to-end agentic pipeline:

<figure style="text-align:center; margin: 2rem 0; background:#071a24; padding:1.25rem; border-radius:12px;">
  <img src="../assets/images/blogs/agentic_dodaf_full_pipeline.svg"
       alt="Agentic DoDAF 2.0 pipeline"
       style="width:100%; max-width:860px; height:auto; display:block; margin:0 auto;">
  <figcaption style="margin-top:0.65rem; font-size:0.92rem; color:#5f6b76;">
    Figure: Agentic DoDAF 2.0 pipeline — document classification, orchestrated stage execution, and multi-pipeline traceability.
  </figcaption>
</figure>
---

# High-Level Architecture

The system follows a layered agentic architecture:

Pipeline (SBB)
→ Principal Orchestrator
→ Stage Squads
→ Specialized Agents
→ Outputs (ICD, OV, SV, PV)

- **Orchestrator** coordinates execution
- **Squads** represent logical stages
- **Agents** perform specific tasks
- **Outputs** are structured DoDAF artifacts

---

# The 6-Stage DoDAF-SE Pipeline

The pipeline is executed as a sequence of **stage-specific squads**, where each squad is responsible for a distinct architectural function aligned with DoDAF 2.0.

## Stage 1 Squad — Identify Intended Use (F2P Gate)

- Performs Fit-for-Purpose evaluation
- Determines document type and mission context
- Produces routing decisions and constraints

---

## Stage 2 Squad — Define Architecture Scope

- Establishes scope boundaries
- Extracts capability gaps
- Seeds ICD Sections 1–2

---

## Stage 3 Squad — Extract Operational Data

- Develops OV viewpoint inputs
- Extracts tasks, activities, and mission flows
- Identifies operational performers and interactions

---

## Stage 4 Squad — Build System & Service Views

- Develops SV/SvcV viewpoints
- Identifies systems, services, and interfaces
- Establishes DM2 correlations

---

## Stage 5 Squad — Performance & Standards Analysis

- Develops PV and StdV viewpoints
- Defines performance measures and constraints
- Produces KPP/KSA seeds

---

## Stage 6 Squad — Present Results

- Assembles the initial ICD and F2P summary
- Applies templates and inserts diagrams
- Produces deliverables for HIL review

The resulting **initial ICD** serves as input to the **JCIDS pipeline**, which performs further capability development.

---

# Multi-Agent Design with CrewAI

Each stage is implemented as a **CrewAI squad**:

- multiple agents collaborate within a stage
- agents share intermediate outputs
- results are validated before moving forward

This provides:

- isolation between stages
- better error containment
- improved reasoning through collaboration

---

# Orchestration Model

A **Principal Orchestrator** acts as the central coordination layer across the system.

It is responsible for:

- interpreting routing decisions from the intake layer
- triggering the appropriate pipeline
- managing stage execution within each pipeline
- controlling data flow between stages and downstream systems

The orchestrator ensures that execution is **structured, deterministic, and traceable**, while still allowing flexibility in how different pipelines are invoked.

---

## Coordinated Pipeline Execution

The system operates across three coordinated pipelines:

- **DoDAF 2.0 Pipeline** → performs architectural decomposition and produces the initial ICD and full architectural views
- **JCIDS Pipeline** → consumes the initial ICD and performs capability development (CDD, CPD, KPP/KSA, DOTmLPF-P)
- **Systems Engineering (DAU) Pipeline** → produces engineering artifacts (SRD, SPS, TEMP, TPMs)

Each pipeline is independently orchestrated but connected through structured outputs and inputs.

The **Principal Orchestrator governs the transitions between these pipelines**, ensuring a consistent and traceable flow from:

> architecture → capability → systems engineering

---

# Technology Architecture

The pipeline integrates multiple technologies, each serving a distinct purpose:

- **CrewAI** → multi-agent orchestration
- **Neo4j** → graph-based architectural data (DM2 alignment)
- **PostgreSQL** → structured stage outputs
- **Rules Engine** → deterministic governance
- **Event Bus (Kafka/Redpanda)** → stage coordination
- **Diagram Generators (PlantUML / SysML tools)** → view rendering

This separation ensures:

- no overlap in responsibilities
- clear data ownership
- scalable architecture

---

# Why This Approach Matters

This architecture introduces a key shift:

> **From document processing → to orchestrated architectural reasoning**

Key benefits:

- end-to-end traceability
- modular architecture generation
- governed decision-making
- scalable multi-agent execution

---

# Key Insight

The most important takeaway is not the use of agents—but **how they are structured**.

> When architecture workflows are modeled as orchestrated agent systems, they become:
>
> - composable
> - extensible
> - and production-ready

---

# Closing Thoughts

Agent frameworks alone are not enough.

The real value comes from combining:

- **architecture-first design**
- **clear orchestration patterns**
- **structured stage decomposition**

This approach transforms complex systems engineering workflows into
**manageable, scalable, and governed pipelines**.

K9-AIF is designed to address exactly this gap. Its squad-based decomposition, orchestration hierarchy, and SBB-driven pipeline structure reflect core architectural patterns that move agentic systems from experimentation to structured, enterprise-ready execution.

While demonstrated here in a DoDAF context, these patterns apply broadly to any governed, enterprise-grade agentic system.

> **Agentic AI becomes truly valuable only when it is architected first.**

---

# Next Steps

Future enhancements could include:

- human-in-the-loop approval stages
- dynamic routing based on mission context
- deeper integration with modeling tools
- automated generation of architectural views

---

If you’re exploring agentic systems in enterprise or systems engineering contexts, this pattern provides a strong foundation to build on.

---
