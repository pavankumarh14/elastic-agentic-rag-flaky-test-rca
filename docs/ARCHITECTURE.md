# Architecture Overview: Elastic Flaky Test RCA Agent

## System Design

This document describes the production-grade architecture for the Elastic Flaky Test RCA Agent built with Elasticsearch Serverless, Agent Builder, RAG, and JinaAI Semantic Reranking.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD Test Execution Layer                   │
│  (Jenkins, GitHub Actions, GitLab CI) - Generates Test Data     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Elasticsearch Serverless Indices                   │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐ │
│  │  test-execution-logs     │  │      host-metrics           │ │
│  │  - test_name             │  │  - host                     │ │
│  │  - status                │  │  - cpu_usage                │ │
│  │  - error_message         │  │  - memory_usage             │ │
│  │  - host                  │  │  - disk_io                  │ │
│  │  - duration_ms           │  │  - timestamp                │ │
│  └──────────────────────────┘  └─────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ES|QL Query Engine                             │
│  - Correlation Analysis (JOIN test logs + host metrics)         │
│  - Time-based filtering (±5 min window)                         │
│  - Aggregations (STATS, COUNT, AVG, etc.)                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    RAG Pipeline (Agent Builder)                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  1. Retrieve: Execute ES|QL query (fetch 50-100 docs)    │  │
│  │  2. Augment: Pass to JinaAI Reranker                      │  │
│  │  3. Generate: LLM produces natural language RCA report   │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              JinaAI Semantic Reranker (v3)                       │
│  - Input: User query + 50-100 retrieved documents               │
│  - Model: jina-reranker-v2-base-multilingual                    │
│  - Output: Top 5-10 semantically relevant documents             │
│  - Score: Relevance score (0-1) for each document               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Agent Builder Interface                         │
│  - User Query: "Why did cart_checkout_flow fail?"               │
│  - Agent Response: Root cause analysis with citations           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Data Ingestion Layer

**Source:** CI/CD pipelines (Jenkins, GitHub Actions, GitLab CI)

**Data Flow:**
1. Test execution completes
2. Results sent to Elasticsearch via:
   - Logstash
   - Elasticsearch Bulk API
   - Filebeat

**Indices Created:**
- `test-execution-logs`: Test results, errors, timing
- `host-metrics`: System resource metrics (CPU, memory, disk I/O)

---

### 2. Elasticsearch Serverless

**Configuration:**
- **Project Type:** Elasticsearch
- **Deployment:** Serverless (auto-scaling)
- **Region:** us-central1 (GCP)
- **Free Tier:** 14 days trial

**Index Mappings:**
```json
{
  "test-execution-logs": {
    "properties": {
      "test_name": {"type": "text"},
      "execution_time": {"type": "date"},
      "status": {"type": "keyword"},
      "duration_ms": {"type": "long"},
      "error_message": {"type": "text"},
      "host": {"type": "keyword"},
      "build_id": {"type": "keyword"}
    }
  },
  "host-metrics": {
    "properties": {
      "host": {"type": "keyword"},
      "timestamp": {"type": "date"},
      "cpu_usage": {"type": "float"},
      "memory_usage": {"type": "float"},
      "disk_io": {"type": "float"}
    }
  }
}
```

---

### 3. ES|QL Query Engine

**Purpose:** Correlate test failures with system metrics

**Key Query Pattern:**
```esql
FROM test-execution-logs, host-metrics
| WHERE test-execution-logs.status == "failed"
  AND test-execution-logs.host == host-metrics.host
  AND ABS(TO_LONG(test-execution-logs.execution_time) - 
          TO_LONG(host-metrics.timestamp)) < 300000
| STATS 
    avg_cpu = AVG(host-metrics.cpu_usage),
    avg_memory = AVG(host-metrics.memory_usage),
    failure_count = COUNT(*)
  BY test-execution-logs.test_name
```

**Features:**
- Time-windowed joins (±5 minutes)
- Multi-index queries
- Aggregations and filtering

---

### 4. RAG Pipeline

**Implemented via:** Elasticsearch Agent Builder

**Workflow:**

**Step 1: Retrieve**
- User asks: *"Why did cart_checkout_flow fail on jenkins-agent-41?"*
- Agent executes ES|QL query
- Retrieves 50-100 candidate documents

**Step 2: Augment**
- Documents passed to JinaAI Reranker
- Reranker scores each document by semantic relevance
- Top 5-10 documents selected

**Step 3: Generate**
- LLM receives:
  - User query
  - Top reranked documents
  - System prompt (RCA expert persona)
- Generates structured RCA report

---

### 5. JinaAI Semantic Reranker

**Model:** jina-reranker-v2-base-multilingual

**Configuration:**
```json
PUT _inference/rerank/jina-reranker-v3
{
  "service": "jinaai",
  "service_settings": {
    "api_key": "YOUR_JINA_API_KEY",
    "model_id": "jina-reranker-v2-base-multilingual"
  }
}
```

**Why Reranking?**
- **Problem:** ES|QL returns BM25-ranked results (keyword-based)
- **Solution:** Semantic reranking prioritizes contextually relevant docs
- **Example:**
  - Query: *"Why timeout on payment gateway?"*
  - BM25 might rank "timeout" docs higher even if unrelated
  - Reranker understands "payment gateway timeout" context

**Performance:**
- Input: 50-100 documents
- Output: Top 5-10 documents
- Latency: ~200-500ms

---

### 6. Agent Builder

**Agent Configuration:**
- **Name:** Elastic AI Agent
- **Indices:** `test-execution-logs`, `host-metrics`
- **Reranker:** jina-reranker-v3
- **RAG Enabled:** Yes

**System Prompt:**
```
You are an expert SRE and test automation specialist. Analyze flaky test 
failures by:
1. Correlating test execution logs with host metrics
2. Identifying resource bottlenecks (CPU, memory, disk I/O)
3. Detecting patterns across failed builds
4. Providing actionable root cause analysis

Always cite specific data points from logs and metrics.
```

---

## Data Flow Example

### User Query:
```
"Why did cart_checkout_flow fail on jenkins-agent-41?"
```

### Processing Steps:

1. **Agent Builder receives query**
   - Extracts entities: test_name="cart_checkout_flow", host="jenkins-agent-41"

2. **ES|QL Query Execution**
   ```esql
   FROM test-execution-logs, host-metrics
   | WHERE test-execution-logs.test_name == "cart_checkout_flow"
     AND test-execution-logs.host == "jenkins-agent-41"
     AND test-execution-logs.status == "failed"
   | LIMIT 50
   ```

3. **Reranking**
   - JinaAI reranks 50 docs
   - Selects top 5 most relevant

4. **Context Augmentation**
   - Top 5 docs:
     1. "Timeout waiting for payment gateway" (score: 0.92)
     2. "CPU usage 92.5% during failure" (score: 0.88)
     3. "jenkins-agent-41 disk I/O spike" (score: 0.85)
     4. "Memory usage 88.3%" (score: 0.82)
     5. "Test passed on jenkins-agent-42" (score: 0.78)

5. **LLM Generation**
   - Agent produces:
   ```
   Root Cause Analysis for cart_checkout_flow failure:
   
   The test failed on jenkins-agent-41 due to a timeout waiting for 
   the payment gateway (3,200ms vs expected 1,800ms). This correlates 
   with critical resource constraints:
   
   - CPU: 92.5% (threshold: 80%)
   - Memory: 88.3% (threshold: 75%)
   - Disk I/O: 450.2 MB/s (spike detected)
   
   Recommendation: Provision more resources for jenkins-agent-41 or 
   distribute tests to underutilized hosts like jenkins-agent-42.
   ```

---

## Scalability Considerations

### Current Setup (Demo)
- **Data Volume:** 100-1,000 test executions/day
- **Indices:** 2 (test-execution-logs, host-metrics)
- **Query Latency:** 200-500ms

### Production Scaling
- **Data Volume:** 10,000+ executions/day
- **Optimizations:**
  - Index lifecycle management (ILM) - rollover after 30 days
  - Partition indices by date: `test-execution-logs-2026-01-15`
  - Increase reranker `top_n` threshold for complex queries
  - Cache frequent ES|QL queries

---

## Security & Access Control

### API Key Management
- **JinaAI API Key:** Stored in Elasticsearch secrets
- **Elasticsearch Access:** Role-based (read-only for Agent Builder)
- **Agent Sharing:** Public read-only URL (no PII in demo data)

### Production Recommendations
- Use Elasticsearch API keys with limited scopes
- Encrypt JinaAI API key in environment variables
- Implement rate limiting on agent queries

---

## Technology Stack Summary

| Component         | Technology                          | Purpose                        |
|-------------------|-------------------------------------|--------------------------------|
| **Data Storage**  | Elasticsearch Serverless            | Index test logs & metrics      |
| **Query Engine**  | ES|QL                               | Correlate data across indices  |
| **RAG Framework** | Agent Builder (Elasticsearch)       | Orchestrate retrieve-generate  |
| **Reranker**      | JinaAI (jina-reranker-v2)           | Semantic relevance scoring     |
| **LLM**           | Built-in Agent Builder LLM          | Generate RCA reports           |
| **CI/CD**         | Jenkins (demo data generator)       | Test execution logs source     |

---

## Future Enhancements

1. **Real-Time Alerting:**
   - Webhook integration to Slack/PagerDuty on flaky test detection

2. **Predictive Analytics:**
   - ML model to predict flakiness before test execution

3. **Multi-Cloud Support:**
   - Azure DevOps, AWS CodePipeline integration

4. **Advanced Visualizations:**
   - Kibana dashboards for failure trends

---

## References

- [Elasticsearch Agent Builder Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/agent-builder.html)
- [ES|QL Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql.html)
- [JinaAI Reranker API](https://jina.ai/reranker/)
- [RAG Architecture Best Practices](https://www.elastic.co/blog/retrieval-augmented-generation)

---

**Architected by:** Pavan Kumar  
**Last Updated:** January 2026  
**Elastic Blogathon 2026 Submission**
