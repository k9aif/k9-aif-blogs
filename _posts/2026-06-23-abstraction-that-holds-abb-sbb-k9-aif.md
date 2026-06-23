---
layout: post
title: "Abstraction That Holds — How ABB/SBB Keeps K9-AIF Extensible Without Breaking"
date: 2026-06-23
author: Ravi Natarajan
---

> "The entire history of software engineering is one of rising levels of abstraction. This is as it was, is now, and always shall be." — Grady Booch

I've carried that quote with me for twenty years. But I didn't fully appreciate what it means for agentic AI until today.

---

## What Happened Today

This morning, K9-AIF had no real vector database integration in its retrieval path. The `K9Retriever` — the framework's out-of-the-box retriever — was a stub. The Squad execution engine passed all accumulated context to every agent in the flow, whether that agent needed it or not. Two embedding agents had the same Ollama embedding code copied and pasted into each.

By afternoon, I had:

- A new `BaseEmbeddingService` ABB contract
- An `OllamaEmbeddingService` OOB implementation
- An `EmbeddingServiceFactory` — config-driven, same pattern as every other factory in the framework
- `K9Retriever` wired to VectorDB + EmbeddingService for real semantic search
- `context_keys` in the Squad flow — agents now declare which upstream results they need, instead of receiving everything
- Both RAG agents refactored to use the new factory instead of duplicated code

Six files modified, four files created. Fourteen new tests added.

And the existing 219 tests didn't blink. Not one failure.

That's what abstraction buys you — when you do it right.

---

## The ABB/SBB Model

K9-AIF's architecture is built on a separation that comes directly from TOGAF:

**Architecture Building Blocks (ABB)** — abstract contracts. They define the interface, the lifecycle, the governance hooks. They never contain domain logic. They live in `k9_aif_abb/k9_core/`.

**Solution Building Blocks (SBB)** — concrete implementations. They extend ABBs with real behavior — domain-specific, provider-specific, organization-specific. They live in `k9_data/`, `k9_agents/`, `k9_factories/`, or in the solution's own directory under `examples/` or `k9_projects/`.

The framework ships OOB (out-of-the-box) SBBs — ready-to-run defaults that work with zero configuration. You don't have to write an embedding service to use vector retrieval. But when your organization needs a different provider, you extend the ABB, register your SBB, and the framework doesn't change.

<a href="../assets/images/blogs/k9-aif-abb-sbb-swimlane.png" target="_blank">
  <img src="../assets/images/blogs/k9-aif-abb-sbb-swimlane.png"
       alt="K9-AIF ABB/SBB Swim Lane"
       style="width:100%;max-width:960px;cursor:zoom-in;">
</a>

---

## The Swim Lane — ABB on Top, SBB Below

Here's how the layers interact. The ABB defines the contract. The SBB fulfills it. The factory wires them together from config. The solution code never touches the ABB — it extends the SBB or uses the factory.

```
┌─────────────────────────────────────────────────────────────────────┐
│  ABB Layer (Abstract Contracts — k9_core/)                         │
│                                                                     │
│  BaseEmbeddingService    BaseVectorDB    BaseRetriever    BaseAgent │
│       │                       │               │               │     │
│       │  embed()              │  search()     │  retrieve()   │     │
│       │  embed_batch()        │  insert()     │               │     │
│       │                       │  delete()     │               │     │
├───────┼───────────────────────┼───────────────┼───────────────┼─────┤
│       │     Factory Layer     │               │               │     │
│       │                       │               │               │     │
│  EmbeddingService        VectorDB       Retriever        Agent      │
│    Factory                Factory        Factory        Registry    │
│       │                       │               │               │     │
│       │   config.yaml         │               │               │     │
│       │   drives selection    │               │               │     │
├───────┼───────────────────────┼───────────────┼───────────────┼─────┤
│  SBB Layer (Concrete Implementations)                               │
│                                                                     │
│  OllamaEmbedding     ChromaDBAdapter   K9Retriever    FraudAgent   │
│  Service (OOB)       (OOB default)     (OOB)         (Domain SBB)  │
│                                                                     │
│  [Your custom        MilvusAdapter     [Your custom   ClaimsAgent  │
│   embedding SBB]     QdrantAdapter      retriever]    (Domain SBB)  │
│                      PgVectorAdapter                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The key: the arrows only go **down**. ABBs don't know which SBB will fulfill them. Factories don't know which provider is configured until runtime. SBBs extend upward — but never modify the layer above.

---

## Why This Matters Across Projects, Business Units, and Enterprises

When three teams in three business units build agentic applications on K9-AIF, they share the same ABB contracts. Their agents implement the same `execute(payload) -> dict`. Their squads run the same flow engine. Their governance follows the same pipeline.

But their SBBs are entirely their own.

| What stays the same (ABB) | What varies (SBB) |
|---|---|
| `BaseAgent.execute()` contract | Domain logic inside `execute()` |
| `BaseVectorDB.search()` interface | ChromaDB, Milvus, Qdrant, PgVector |
| `BaseEmbeddingService.embed()` interface | Ollama, OpenAI, Watsonx |
| Squad flow engine + `context_keys` | Which agents, what flow, what scoping |
| Governance pipeline hooks | What gets checked, what gets blocked |
| `K9ModelRouter` scoring | Custom routers with different scoring signals |

This is what OOA and OOD give you. Not class hierarchies for the sake of class hierarchies — but contracts that hold under change. Every feature I add to K9-AIF follows the same discipline:

1. Define the ABB contract (what, not how)
2. Ship an OOB SBB (works out of the box, zero config)
3. Wire it through a factory (config-driven provider selection)
4. Make it additive (existing code doesn't change)

---

## Today's Example — Concrete

Here's what `BaseEmbeddingService` looks like. This is the entire ABB:

```python
class BaseEmbeddingService(ABC):

    def __init__(self, config=None):
        self.config = config or {}

    @abstractmethod
    def embed(self, text: str) -> List[float]:
        raise NotImplementedError

    def embed_batch(self, texts: List[str]) -> List[List[float]]:
        return [self.embed(t) for t in texts]
```

That's it. Twelve lines. It will never change. Every embedding provider — Ollama, OpenAI, Watsonx, a custom on-premise model — implements those two methods and registers with the factory.

The OOB SBB for Ollama:

```python
class OllamaEmbeddingService(BaseEmbeddingService):

    def __init__(self, config=None):
        super().__init__(config)
        self._client = None
        vdb_cfg = self.config.get("vectordb", {})
        self._model = vdb_cfg.get("embedding_model", "nomic-embed-text")
        self._host = vdb_cfg.get("embedding_endpoint", "http://localhost:11434")

    def _ensure_client(self):
        if self._client is not None:
            return
        from ollama import Client
        self._client = Client(host=self._host)

    def embed(self, text: str) -> List[float]:
        self._ensure_client()
        result = self._client.embeddings(model=self._model, prompt=text)
        return result.get("embedding", [])
```

An enterprise team that uses Watsonx writes their own `WatsonxEmbeddingService`, registers it with the factory, sets `vectordb.embedding_provider: watsonx` in their config.yaml — and the framework, the squad flow, the agents, the retriever — none of them know or care.

---

## What Other Frameworks Get Wrong

Most agentic frameworks — LangChain, CrewAI, AutoGen — start from the bottom up. They build concrete integrations first and extract abstractions later, if ever. The result: provider assumptions leak into the framework. Swapping a vector DB means rewriting agent code. Changing the LLM provider means touching the orchestration layer.

K9-AIF starts from the top down. The ABB exists before the SBB. The contract exists before the implementation. The factory exists before the provider. That's not academic discipline for its own sake — it's the reason today's work touched zero lines in the agent, orchestrator, or router layers.

Booch wasn't just making a philosophical observation. He was describing the mechanism by which software systems survive change. Abstraction isn't a style preference — it's a survival strategy.

---

## The Discipline

Every capability in K9-AIF follows the same pattern:

```
k9_core/<concern>/base_<concern>.py       ← ABB contract (abstract)
k9_<concern>/<impl>/<name>_service.py     ← SBB implementation (concrete)
k9_factories/<concern>_factory.py         ← Config-driven wiring
config.yaml                               ← Provider selection
```

This pattern now covers:

| Concern | ABB | OOB SBB | Factory |
|---|---|---|---|
| Inference | `BaseLLM` | `OllamaLLM`, `OpenAILLM` | `LLMFactory` |
| Model Routing | `BaseModelRouter` | `K9ModelRouter` | `ModelRouterFactory` |
| Embedding | `BaseEmbeddingService` | `OllamaEmbeddingService` | `EmbeddingServiceFactory` |
| Vector Store | `BaseVectorDB` | `ChromaDBAdapter` | `VectorDBFactory` |
| Retrieval | `BaseRetriever` | `K9Retriever` | `RetrieverFactory` |
| Cache | `BaseCache` | `InMemoryAdapter` | `CacheFactory` |
| Secret Management | `BaseSecretManager` | `EnvSecretAdapter` | `SecretManagerFactory` |
| Agent | `BaseAgent` | `K9ValidationLoopAgent` | `AgentRegistry` |
| Governance | `BaseGovernance` | `NoopGovernance` | `require_governance()` |

Nine concerns. Same pattern. Same discipline. And when the tenth comes — it will follow the same structure, and it will not break the other nine.

---

## Closing

Abstraction is not about hiding complexity. It's about drawing the right lines — the lines that hold when requirements change, when providers change, when teams change, when the enterprise scales.

Booch said it. K9-AIF encodes it.

Architecture first — built to last.

---
