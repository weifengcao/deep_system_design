# Vector Databases for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Embeddings, ANN algorithms (HNSW, IVF, PQ), major vector DBs, RAG architecture, hybrid search, security, operational considerations

---

## Why Vector Databases

Traditional databases find records by exact matches or range conditions on structured fields. They can't answer: **"Find documents similar in meaning to this query."**

Vector databases enable **semantic search** — finding similarity in a high-dimensional embedding space. This is the infrastructure layer beneath:
- RAG (Retrieval-Augmented Generation) for LLMs
- Semantic document search
- Image/audio similarity search
- Recommendation systems (embedding-based)
- Anomaly detection
- Duplicate detection

---

## Embeddings: The Foundation

An **embedding** is a dense, fixed-size vector of floats that encodes semantic meaning. Semantically similar inputs map to nearby vectors.

```
text-embedding-3-small produces 1536-dimensional vectors:
  "dog"     → [0.23, -0.45, 0.12, ..., 0.67]  (1536 floats)
  "puppy"   → [0.25, -0.43, 0.14, ..., 0.65]  ← close! (similar concept)
  "database"→ [0.89,  0.12, -0.34, ..., -0.12] ← far away

Distance metrics:
  Cosine similarity: 1 - (A·B)/(|A||B|)  → most common for text
  L2 (Euclidean) distance: sqrt(Σ(aᵢ-bᵢ)²) → common for images
  Dot product: A·B → equivalent to cosine for normalized vectors
```

**Embedding models:**
- **Text:** OpenAI text-embedding-3-small (1536d, $0.02/1M tokens), Cohere Embed v3 (1024d), open-source BGE, E5
- **Images:** CLIP (512d), DINO, ImageBind
- **Code:** CodeBERT, StarCoder embeddings
- **Multimodal:** CLIP (handles both text and images in same space)

---

## The Core Problem: ANN Search

Finding the exact nearest neighbor in 1536 dimensions across 100 million vectors requires comparing the query against all 100M vectors — O(N) time, ~1 second at this scale.

**Approximate Nearest Neighbor (ANN):** Trade a small accuracy loss for 100–1000× speedup. Return the K most similar vectors with >95% recall.

### HNSW (Hierarchical Navigable Small World)

The dominant ANN algorithm for high-recall, in-memory workloads. Used by Pinecone, Weaviate, pgvector, Redis, Qdrant.

**Intuition:** Build a multi-layer graph. Higher layers are a "highway" for long jumps; lower layers provide local precision.

```
Layer 2 (sparse): A ──────────── F ──────────── K
                    \                           /
Layer 1 (medium): A ─── C ─── F ─── H ─── K ─── N
                    \   |   /   \   |   /
Layer 0 (dense):  A─B─C─D─E─F─G─H─I─J─K─L─M─N

Search for query Q:
  Start at random node in top layer
  Greedily traverse toward Q using nearest neighbors at each layer
  Drop to next layer when no closer node found
  At layer 0: explore local neighborhood → return top-K results
```

**HNSW parameters:**
- `M`: Number of edges per node (default 16). Higher = better recall, more memory, slower build
- `ef_construction`: Build-time search width (default 200). Higher = better graph quality, slower build
- `ef_search` (query-time): Higher = better recall, slower query. Set at query time, not index time.

**Performance:** O(log N) average search time. Handles 100M vectors on a single node with ~50GB RAM.

### IVF (Inverted File Index)

Clusters vectors into Voronoi cells using k-means. Query: assign query to nearest cluster(s), search within those clusters only.

```
Build time:
  k-means cluster all vectors into K centroids
  Each vector assigned to nearest centroid

Query time:
  Find top nprobe nearest centroids to query
  Search all vectors in those clusters
  Return top-K results
```

**IVF parameters:**
- `nlist`: Number of clusters (√N is a good starting point for N vectors)
- `nprobe`: Number of clusters to search at query time (trade recall vs speed)

**Advantage over HNSW:** Lower memory (on-disk friendly), good for datasets > RAM. Works well with PQ compression.

### PQ (Product Quantization)

Compression technique — reduces vector size 8–32× at the cost of some recall.

```
Original: 1536-dim float32 vector = 6,144 bytes
PQ: Split into 96 sub-vectors of 16 dims each
    Quantize each to one of 256 centroids (1 byte)
    Compressed: 96 bytes (64× compression!)
```

Combined as IVF-PQ: the standard for billion-scale datasets where vectors can't all fit in RAM.

### ScaNN (Google's Algorithm)

Optimized for high-throughput in-memory ANN. Used internally at Google. Available as open-source. Performs well on Google's benchmarks but less widely deployed than HNSW.

---

## Major Vector Database Products

### Pinecone
**Fully managed, serverless vector database.** No ops, auto-scales.

```python
import pinecone

pc = pinecone.Pinecone(api_key="...")
index = pc.Index("documents")

# Upsert vectors
index.upsert([
    ("doc-1", [0.23, -0.45, ...], {"source": "wiki", "date": "2025-01"}),
    ("doc-2", [0.89,  0.12, ...], {"source": "blog"})
])

# Query
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={"source": "wiki"},  # pre-filter metadata
    include_metadata=True
)
```

**Strengths:** Zero ops, excellent performance, metadata filtering  
**Weaknesses:** Proprietary/costly at scale, data leaves your infrastructure  
**Use for:** Rapid prototyping, teams without ML infra expertise

### Weaviate
**Open-source vector database with hybrid search and generative AI modules.**

```python
client = weaviate.Client("http://localhost:8080")

# Hybrid search: combine BM25 + vector search
result = client.query.get("Document", ["content", "source"]) \
    .with_hybrid(query="machine learning", alpha=0.5) \
    .with_limit(10) \
    .do()

# alpha=0: pure BM25; alpha=1: pure vector; alpha=0.5: equal blend
```

**Strengths:** Hybrid search built-in, GraphQL API, multi-modal, active community  
**Weaknesses:** Resource-heavy, complex config  
**Use for:** Production RAG with hybrid search, open-source requirement

### Qdrant
**Open-source, Rust-based, high-performance vector DB.**

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

client = QdrantClient("localhost", port=6333)

client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
)

client.upsert("documents", points=[
    PointStruct(id=1, vector=[0.23, -0.45, ...], payload={"source": "wiki"})
])

results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    query_filter={"must": [{"key": "source", "match": {"value": "wiki"}}]},
    limit=10
)
```

**Strengths:** Excellent performance, rich filtering, sparse+dense hybrid, on-disk indexing, payload indexing  
**Weaknesses:** Newer ecosystem  
**Use for:** High-performance production deployments, self-hosted

### pgvector (PostgreSQL Extension)
Keep vectors in Postgres alongside relational data:

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding vector(1536)
);

CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Semantic search with metadata filter
SELECT id, content, embedding <=> $1 AS distance
FROM documents
WHERE metadata->>'source' = 'wiki'
ORDER BY embedding <=> $1
LIMIT 10;
```

**Strengths:** No new infrastructure, SQL joins, ACID, familiar ops  
**Weaknesses:** Not as fast as purpose-built at > 10M vectors, HNSW index fully in-memory  
**Use for:** < 5M vectors, team already on Postgres, need joins with relational data

### Chroma
Developer-friendly, local-first, Python-native. Excellent for prototyping and local RAG:

```python
import chromadb
client = chromadb.Client()
collection = client.create_collection("docs")
collection.add(documents=["doc text..."], ids=["1"], embeddings=[[0.1, ...]])
results = collection.query(query_embeddings=[query_vec], n_results=5)
```

**Use for:** Local dev, notebooks, prototyping — not production at scale.

---

## RAG Architecture

Retrieval-Augmented Generation is the primary production use case for vector databases:

```
Build Phase (offline):
  Documents → Chunking → Embedding model → Vector DB
  (chunk: split doc into 256–512 token segments with overlap)

Query Phase (online):
  User query → Embedding model → query vector
              → Vector DB: ANN search → top-K chunks
              → Reranker: cross-encoder reranks top-K → top-N
              → LLM: [system prompt] + [top-N chunks] + [user query] → answer
```

### Chunking Strategy

```
Document: "Redis is an in-memory data structure store...
           [1000 more words]"

Fixed-size chunking:
  chunk1 = first 512 tokens
  chunk2 = next 512 tokens (with 50-token overlap for continuity)

Semantic chunking:
  Split at natural paragraph/section boundaries
  Better context preservation but variable chunk sizes

Hierarchical chunking:
  Small chunks for retrieval precision
  Parent chunks for LLM context
```

### Hybrid Search (Critical for Production)

Pure vector search misses exact keyword matches. Pure keyword search misses semantic similarity. Production RAG uses both:

```python
# 1. Sparse retrieval (BM25 / keyword)
bm25_results = bm25_index.search(query, top_k=50)

# 2. Dense retrieval (vector)
vector_results = vector_db.search(query_embedding, top_k=50)

# 3. Combine with Reciprocal Rank Fusion (RRF)
combined = rrf_merge(bm25_results, vector_results)

# 4. Rerank with cross-encoder (most accurate but expensive)
reranked = cross_encoder.rerank(query, combined[:20])

# 5. Return top-5 to LLM
context = reranked[:5]
```

**RRF formula:** `score(d) = Σ 1/(k + rank(d))` where k=60 typically.

### Reranking

After ANN retrieval, a **cross-encoder reranker** (e.g., Cohere Rerank, BGE-reranker) scores each query-document pair jointly — far more accurate than embedding dot product but O(N) vs O(log N). Use on the top-50 candidates to produce the final top-5.

---

## Operational Considerations

### Index Freshness
ANN indexes (HNSW, IVF) are expensive to update. Strategies:
- **Immutable segments:** Append new vectors to a new segment; merge periodically (like LSM-Tree)
- **Incremental HNSW:** Insert incrementally (supported by most modern vector DBs)
- **Rebuild on schedule:** For slowly changing data, rebuild nightly

### Sharding at Scale
For > 100M vectors:
```
Strategy: shard by metadata (e.g., by tenant_id or document_category)
Each shard is a separate HNSW index on a different node
Query fan-out: search all shards, merge results
```

### Memory Estimation
```
HNSW index memory:
  Each vector: dims × 4 bytes (float32)
  HNSW overhead: ~M × 8 bytes per node (graph edges)
  
  1M vectors × 1536 dims × 4 bytes = 6 GB (raw vectors)
  1M vectors × 16 edges × 8 bytes = 0.13 GB (graph)
  Total: ~6.1 GB per million 1536-dim vectors

100M vectors → 610 GB RAM (needs multiple nodes or on-disk index)
```

### Security
- **Namespace isolation:** Multi-tenant systems must enforce namespace-level access control. Never allow cross-tenant vector retrieval.
- **Embedding inversion:** Embeddings can partially leak the original content. Treat them as sensitive data.
- **Prompt injection via retrieval:** Malicious documents embedded in the vector DB can inject instructions. Sanitize retrieved content before inserting into LLM context.

---

## Choosing a Vector Database

```
Criteria                    → Recommendation
─────────────────────────────────────────────────────────────────
Already on Postgres          → pgvector (simple, no new infra)
< 5M vectors                 → pgvector or Chroma (dev), Qdrant (prod)
Need hybrid search           → Weaviate or Qdrant
Fully managed, AWS           → Pinecone or OpenSearch with k-NN
High perf, self-hosted       → Qdrant
Multi-modal (image + text)   → Weaviate
Billion-scale                → Milvus, Qdrant with on-disk IVF-PQ
```

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| What is an embedding? | Dense vector encoding semantic meaning; similar inputs → nearby vectors |
| Why not use Postgres for vector search? | No ANN index (full scan = O(N)); pgvector adds HNSW but not as scalable |
| What is ANN? | Approximate Nearest Neighbor — trades recall% for 100–1000× speedup |
| What is HNSW? | Hierarchical navigable small world graph — O(log N) ANN, high recall, in-memory |
| What is RAG? | Retrieve relevant chunks from vector DB, inject into LLM context |
| How to handle keyword + semantic? | Hybrid search: BM25 + vector, merged via RRF, reranked by cross-encoder |
| How to prevent prompt injection via RAG? | Sanitize retrieved chunks, separate system/user/retrieved content in prompt |
| How to scale beyond RAM? | IVF-PQ (on-disk quantized index), sharding by metadata across nodes |

---

*Previous: [11 - Data Structures for Big Data](./11-data-structures-big-data.md)*  
*End of Key Technologies section*
