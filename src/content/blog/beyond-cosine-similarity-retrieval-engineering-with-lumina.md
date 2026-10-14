---
title: "Beyond Cosine Similarity"
description: "Retrieval Engineering with Luminary: Discover how to avoid the prototype trap and build robust, local-first AI apps with Luminary."
pubDate: "Oct 14 2026"
project: "Luminary"
series: "Luminary Chronicles"
tags: ["Luminary", "Learning", "Retrieval", "LoFi", "Evals", "Agentic", "Hybrid-RAG"]
---

# Beyond Cosine Similarity:  Retrieval Engineering with Luminary

---

Late last year, I stood on stage at a meetup in Brisbane talking about how to build a simple AI assistant to converse with your books. Since then, the AI landscape has evolved tremendously. Models have gotten so ridiculously smart that some days I feel like I barely even program anymore!

Let’s be honest: there’s never been a better time to materialize your ideas into product. But alongside that excitement, there’s also a level of anxiety. 
How many times have you felt that AI adoption is creating a quiet anxiety within you? A feeling that we’re letting AI take over control of how we learn, solve problems, and make decisions?

That exact feeling is what propelled me to build **Luminary**—an open-source, local-first learning app. Instead of outsourcing our thinking to the cloud, Luminary uses AI to exercise **human agency** through deliberate learning. It runs 100% offline on your laptop—using embedded databases and local models via Ollama—so your personal data never leaves your machine.

In my quest to build Luminary, I spent six months diving deep into advanced retrieval techniques. I burned my fingers, wasted time, and had my fair share of failures and wins. Today, I want to share those learnings with you and show you what it actually takes to move the needle for production-grade AI apps.

---

## The Prototype Trap

Imagine this scenario: your manager calls you in and asks you to build an internal chat assistant over your company’s documentation. 

You grab the opportunity, fire up your favorite coding assistant, and build a RAG (Retrieval-Augmented Generation) prototype within a week. You run a few sample PDFs through it, ask a couple of queries, and the answers look amazing. You feel like a wizard, don't you? You showcase it to stakeholders, everyone is thrilled, and you get the green light to productionize it.

Now it's a whole different ball game. 

Suppose you work in healthcare or enterprise HR. You ingest thousands of complex radiology safety procedure manuals or internal parental leave policies, and you start testing real user queries. Suddenly, the answers come back flimsy, nonsensical, or completely wrong. You start wondering: *Aren't RAG systems supposed to stop hallucinations? Should I switch to a bigger LLM? Is my prompt engineering wrong?*

Here is the key realization: **The problem is not with comprehension; it is with the context!**

Prompting won't save you if you feed the LLM garbage context. Retrieval is not a novel "AI problem"—it is an **age-old optimization problem** of maximizing relevance subject to a strict latency and compute budget.

---

## The Relevance Triangle & System Trade-Offs

As Martin Kleppmann highlights in *Designing Data-Intensive Applications*, every system design decision comes down to trade-offs. In retrieval engineering, this is defined by the **Relevance Triangle**:

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset1.png" alt="Pasted Image" style="display:block;max-width:800px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

You have to define what you are after:
* **Recall vs. Precision**: Do you want to find *all* relevant documents across your corpus (Recall), or do you only want exact, pure hits (Precision)?
* **Exactness vs. Meaning**: Are you searching for exact part numbers, error codes, and acronyms (Tokens/Lexical), or are you searching for intent and underlying concepts (Semantic/Vector)?
* **Latency vs. Quality**: Are you willing to pay a heavy compute and time penalty per query to get pinpoint accuracy?

Because no single search engine solves all three corners, production retrieval requires a **three-legged hybrid search** pipeline.

---

## The Three Legs of Search

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset2.png" alt="Pasted Image" style="display:block;max-width:800px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

### Leg 1: Lexical Search (The Card Catalog Expert)
Lexical search has been the backbone of information retrieval for over 50 years. At its core lies an **inverted index**—just like the index at the back of a textbook mapping words directly to page numbers.

The modern workhorse here is **BM25 (Best Matching 25)**, which improves on standard TF-IDF by adding two essential knobs:

* **Term Saturation ($k_1 \approx 1.2$)**: $k_1$ acts like **salt in soup**. The first pinch transforms the flavor, but the 10th pinch barely registers, and the 50th does nothing. If a keyword like "data engineering" appears 50 times in a doc, $k_1$ caps its influence so keyword stuffing doesn't dominate rankings.
* **Length Normalization ($b \approx 0.75$)**: $b$ grades documents fairly regardless of word count. Four keyword mentions in a 1-page memo mean much more than four mentions buried in a 100-page manual.

> **Why Lexical is Indispensable**: Embedding models notoriously fumble exact tokens. If a user searches for an exact error code like `02dkkdf`, an acronym, or specific jargon, BM25 lands directly on the right document where vector search gets confused.
> 
> **Local Implementation in Luminary**: Luminary implements its lexical leg locally using **SQLite FTS5**—a zero-dependency full-text engine running right on your laptop.

---

### Leg 2: Vector Search (The Subject Matter Expert)
Vector search retrieves text by **meaning**. An embedding model (bi-encoder) assigns every chunk of text a coordinate ("street address") inside a high-dimensional *Semantic City*. Phrases like *"password recovery"* and *"lost credentials"* share zero words, yet sit on the exact same block in vector space.

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset3.png" alt="Pasted Image" style="display:block;max-width:480px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

#### Scaling Nearest Neighbors with HNSW
If your corpus is huge, doing an exact brute-force vector comparison requires a linear scan ($O(N)$), which collapses at scale. We use **Approximate Nearest Neighbor (ANN)** indexes like **HNSW (Hierarchical Navigable Small World)**:

* **The International Flight Analogy**: HNSW works like long-distance air travel. The top sparse layers represent long-haul flights between hub cities (e.g., LA to Sydney)—making massive geographic hops. The search then descends into lower, denser layers for local street navigation to find the exact address.
* This converts an $O(N)$ brute-force scan into logarithmic $O(\log N)$ traversal.

> **Local Implementation in Luminary**: Luminary uses a local **BAAI/bge-m3** model (running via ONNX) to embed text into 384-dimensional vectors, storing them in **LanceDB**—a file-based vector database that lives right under your `.luminary/` folder.

---

### Leg 3: Graph Search (The Detective's Corkboard)
Standard vector search fails when an answer cannot be found inside a single chunk of text. For multi-hop questions like *"How is Entity A connected to Entity D across these documents?"*, we turn to **Knowledge Graphs**.

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset4.png" alt="Pasted Image" style="display:block;max-width:800px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

Think of a knowledge graph as a **detective’s corkboard** connecting the dots between pieces of a mystery. It extracts **entities** (nodes) and **relationships** (edges) from text, allowing traversal algorithms to follow string connections across pages.

> **Local Implementation in Luminary**: Luminary uses **GLiNER**—a zero-shot Named Entity Recognition model that requires no pre-defined taxonomies—to extract entities locally. It stores relationships in **Kuzu** (an embedded graph database) and visualizes them interactively with Sigma.js.

---

## The Fusion Problem: Reciprocal Rank Fusion (RRF)

Now you have three search legs running in parallel. But here is the catch: **How do you merge their scores?**

BM25 outputs unbounded positive numbers, Cosine Similarity yields decimals between $-1$ and $1$, and Graph algorithms return traversal steps. **You cannot simply add these raw scores together!**

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset5.png" alt="Pasted Image" style="display:block;max-width:480px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

The industry standard solution is **Reciprocal Rank Fusion (RRF)**:

$$\text{RRF Score}(d \in D) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$

Where $r_m(d)$ is document $d$'s rank in search leg $m$, and $k$ is a smoothing constant (typically $k=60$).

### The Olympic Judging Analogy
RRF acts like **Olympic judging by rank placement rather than raw points**. One judge rates out of 10, another out of 100—raw scores can't be added, but $1^\text{st}, 2^\text{nd}, 3^\text{rd}$ place ranks are always comparable.

With $k=60$, rank 1 gives $\frac{1}{61} \approx 0.0164$ and rank 2 gives $\frac{1}{62} \approx 0.0161$. Because the score drop-off is gentle, **a document ranked $3^\text{rd}$ across all three search legs will outscore a document that ranked $1^\text{st}$ in one leg but was absent in the others**. RRF systematically prioritizes consensus and eliminates "lone champions."

---

## The Production Funnel Architecture ($L1 \rightarrow L2 \rightarrow L3$)

To achieve high recall without blowing your latency and compute budget, the gold standard architecture used by Google, Meta, and Vespa is the **Phased Ranking Funnel**:

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset6.png" alt="Pasted Image" style="display:block;max-width:480px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

### L1: Candidate Generation (High Recall, Cheap Compute)
* **Goal**: Cast a massive net across millions of documents to gather 100–1,000 candidates.
* **Mechanism**: Hybrid search (BM25 + Bi-Encoders + Graph) fused via RRF.
* **Rule**: L1's only job is **not to lose the winning document**.

### L2: Reranking & Refinement (High Precision, Heavy Compute)
* **Goal**: Deeply scrutinize the top 50–100 survivors from L1 to order them with pinpoint precision.
* **Mechanism**: **Cross-Encoder Models**.

A Bi-Encoder is like **screening resumes**: profiles are precomputed offline and matched fast. A Cross-Encoder is the **in-person job interview**: it concatenates the query and the document and passes them through the transformer *together*, allowing **Full Joint Attention**.

A Cross-Encoder grasps **negation, coreference, and logical nuance**. For example, a Bi-Encoder views *"aspirin causes stomach ulcers"* and *"aspirin does NOT cause stomach ulcers"* as near-identical vectors. A Cross-Encoder reads the joint context, catches the negation, and correctly penalizes the wrong claim. Because Cross-Encoders require a full forward pass per document, they are computationally heavy—making them ideal for the top 50 survivors of L1.

### L3: Business Logic & Slate Optimization
* **Goal**: Apply domain rules to shape the final user slate.
* **Mechanism**: Maximal Marginal Relevance (MMR) for result diversity, recency boosts, deduplication, and watch/permission filtering.

---

### Real-World Case Study: Streaming Catalog Search at Foxtel
Since I work at Foxtel Group, think of a streaming catalog when a user queries: ***"Not so popular Christopher Nolan movies"***:

1. **L1 (Candidate Generation)**: Does a cheap, broad sweep across catalog genres, tags, and metadata. It identifies "Nolan" $\rightarrow$ Christopher Nolan, pulls his filmography, and gathers hundreds of candidate titles.
2. **L2 (Cross-Encoder Reranking)**: Analyzes the joint attention of the query. It catches the negation **"not so popular"**—demoting massive blockbusters like *Inception*, *Interstellar*, and *The Dark Knight*, while elevating lesser-known critical gems like *Insomnia* or *Following*.
3. **L3 (Business Logic)**: Applies **Maximal Marginal Relevance (MMR)** so the UI doesn't recommend *Batman Begins*, *The Dark Knight*, and *The Dark Knight Rises* all in a row. It filters out movies the user has already watched, applies recency boosts, and passes the clean slate to the LLM.

---

## "Retrieval Fails Silently": Evals and Observability

In standard software engineering, a failing pipeline throws a loud stack trace. In retrieval engineering, **systems fail silently**—returning fluent, highly confident, but completely wrong context.

Just as you test data pipelines with TDD, you must evaluate RAG with a **Golden Q&A Dataset** using three core metrics across the funnel:

| Metric | What It Measures |
| :--- | :--- |
| **Hit Rate @ k (Recall @ k)** | **L1 Performance**: Did the correct document survive candidate generation? |
| **Mean Reciprocal Rank (MRR)** | **L2 Performance**: Was the best document ranked near position #1? |
| **Faithfulness / Grounding** | **Generation Performance**: Is the LLM output strictly backed by context? |

In Luminary, I built a local RAGAS evaluation harness targeting strict thresholds: **Hit Rate@5 $\ge 0.60$, MRR $\ge 0.45$, and Faithfulness $\ge 0.65$**. 

When quality drops, your evaluation metrics pinpoint the exact failure stage:
* If **Hit Rate @ 5** is low $\rightarrow$ Fix L1 chunking, parsing, or retrieval legs.
* If **Hit Rate @ 5** is high but **MRR** is low $\rightarrow$ Add or tune an L2 Cross-Encoder reranker.
* If **MRR** is high but the answer is wrong $\rightarrow$ Fix prompt instructions or generation faithfulness.

---

## Agentic Memory: Retrieval Turned Inward

If retrieval is finding knowledge across external documents, **Agentic Memory** is retrieval turned inward—persisting and recalling an agent's own history across context window boundaries.

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset7.png" alt="Pasted Image" style="display:block;max-width:480px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

AI Agent memory parallels human cognitive taxonomy:
1. **Working Memory**: The immediate context window (system prompt, recent conversation turns, scratchpad).
2. **Episodic Memory**: Time-stamped records of specific past user interactions.
3. **Semantic Memory**: Consolidated, timeless facts extracted from past episodes (e.g., *"User prefers TypeScript; deploys on Fridays are forbidden"*).
4. **Procedural Memory**: System instructions, tool rules, and executable skill files (`SKILL.md`).

### The Mathematical Symmetry of Memory Decay
In Luminary, human memory retention is managed using the **FSRS (Free Spaced Repetition Scheduler)** algorithm. FSRS calculates stability and retrievability curves to schedule flashcard reviews right before a user forgets.


Fascinatingly, **the mathematical equations governing human forgetting curves match the Time-To-Live (TTL) and recency decay formulas used in AI agent memory stores**. Whether memory lives in human neurons or an AI's vector store, managing context requires balancing decay, recency, and relevance.

---

## Architectural Mapping: Laptop to Enterprise Cloud

The central takeaway for data engineers is that **retrieval architecture is universal**. The exact multi-leg funnel that runs locally on a laptop inside Luminary maps directly to enterprise cloud infrastructure:

<img src="/blog/beyond-cosine-similarity-retrieval-engineering-with-lumina/asset8.png" alt="Pasted Image" style="display:block;max-width:800px;width:100%;margin:1.5rem auto;border-radius:0.5rem;" />

| Funnel Layer | Luminary Local-First Implementation | Enterprise Cloud Equivalent |
| :--- | :--- | :--- |
| **Lexical (BM25)** | SQLite FTS5 | Elasticsearch / OpenSearch / Vespa |
| **Vector Store** | LanceDB (`bge-m3` ONNX) | Qdrant / Pinecone / Weaviate |
| **Knowledge Graph** | Kuzu Graph DB (GLiNER) | Neo4j / Memgraph |
| **Fusion** | Reciprocal Rank Fusion (RRF) | Azure AI Search RRF / Elastic Fusion |
| **L2 Reranking** | Local Cross-Encoder (`bge-reranker`) | Cohere Rerank / Voyage Rerank |
| **Orchestration** | LangGraph + LiteLLM | LangGraph / LlamaIndex |

---

## Conclusion

The barrier to entry has completely vanished. Everything we discussed today—a multi-leg hybrid search with lexical, semantic, and graph retrieval, fused with RRF, reranked with cross-encoders, and evaluated with RAGAS—can fit on a standard laptop without a single cloud server. Start with a solid BM25 lexical baseline, add vector search, measure with an evaluation harness, and build your funnel sequentially. Go build your own thing. 

*Unplug and run Luminary locally without distractions; Exercise your own creative mental activity !!*
www.github.com/nupsea/luminary
