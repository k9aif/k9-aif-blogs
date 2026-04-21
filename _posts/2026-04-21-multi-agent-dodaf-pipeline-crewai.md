---
title: "Designing a Multi-Agent DoDAF 2.0 Pipeline with CrewAI"
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

![Agentic DoDAF Pipeline](../assets/images/blogs/dow-crewai-blog.png)

*Figure: Multi-agent DoDAF 2.0 pipeline showing document routing, orchestration, and stage-based execution.*

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

# The 6-Stage DoDAF Pipeline

The pipeline is divided into six stages, each aligned with DoDAF 2.0 viewpoints and lifecycle practices:

## Stage 1 — Fit-for-Purpose (F2P) Gate

Determines whether full architectural processing is required.

- Applies rules-based evaluation  
- Produces a routing decision  
- Prevents unnecessary computation  

---

## Stage 2 — Architecture Scope & ICD Framing

Initializes the architecture:

- capability definition  
- scope identification  
- gap analysis  

This stage begins the **Initial Capabilities Document (ICD)**.

---

## Stage 3 — Operational Data Extraction

Extracts operational facts:

- actors  
- activities  
- operational flows  

Outputs structured data aligned with DoDAF’s **Operational Viewpoint (OV)**.

---

## Stage 4 — System & Service Modeling

Maps operational requirements to systems and services:

- system relationships  
- service interactions  
- traceability to operational needs  

Supports **System View (SV)** and **Service View (SvcV)**.

---

## Stage 5 — Performance & Standards Analysis

Evaluates:

- Key Performance Parameters (KPPs)  
- Key System Attributes (KSAs)  
- standards compliance  

Aligns with **Performance View (PV)** and **Standards View (StdV)**.

---

## Stage 6 — Final Document Assembly

Assembles outputs into structured deliverables:

- ICD document  
- summary artifacts  
- architecture outputs  

No new architecture is created here—this stage focuses on presentation.

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

A **central orchestrator** manages:

- stage execution order  
- data flow between squads  
- conditional routing based on Stage 1 output  

This enables:

- deterministic flow control  
- flexible execution paths  
- extensibility for additional stages  

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

This approach transforms complex systems engineering workflows into **manageable, scalable, and governed pipelines**.

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

