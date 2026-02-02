# Setup Guide: Elastic Flaky Test RCA Agent

## Prerequisites

1. **Elastic Cloud Account** (14-day free trial)
2. **JinaAI API Key** (free tier available at https://jina.ai/)
3. Basic knowledge of Elasticsearch and ES|QL

---

## Step 1: Create Elasticsearch Serverless Project

1. Navigate to [Elastic Cloud Console](https://cloud.elastic.co/)
2. Click **Create Project** → Select **Elasticsearch**
3. Project Name: `Flaky-Test-RCA` (or your choice)
4. Region: Choose closest to your location
5. Wait for project provisioning (~2 minutes)

---

## Step 2: Setup Indices

### A. Create Test Execution Logs Index

```json
PUT /test-execution-logs
{
  "mappings": {
    "properties": {
      "test_name": { "type": "text" },
      "execution_time": { "type": "date" },
      "status": { "type": "keyword" },
      "duration_ms": { "type": "long" },
      "error_message": { "type": "text" },
      "host": { "type": "keyword" },
      "build_id": { "type": "keyword" }
    }
  }
}
```

### B. Create Host Metrics Index

```json
PUT /host-metrics
{
  "mappings": {
    "properties": {
      "host": { "type": "keyword" },
      "timestamp": { "type": "date" },
      "cpu_usage": { "type": "float" },
      "memory_usage": { "type": "float" },
      "disk_io": { "type": "float" }
    }
  }
}
```

---

## Step 3: Load Sample Data

### Test Execution Logs

```json
POST /test-execution-logs/_bulk
{"index":{}}
{"test_name":"cart_checkout_flow","execution_time":"2026-01-15T10:30:00Z","status":"failed","duration_ms":3200,"error_message":"Timeout waiting for payment gateway","host":"jenkins-agent-41","build_id":"build-1523"}
{"index":{}}
{"test_name":"cart_checkout_flow","execution_time":"2026-01-15T11:00:00Z","status":"passed","duration_ms":1800,"host":"jenkins-agent-42","build_id":"build-1524"}
{"index":{}}
{"test_name":"user_login_test","execution_time":"2026-01-15T10:45:00Z","status":"passed","duration_ms":1200,"host":"jenkins-agent-41","build_id":"build-1523"}
```

### Host Metrics

```json
POST /host-metrics/_bulk
{"index":{}}
{"host":"jenkins-agent-41","timestamp":"2026-01-15T10:30:00Z","cpu_usage":92.5,"memory_usage":88.3,"disk_io":450.2}
{"index":{}}
{"host":"jenkins-agent-42","timestamp":"2026-01-15T11:00:00Z","cpu_usage":45.2,"memory_usage":62.1,"disk_io":120.5}
```

---

## Step 4: Configure JinaAI Reranker

### A. Get JinaAI API Key

1. Visit https://jina.ai/embeddings/
2. Sign up (free tier: 1M tokens/month)
3. Navigate to **API Keys** section
4. Generate new API key → Copy it

### B. Add Reranker to Elasticsearch

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

### C. Test Reranker

```json
POST _inference/rerank/jina-reranker-v3/_rerank
{
  "query": "Why did cart_checkout_flow fail?",
  "documents": [
    "Timeout waiting for payment gateway on jenkins-agent-41",
    "Test passed successfully on jenkins-agent-42",
    "CPU usage was 92.5% on jenkins-agent-41 during failure"
  ],
  "top_n": 3
}
```

---

## Step 5: Create Agent Builder

1. Navigate to **Agent Builder** in Kibana sidebar
2. Click **Create new agent**
3. **Agent Name:** `Elastic AI Agent`
4. **Description:** "Production-grade RCA assistant for flaky test analysis"
5. **Indices to search:** `test-execution-logs`, `host-metrics`
6. **System Prompt:**

```
You are an expert SRE and test automation specialist. Analyze flaky test failures by:
1. Correlating test execution logs with host metrics
2. Identifying resource bottlenecks (CPU, memory, disk I/O)
3. Detecting patterns across failed builds
4. Providing actionable root cause analysis

Always cite specific data points from logs and metrics.
```

7. **Enable RAG:** Check "Retrieve from indices"
8. **Reranker:** Select `jina-reranker-v3`
9. Click **Create Agent**

---

## Step 6: Test the Agent

### Sample Queries

1. **"Why did cart_checkout_flow fail on jenkins-agent-41?"**
   - Expected: RCA showing timeout + high CPU correlation

2. **"Compare test performance across different hosts"**
   - Expected: Performance metrics comparison

3. **"Show me all failed tests with resource bottlenecks"**
   - Expected: ES|QL query with correlated data

---

## Verification Checklist

- [ ] Elasticsearch project created
- [ ] Indices `test-execution-logs` and `host-metrics` populated
- [ ] JinaAI reranker configured and tested
- [ ] Agent Builder agent created with RAG enabled
- [ ] Sample queries return accurate RCA results
- [ ] ES|QL queries execute without errors

---

## Troubleshooting

### Issue: "400 Bad Request" from JinaAI
**Solution:** Check API key validity and model_id spelling

### Issue: Agent returns no results
**Solution:** Verify indices have data using `GET /test-execution-logs/_search`

### Issue: ES|QL syntax errors
**Solution:** Ensure `FROM` statement lists both indices: `FROM test-execution-logs, host-metrics`

---

## Next Steps

- Add your own test execution data
- Customize ES|QL queries for your use case
- Integrate with CI/CD pipeline (Jenkins/GitHub Actions)
- Export RCA reports for team dashboards

---

**Demo Video:** [Add your YouTube link here]
**Blog Post:** [Add your Medium/Dev.to link here]
