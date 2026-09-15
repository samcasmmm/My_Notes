[🏠 Back to AI Index](./README.md) • [⬅️ Prev: LLMs & GenAI](./04-llms-transformers-genai.md) • [Next: AI Agents & Tool Calling ➡️](./06-autonomous-agents-tool-use.md)

<div align="center">
  <h1>05. Modern RAG & Vector Databases</h1>
  <p><b>Hybrid Search, HNSW Indexing, Cross-Encoder Reranking, GraphRAG & Vector Storage</b></p>
</div>

---

## 📑 Module Index
- [1. Evolution of RAG Architectures](#1-evolution-of-rag-architectures)
  - [Naive RAG vs Advanced Production RAG vs Modular RAG](#naive-vs-advanced-rag)
- [2. Ingestion & Advanced Chunking Strategies](#2-ingestion--advanced-chunking-strategies)
  - [Recursive Character vs Semantic vs Parent-Child (Hierarchical) Chunking](#chunking-strategies)
- [3. Vector Indexing & Search Mechanics](#3-vector-indexing--search-mechanics)
  - [Dense Vectors vs Sparse Lexical Vectors (BM25 / SPLADE)](#dense-vs-sparse)
  - [Approximate Nearest Neighbor (ANN): HNSW vs IVF-PQ](#ann-indexing-hnsw-vs-ivf)
- [4. Search Quality Enhancements](#4-search-quality-enhancements)
  - [Hybrid Search & Reciprocal Rank Fusion (RRF)](#hybrid-search--rrf)
  - [Two-Stage Retrieval with Cross-Encoder Rerankers](#cross-encoder-reranking)
  - [Query Transformation (HyDE & Multi-Query Expansion)](#query-transformations)
- [5. GraphRAG (Knowledge Graph Augmented RAG)](#5-graphrag-knowledge-graph-augmented-rag)
  - [Entity Extraction, Communities & Global Summarization](#graphrag-internals)
- [6. Vector Databases & PGVector Comparison](#6-vector-databases--pgvector-comparison)

---

## 1. Evolution of RAG Architectures

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ NAIVE RAG (2023):                                                                      │
│ User Query ──► Vector Embed ──► Top-K Cosine Search ──► Dump into Context ──► LLM     │
│ ❌ Failure Modes: Low precision, lost in the middle, bad chunks, hallucinations.       │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ ADVANCED PRODUCTION RAG (2026):                                                        │
│ User Query ──► Query Rewriter / HyDE                                                   │
│                     │                                                                  │
│                     ├──► Dense Vector Search (HNSW) ──┐                                │
│                     └──► Sparse BM25 Search ──────────┴──► Hybrid Fusion (RRF)         │
│                                                                   │                    │
│                                                        Cross-Encoder Reranker          │
│                                                                   │                    │
│                                                        Context Compression             │
│                                                                   │                    │
│                                                         LLM Synthesis with Citations   │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Ingestion & Advanced Chunking Strategies

Choosing the wrong chunk size destroys retrieval quality.

```
Small Chunks (128 tokens)     ──► High embedding accuracy, but loses surrounding context.
Large Chunks (1024 tokens)    ──► Dilutes specific facts; embedding vector becomes blurry.
```

### 1. Recursive Character Chunking
Splits by paragraphs (`\n\n`), then sentences (`\n`, `. `), preserving natural semantic boundaries with an overlap (e.g., 512 tokens with 50-token overlap).

### 2. Parent-Child (Hierarchical) Chunking
- Split documents into tiny child chunks (100 tokens) for **high-precision vector indexing**.
- When a child chunk is retrieved, return its larger parent section (800 tokens) to the LLM for rich context.

### 3. Semantic Chunking
Calculates cosine distance between consecutive sentences; splits dynamically only when the topic shift exceeds a threshold.

---

## 3. Vector Indexing & Search Mechanics

### Dense vs Sparse Embeddings

| Feature | Dense Embeddings (`text-embedding-3`, `bge-large`) | Sparse Lexical Search (BM25, SPLADE) |
| :--- | :--- | :--- |
| **Representation** | 768 to 3072 floating-point numbers | High-dimensional sparse dictionary of term weights |
| **Strength** | Semantic concepts, synonyms ("car" $\leftrightarrow$ "automobile") | Exact keyword matches, part numbers, SKUs, code tokens, IDs |
| **Weakness** | Can miss exact unique identifiers | Fails on conceptual paraphrasing |

---

### ANN Indexing: HNSW vs IVF-PQ

Exact nearest neighbor search ($O(N \cdot d)$) is too slow for 10M+ vectors. Databases use **Approximate Nearest Neighbor (ANN)** algorithms:

```
                  HNSW (Hierarchical Navigable Small World)
         Layer 2:  O ─────────────────────────────► O   (Express Highway)
                   │                                │
         Layer 1:  O ──────────► O ───────────────► O   (Local Roads)
                   │             │                  │
         Layer 0:  O ──► O ──► O ──► O ──► O ──► O ─► O (All Vectors)
```

1. **HNSW (Graph-based):** Builds a multi-layer skip-list graph. Fast search latency ($O(\log N)$) with state-of-the-art recall ($>98\%$). Standard in modern vector DBs.
2. **IVF-PQ (Inverted File with Product Quantization):** Clusters vector space into Voronoi cells and compresses vectors into byte-codes. Drastically lowers RAM usage at slight recall cost.

---

## 4. Search Quality Enhancements

### Hybrid Search & Reciprocal Rank Fusion (RRF)

Combines rankings from Dense Vector search and Sparse BM25 search without needing score calibration:

$$RRF\_Score(d) = \sum_{m \in \{\text{dense}, \text{sparse}\}} \frac{1}{k + \text{Rank}_m(d)}$$
*(where $k \approx 60$ is a smoothing constant)*.

```python
def reciprocal_rank_fusion(dense_ranks, bm25_ranks, k=60):
    scores = {}
    for rank, doc_id in enumerate(dense_ranks):
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)
    for rank, doc_id in enumerate(bm25_ranks):
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

---

### Cross-Encoder Reranking

Bi-Encoders encode query and document separately into vectors (fast, but shallow interaction). **Cross-Encoders** evaluate the Query and Document *together* through all self-attention layers:

```
Bi-Encoder (1st Stage: Fast Retrieval)       Cross-Encoder (2nd Stage: Precision Rerank)
  Query ──► [ Embed ] ──┐                       [ Query + Document ] ──► [ Full Transformer ] ──► Score (0.97)
                        ├──► Cosine Top 50                               (Evaluates top 20 candidates)
  Docs  ──► [ Embed ] ──┘
```

---

## 5. GraphRAG (Knowledge Graph Augmented RAG)

Standard RAG fails at **global dataset reasoning** (e.g., *"What are the top 5 emerging risks across all 500 company incident reports?"*).

```
Raw Documents ──► [ LLM Entity/Relation Extraction ] ──► [ Knowledge Graph ]
                                                                │
                                                                ▼
                                                [ Community Detection (Leiden) ]
                                                                │
                                                                ▼
                                                [ Pre-Generated Community Summaries ]
```

- Extracts Nodes (Entities) and Edges (Relationships) into a Knowledge Graph.
- Clusters graph into thematic communities and generates pre-computed hierarchical summaries.
- Allows both **Local queries** (specific facts) and **Global queries** (holistic themes).

---

## 6. Vector Databases & PGVector Comparison

```
┌─────────────────┬───────────────────┬────────────────────────────────────────────────────────┐
│ Database        │ Type              │ Best For                                               │
├─────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ **PGVector**    │ PostgreSQL Ext.   │ Default choice when you already use PostgreSQL.        │
│ **Qdrant**      │ Standalone (Rust) │ Production payload filtering, hybrid search, fast HNSW.│
│ **Pinecone**    │ Managed Serverless│ Zero infrastructure overhead, turnkey enterprise scale.│
│ **Milvus**      │ Distributed       │ Massive scale (100M to Billions of vectors).           │
│ **ChromaDB**    │ Lightweight       │ Local development, prototyping, embedded Python apps.  │
└─────────────────┴───────────────────┴────────────────────────────────────────────────────────┘
```

#### PGVector SQL Example:
```sql
-- Enable extension & create vector table
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536) -- OpenAI text-embedding-3-small dimension
);

-- Create HNSW Index for ultra-fast cosine search
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops);

-- Query top 5 most relevant chunks
SELECT id, content, 1 - (embedding <=> $1) AS cosine_similarity
FROM document_chunks
ORDER BY embedding <=> $1
LIMIT 5;
```

---

[➡️ Continue to Module 06: Autonomous AI Agents & Tool Calling](./06-autonomous-agents-tool-use.md)
