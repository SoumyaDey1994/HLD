# Vector Database — Step-by-Step Workflow

## 1. Raw Data Input

Application sends:

* Text
* Image
* Audio
* Video
* Documents

Example:

```text id="k1a2mx"
"Sky is blue"
```

---

## 2. Embedding Generation

An embedding model converts the input into a high-dimensional numerical vector.

```text id="b7v4np"
"Sky is blue"
      ↓
[0.12, -0.44, 0.91, ...]
```

Popular embedding models:

* Transformer Models
* BERT
* OpenAI Embeddings
* Sentence Transformers

---

## 3. Store in Vector DB

Vector DB stores:

* Original content
* Metadata
* Embedding vector

| ID  | Text        | Vector      | Metadata         |
| --- | ----------- | ----------- | ---------------- |
| 101 | Sky is blue | [0.12, ...] | category=science |

---

## 4. Vector Indexing

Vector DB builds ANN (Approximate Nearest Neighbor) indexes for fast similarity search.

Common indexing algorithms:

* HNSW
* IVF
* PQ
* FAISS

Purpose:

* Avoid full vector scan
* Enable millisecond-level search

---

## 5. User Query Arrives

Example query:

```text id="m9zq3r"
"Why ocean looks blue?"
```

---

## 6. Query Embedding Creation

The same embedding model converts the query into a vector.

```text id="tw4j6l"
"Why ocean looks blue?"
           ↓
[0.15, -0.39, 0.87, ...]
```

---

## 7. Similarity Search

Vector DB compares query vector against stored vectors using:

* Cosine Similarity
* Euclidean Distance
* Dot Product

ANN indexes help retrieve nearest vectors quickly.

---

## 8. Top-K Matching Results Returned

Example:

| Rank | Matching Text                              |
| ---- | ------------------------------------------ |
| 1    | Sky is blue                                |
| 2    | Ocean appears blue due to light scattering |

---

## 9. RAG / AI Application Uses Results

Retrieved documents are sent to LLM as context.

```text id="uq3d9v"
Retrieved Context
        ↓
LLM
        ↓
Final AI Response
```

---

# High-Level Summary

```text id="tb2s4k"
Raw Data
   ↓
Embedding Model
   ↓
Vector Creation
   ↓
Store + ANN Indexing
   ↓
Query → Vector
   ↓
Similarity Search
   ↓
Top-K Matches
   ↓
LLM / Application Response
```
