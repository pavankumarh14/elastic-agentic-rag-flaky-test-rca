# RAG Pipeline for Flaky Test RCA

## Overview

This document explains the Retrieval Augmented Generation (RAG) pipeline used in the Elastic AI Agent for intelligent flaky test root cause analysis.

---

## Pipeline Architecture

### High-Level Flow

```
User Query → Query Processing → Vector Search (ELSER) → JinaAI Reranking → Context Assembly → LLM Generation → Response
```

---

## Key Components

### 1. **Query Processing**

**Purpose**: Transform natural language queries into semantic representations

**Implementation**:
- User queries are processed through the Agent Builder interface
- Queries are analyzed for intent (troubleshooting, metrics, history)
- Context is extracted from conversation history

**Example Queries**:
```
"Why did the login test fail?"
"Show me all flaky tests from yesterday"
"What's the root cause of checkout failures?"
```

---

### 2. **Vector Search with ELSER**

**What is ELSER?**
- Elastic Learned Sparse EncodeR (ELSER) v2
- Semantic search model trained by Elastic
- Provides better relevance than keyword matching

**Configuration**:
```json
{
  "model_id": ".elser_model_2",
  "field": "ml.inference.error_message_expanded",
  "similarity": "cosine"
}
```

**Indexed Fields**:
1. `test_name` - Test identifier
2. `error_message` - Failure details
3. `stack_trace` - Technical context
4. `test_category` - Classification
5. `timestamp` - Temporal data

**Search Process**:
```
1. Convert query to ELSER embeddings
2. Search across test-execution-logs index
3. Retrieve top k candidates (k=10)
4. Score by semantic similarity
```

---

### 3. **JinaAI Reranking**

**Why Reranking?**
- Initial retrieval may include irrelevant results
- Reranking refines results based on query-document relevance
- Improves final context quality for LLM

**Implementation**:

**Model**: `jina-reranker-v3`

**Configuration**:
```json
{
  "service": "jinaai",
  "service_settings": {
    "api_key": "<JINA_API_KEY>",
    "model_id": "jina-reranker-v2-base-multilingual"
  },
  "task_settings": {
    "return_documents": false,
    "top_n": 5
  }
}
```

**Reranking Process**:
```
1. Take top 10 ELSER results
2. Send to JinaAI reranker with original query
3. Get relevance scores for each document
4. Select top 5 most relevant documents
5. Pass to LLM as context
```

**Benefits**:
- ✅ Cross-encoder architecture (better than bi-encoder)
- ✅ Multilingual support
- ✅ Query-document relevance scoring
- ✅ Reduces hallucination by improving context quality

---

### 4. **Context Assembly**

**Purpose**: Build comprehensive context for LLM from retrieved documents

**Context Structure**:
```json
{
  "query": "Why did login test fail?",
  "retrieved_documents": [
    {
      "test_name": "test_user_login",
      "error_message": "Element not found: #login-button",
      "timestamp": "2025-01-15T10:30:00Z",
      "environment": "staging",
      "relevance_score": 0.95
    }
  ],
  "conversation_history": [],
  "system_context": "You are an expert QA analyst..."
}
```

**Context Optimization**:
- Limit to top 5 documents (prevent token overflow)
- Include timestamps for temporal analysis
- Add environment metadata
- Preserve relevance scores

---

### 5. **LLM Generation**

**Model**: OpenAI GPT-4 (configurable)

**System Prompt**:
```
You are an expert QA automation engineer specializing in flaky test analysis.
Use the provided test execution logs and error messages to identify root causes.
Provide actionable recommendations to fix the issues.
```

**Generation Parameters**:
```json
{
  "temperature": 0.3,
  "max_tokens": 1000,
  "top_p": 0.9,
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0
}
```

**Response Structure**:
- Root cause identification
- Evidence from logs
- Fix recommendations
- Related issues (if any)

---

## Data Flow Diagram

```
┌─────────────────┐
│   User Query    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Query Embedding │  (ELSER)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Vector Search   │  (Elasticsearch)
│  (Top 10)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ JinaAI Rerank   │  (Top 5)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Context Assembly│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM Generation │  (GPT-4)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Response      │
└─────────────────┘
```

---

## Metrics & Performance

### Key Metrics

1. **Response Time**: Target <5 seconds
2. **Relevance Score**: JinaAI scores >0.7
3. **Retrieval Precision**: Top-5 accuracy
4. **Query Success Rate**: % of actionable responses

### Performance Optimization

**Caching Strategy**:
```
- Cache frequent queries
- TTL: 1 hour
- Invalidate on new data ingestion
```

**Batch Processing**:
```
- Group similar queries
- Parallel retrieval
- Async reranking
```

---

## Configuration

### ELSER Setup

```bash
# 1. Start ELSER model
POST _ml/trained_models/.elser_model_2/deployment/_start

# 2. Create ingest pipeline
PUT _ingest/pipeline/elser-ingest-pipeline
{
  "processors": [
    {
      "inference": {
        "model_id": ".elser_model_2",
        "field_map": {
          "error_message": "text_field"
        },
        "target_field": "ml.inference"
      }
    }
  ]
}

# 3. Apply to index
PUT test-execution-logs/_settings
{
  "index.default_pipeline": "elser-ingest-pipeline"
}
```

### JinaAI Reranker Setup

```bash
# Create inference endpoint
PUT _inference/rerank/jina-reranker-v3
{
  "service": "jinaai",
  "service_settings": {
    "api_key": "${JINA_API_KEY}",
    "model_id": "jina-reranker-v2-base-multilingual"
  },
  "task_settings": {
    "top_n": 5
  }
}
```

---

## Usage Examples

### Example 1: Troubleshooting Query

**Query**: "Why is the payment test failing?"

**RAG Process**:
```
1. ELSER retrieves 10 payment-related test failures
2. JinaAI reranks and selects 5 most relevant
3. Context includes:
   - Recent payment test errors
   - Stack traces
   - Environment details
4. LLM generates root cause analysis
```

**Response**:
```
The payment test is failing due to a timeout waiting for the
payment gateway response. Evidence:
- Error: "TimeoutException: Element #payment-confirm not found"
- 3 out of 5 recent runs show same issue
- Occurs only in staging environment

Recommendation:
- Increase wait timeout from 5s to 10s
- Add retry logic for payment gateway calls
- Check staging payment gateway health
```

### Example 2: Trend Analysis

**Query**: "Show flaky test trends this week"

**RAG Process**:
```
1. ELSER retrieves test failures from past 7 days
2. Reranker prioritizes recent failures
3. Context includes temporal patterns
4. LLM analyzes trends
```

**Response**:
```
Flaky test trends this week:
1. login_test: 60% failure rate (up from 20%)
2. checkout_test: 15% failure rate (stable)
3. search_test: 5% failure rate (improved)

Primary issues:
- login_test: Increased timeout errors
- Likely cause: Recent API latency increase
```

---

## Best Practices

### 1. **Query Optimization**
- Be specific in queries
- Include context (time, environment)
- Use natural language

### 2. **Index Maintenance**
```bash
# Regular reindexing
POST _reindex
{
  "source": {"index": "test-execution-logs"},
  "dest": {"index": "test-execution-logs-v2"}
}

# Clean old data
DELETE test-execution-logs/_doc/_query
{
  "query": {
    "range": {
      "timestamp": {"lt": "now-90d"}
    }
  }
}
```

### 3. **Monitoring**
- Track retrieval latency
- Monitor reranker API usage
- Alert on low relevance scores

---

## Troubleshooting

### Issue: Low Relevance Scores

**Solution**:
```
1. Check ELSER model status
2. Verify index mappings
3. Tune reranker top_n parameter
4. Review query phrasing
```

### Issue: Slow Response Times

**Solution**:
```
1. Enable caching
2. Reduce top_n in reranker
3. Optimize index shard configuration
4. Use async processing
```

### Issue: Irrelevant Results

**Solution**:
```
1. Improve query specificity
2. Add filters (date, environment)
3. Increase reranker threshold
4. Retrain ELSER on domain data
```

---

## Advanced Features

### Hybrid Search

Combine ELSER with keyword search:

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "text_expansion": {
            "ml.inference.error_message_expanded": {
              "model_id": ".elser_model_2",
              "model_text": "login failure"
            }
          }
        },
        {
          "match": {
            "test_name": "login"
          }
        }
      ]
    }
  }
}
```

### Multi-Index Search

Search across multiple indices:

```
GET test-execution-logs,host-metrics/_search
{
  "query": {...},
  "rerank": {...}
}
```

---

## References

- [Elasticsearch ELSER Documentation](https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-elser.html)
- [JinaAI Reranker API](https://jina.ai/reranker/)
- [RAG Best Practices](https://www.elastic.co/blog/retrieval-augmented-generation)

---

## Next Steps

✅ **Completed**: Basic RAG pipeline setup

**TODO**:
1. Add feedback loop for relevance tuning
2. Implement A/B testing for reranker models
3. Create custom ELSER fine-tuning dataset
4. Add multi-modal support (logs + screenshots)
