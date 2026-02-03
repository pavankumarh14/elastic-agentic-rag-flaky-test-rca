# 🛠️ Technology Stack Visual

## Core Technologies

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          AI & SEMANTIC SEARCH STACK                             │
└─────────────────────────────────────────────────────────────────────────────────┘

       ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
       │   ELSER      │         │   JinaAI     │         │   AGENT      │
       │   MODEL      │         │   RERANKER   │         │   BUILDER    │
       └──────────────┘         └──────────────┘         └──────────────┘
              ▲                        ▲                        ▲
              │                        │                        │
              └───────────────┬────────┴────────────────────────┘
                              │
                    ┌──────────────────┐
                    │  ELASTICSEARCH   │
                    │   SERVERLESS     │
                    └──────────────────┘
                              ▲
                              │
                    ┌──────────────────┐
                    │      ESQL        │
                    │     ENGINE       │
                    └──────────────────┘
```

## Detailed Component Breakdown

### 1. Data & Infrastructure Layer
- **Elasticsearch Serverless**: The foundational vector database and search engine.
  - *Role*: Scalable storage, high-speed retrieval, and vector processing.
  - *Key Benefit*: No server management, auto-scaling for hackathon-scale traffic.
- **Elastic Cloud**: Managed infrastructure hosting the serverless project.

### 2. Search & Intelligence Layer
- **ELSER (Elastic Learned Sparse Encoder)**:
  - *Role*: Text expansion and semantic embedding.
  - *Key Benefit*: Bridges the gap between user intent and document content without complex fine-tuning.
- **JinaAI Reranker**:
  - *Role*: Precision cross-encoder reranking.
  - *Key Benefit*: Ensures the most relevant RCA documents are always at the top of the context.
- **ESQL (Elasticsearch Query Language)**:
  - *Role*: Advanced analytics and multi-index correlation.
  - *Key Benefit*: Single query language for filtering, joining, and aggregating test failure data.

### 3. AI Agent Layer
- **Elastic Agent Builder**:
  - *Role*: The "Brain" of the system.
  - *Key Benefit*: Built-in RAG capabilities, conversational memory, and LLM orchestration.
- **RAG Pipeline**:
  - *Role*: Context augmentation and grounding.
  - *Key Benefit*: Provides factual, high-confidence root cause analysis based on verified data.

## Why This Stack Wins?

| Component | Why it matters? |
|-----------|-----------------|
| **Serverless** | Rapid deployment and infinite scale. |
| **ELSER** | Semantic search that actually works out-of-the-box. |
| **JinaAI** | Industry-leading precision for reranking. |
| **ESQL** | Clean, pipe-based syntax for complex data analysis. |
| **Agent Builder** | Low-code/No-code speed for building complex AI agents. |

## Integrated Workflow

```
[ User ] -> [ Agent Builder ] -> [ RAG ] -> [ ELSER + JinaAI ] -> [ ESQL ] -> [ Results ]
```
