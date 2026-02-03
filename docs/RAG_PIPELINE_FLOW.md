# 🌊 RAG Pipeline Flow Diagram

## End-to-End RAG Workflow

```
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│   USER QUERY     │          │   PRE-PROCESS    │          │   EMBEDDING      │
│ (Failed Test RCA)│───1─────►│  (Clean/Token)   │───2─────►│  (ELSER Model)   │
└──────────────────┘          └──────────────────┘          └──────────────────┘
                                                                    │
                                                                    3
                                                                    ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│    PRECISION     │          │    RETRIEVAL     │          │   VECTOR SEARCH  │
│    RERANKING     │◄──6──────│   (Hybrid)       │◄──5──────│   (k-NN HNSW)    │
│ (JinaAI Reranker)│          │ (BM25 + Vector)  │          │ (Elastic Search) │
└──────────────────┘          └──────────────────┘          └──────────────────┘
         │
         7
         ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│   CONTEXT        │          │   LLM PROMPT     │          │    FINAL         │
│   CONSTRUCTION   │───8─────►│   (Augmented)    │───9─────►│    RESPONSE      │
│ (Top-K Matches)  │          │ (Context + Query)│          │ (Root Cause Fix) │
└──────────────────┘          └──────────────────┘          └──────────────────┘
```

## Detailed Pipeline Steps

### 1. User Input & Query Processing
- **Input**: User asks "Why did test `cart_checkout_flow` fail with timeout?"
- **Processing**: The system extracts key entities (Test Name, Error Type).

### 2. Semantic Embedding (ELSER)
- **Model**: Elastic Learned Sparse Encoder (ELSER).
- **Function**: Converts the natural language query into a high-dimensional sparse vector.
- **Benefit**: Captures the semantic meaning rather than just matching keywords.

### 3. Hybrid Retrieval (BM25 + Vector)
- **Vector Search**: Performs k-Nearest Neighbor (k-NN) search using HNSW index.
- **Lexical Search**: Uses BM25 for exact keyword matching (e.g., specific Error IDs).
- **Reciprocal Rank Fusion (RRF)**: Combines results from both searches for a balanced list.

### 4. Precision Reranking (JinaAI)
- **Input**: Top 50 results from hybrid retrieval.
- **Model**: JinaAI Cross-Encoder Reranker.
Add RAG Pipeline Flow diagram- **Output**: Top 5-10 highly relevant documents (solutions, patterns, history).

### 5. Context Construction
- **Augmentation**: The retrieved documents are formatted into a context block.
- **Grounding**: This context provides the LLM with factual data to prevent hallucinations.

### 6. LLM Generation
- **Prompt**: `Query + Construction Context + System Instructions`.
- **Model**: Integrated LLM via Elastic Agent Builder.
- **Constraint**: The model is instructed to ONLY use the provided context for the RCA.

### 7. Final Response
- **Output**: A structured explanation of the root cause.
- **Extras**: Suggested fix steps, link to similar past issues, and confidence score.

## Feedback Loop (Enhancement)

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Agent       │      │   User       │      │  Continuous  │
│  Response    │─────►│   Feedback   │─────►│  Learning    │
└──────────────┘      └──────────────┘      └──────────────┘
                             │                     │
                             └──────────┬──────────┘
                                        ▼
                              ┌──────────────────┐
                              │  Index Update    │
                              │ (New Solutions)  │
                              └──────────────────┘
```
- **Thumbs Up/Down**: Users can rate the accuracy of the RCA.
- **Knowledge Base**: Verified solutions are automatically indexed back into the system.
