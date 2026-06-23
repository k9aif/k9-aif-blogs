---
layout: post
title: "Abstraction That Holds — How ABB/SBB Keeps K9-AIF Extensible Without Breaking"
date: 2026-06-23
author: Ravi Natarajan
---
## The Power of Abstraction

> "The entire history of software engineering is one of rising levels of abstraction. This is as it was, is now, and always shall be." — Grady Booch

It was 1993. C++ was incredibly popular, and Grady Booch's *Object-Oriented Analysis and Design with Applications* was the book that changed how I thought about software. It made me transition from writing procedural code to thinking in objects — in contracts, in hierarchies, in abstractions that held. Inheritance, multiple inheritance, dynamic binding, polymorphism — these were not textbook concepts to me. They were tools that made software *beautiful*. You could write something that held its shape under change. You could build a system where adding a new capability didn't require rewriting what already worked.

From that point on, every project I touched started to incorporate that approach. I coupled it with STL in C++, began using Rogue Wave's Tools.h++ — libraries that themselves were built on the same principles of abstraction and reuse. Those are the strong building blocks for Software Engineering.

In the late 1990s, while designing a C++ application for SITA — an airlines baggage tracking system — I had the opportunity to interact directly with Bjarne Stroustrup, the creator of C++. I reached out to him about my design approach, and he responded confirming it was right. That validation, from the creator of C++ himself, reinforced what his writings and Booch's had already taught me. I credit both of them for shaping how I approach software design to this day.

And from then onwards, I have applied OOA, OOD, and Design Patterns everywhere, in every solution I have designed and built. Over thirty-plus years, that idea — that abstraction is the mechanism by which software survives change — has guided how I think about every system.

Abstraction is not about hiding complexity. It is about drawing the right lines — the lines that hold when requirements change, when providers change, when teams change, when the enterprise scales.

A well-defined abstract class is a contract. It says: "Here is what you must provide. Here is what I guarantee. Everything else is your business." When that contract is right — when it captures the essential interface without leaking implementation details — you can swap the concrete implementation without touching the caller. You can add a new implementation without modifying the framework. You can test the contract independent of any specific provider.

This is Liskov. This is the Open-Closed Principle. This is what every Enterprise Architect and AI Architect learned — or should have learned — before they ever touched an LLM.

---

## What Is Happening in the Agentic AI Space

Everyone is diving into agentic process flows. The LLM is the new hammer, and everything looks like a nail. Teams spin up agents, wire them together, call an LLM, get a result, call it a POC — and declare success.

But look under the hood. What's actually there?

One big monolithic script. Hardcoded provider calls. No separation between the workflow and the infrastructure. No contracts. No abstraction boundaries. No governance. The prompt, the model selection, the data retrieval, the orchestration logic — all tangled together in a single file, written by an LLM that was told "make this work," not "make this last."

And it does work. For that one demo. For that one model. For that one use case.

Then someone asks: "Can we swap the LLM provider?" Rewrite everything. "Can we add a governance check before the inference call?" Rewrite everything. "Can we switch the vector database?" Rewrite everything. "Can another team reuse this agent in a different flow?" They can't. The agent doesn't know where its own logic ends and the orchestration begins.

This is not engineering. This is prototyping without discipline — and it is dangerous, because these POCs get promoted as production-ready, and the technical debt they carry is invisible until it's too late.

---

## ABB and SBB — TOGAF Applied to Agentic AI

K9-AIF's architecture is built on a separation that comes directly from TOGAF:

**Architecture Building Blocks (ABB)** — abstract contracts. They define the interface, the lifecycle, the governance hooks. They never contain domain logic. They live in `k9_core/`.

**Solution Building Blocks (SBB)** — concrete implementations. They extend ABBs with real behavior — domain-specific, provider-specific, organization-specific.

The framework ships OOB (out-of-the-box) SBBs — ready-to-run defaults that work with zero configuration. You don't have to write an embedding service to use vector retrieval. But when your organization needs a different provider, you extend the ABB, register your SBB, set one config key — and the framework doesn't change.

The key principle: **arrows only go down**. ABBs don't know which SBB will fulfill them. Factories don't know which provider is configured until runtime. SBBs extend upward — but never modify the layer above.

---

## How the ABB Contract Works — A Concrete Example

Here is the ABB contract for the vector database in K9-AIF:

```python
class BaseVectorDB(BaseComponent, ABC):

    @abstractmethod
    def insert(self, doc_id: str, embedding: List[float],
               metadata: Dict[str, Any]) -> None:
        raise NotImplementedError

    @abstractmethod
    def search(self, query_embedding: List[float],
               top_k: int = 5) -> List[Dict[str, Any]]:
        raise NotImplementedError

    @abstractmethod
    def delete(self, doc_id: str) -> None:
        raise NotImplementedError
```

That contract is stable. It defines `insert`, `search`, `delete` — the essential operations. Nothing about ChromaDB. Nothing about Milvus. Nothing about any specific provider. Every vector database implements these three methods and registers with the factory.

The OOB SBB for ChromaDB:

```python
class ChromaDBAdapter(BaseVectorDB):

    def __init__(self, config=None, monitor=None):
        super().__init__(name="ChromaDBAdapter", monitor=monitor)
        self._config = config or {}
        self._client = None

    def insert(self, doc_id, embedding, metadata):
        self._ensure_client()
        self._collection.add(
            ids=[doc_id], embeddings=[embedding], metadatas=[metadata])

    def search(self, query_embedding, top_k=5):
        self._ensure_client()
        results = self._collection.query(
            query_embeddings=[query_embedding], n_results=top_k)
        return [{"text": doc, "score": 1 - score, "metadata": meta}
                for doc, score, meta in zip(...)]

    def delete(self, doc_id):
        self._ensure_client()
        self._collection.delete(ids=[doc_id])
```

An enterprise team that needs Milvus writes their own `MilvusAdapter` — same three methods, different provider underneath. They register it with the factory, set one config key — and the framework, the retriever, the agents, the squads — none of them know or care:

```yaml
vectordb:
  provider: milvus
```

No code changes. No rewrite. No touching the agents, the squads, the orchestrators, or the retriever. That is what abstraction buys you.

---

## Consistency Across the Enterprise

When three teams in three business units build agentic applications on K9-AIF, they share the same ABB contracts. Their agents implement the same `execute(payload) -> dict`. Their squads run the same flow engine. Their governance follows the same pipeline.

But their SBBs are entirely their own.

| What stays the same (ABB)                  | What varies (SBB)                             |
| ------------------------------------------ | --------------------------------------------- |
| `BaseAgent.execute()` contract           | Domain logic inside`execute()`              |
| `BaseVectorDB.search()` interface        | ChromaDB, Milvus, Qdrant, PgVector            |
| `BaseEmbeddingService.embed()` interface | Ollama, OpenAI, Watsonx                       |
| Squad flow engine +`context_keys`        | Which agents, what flow, what scoping         |
| Governance pipeline hooks                  | What gets checked, what gets blocked          |
| `K9ModelRouter` scoring                  | Custom routers with different scoring signals |

Code developed across projects, across business units, across the enterprise remains **consistent**. Every team follows the same contracts. Every agent is testable the same way. Every infrastructure concern is swappable the same way. A developer who learns the pattern on one project applies it immediately on the next.

This is what OOA, OOD, and the discipline of patterns give you. Not theoretical elegance — practical survival.

---

## The Discipline Behind Every Feature

Every capability in K9-AIF follows the same pattern:

```
k9_core/<concern>/base_<concern>.py       ← ABB contract (abstract)
k9_<concern>/<impl>/<name>_service.py     ← SBB implementation (concrete)
k9_factories/<concern>_factory.py         ← Config-driven wiring
config.yaml                               ← Provider selection at deployment
```

This pattern now covers nine concerns — and growing:

| Concern           | ABB                      | OOB SBB                      | Factory                     |
| ----------------- | ------------------------ | ---------------------------- | --------------------------- |
| Inference         | `BaseLLM`              | `OllamaLLM`, `OpenAILLM` | `LLMFactory`              |
| Model Routing     | `BaseModelRouter`      | `K9ModelRouter`            | `ModelRouterFactory`      |
| Embedding         | `BaseEmbeddingService` | `OllamaEmbeddingService`   | `EmbeddingServiceFactory` |
| Vector Store      | `BaseVectorDB`         | `ChromaDBAdapter`          | `VectorDBFactory`         |
| Retrieval         | `BaseRetriever`        | `K9Retriever`              | `RetrieverFactory`        |
| Cache             | `BaseCache`            | `InMemoryAdapter`          | `CacheFactory`            |
| Secret Management | `BaseSecretManager`    | `EnvSecretAdapter`         | `SecretManagerFactory`    |
| Agent             | `BaseAgent`            | `K9ValidationLoopAgent`    | `AgentRegistry`           |
| Governance        | `BaseGovernance`       | `NoopGovernance`           | `require_governance()`    |

When the tenth concern comes — and it will — it follows the same structure. It will not break the other nine. That guarantee is not a hope. It is an architectural property that comes from doing OOA and OOD correctly.

---

## A Message to Enterprise Architects and AI Architects

Let me be direct about something.

OOA is not outdated. OOD is not outdated. UML is not outdated. Design Patterns are not outdated. In fact, they are the solid foundation for any successful software design and development. They always have been.

The industry has a short memory. Every new technology wave brings voices that say the old disciplines no longer apply — that the new tool is so transformative it renders architecture thinking unnecessary. It happened with web development. It happened with cloud. It is happening now with generative AI and agentic flows.

It is wrong every time.

If you learned OOA and OOD — use it. Do not abandon thirty years of software engineering discipline because the LLM made the code compile on the first try. Compilation is not architecture. A working demo is not a production system. And a monolithic script that calls an LLM is not an agentic framework.

Abstraction is important. Building modularized code that complies with a contract is vital. Configuration-driven provider selection is not over-engineering — it is the mechanism that lets your system survive the next provider change, the next team handoff, the next compliance requirement.

Inheritance, polymorphism, the factory pattern, the adapter pattern, the open-closed principle — these are not relics of the 90s. They are the reason some systems last and others are rewritten every eighteen months. They are the foundation on which K9-AIF stands, and they are the foundation on which every serious agentic system should be built.

Booch said it thirty years ago. It is still true.

> "The entire history of software engineering is one of rising levels of abstraction."

Do not forget the fundamentals. They are what make this work.

Architecture first — built to last.

---
