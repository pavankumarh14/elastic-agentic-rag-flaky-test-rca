# Agent Builder Creation Guide

## Overview

This guide walks you through creating the "Elastic AI Agent" using Elasticsearch Agent Builder for intelligent flaky test root cause analysis.

---

## Prerequisites

Before creating the agent, ensure you have:

1. ✅ Elasticsearch Serverless project created (`Flaky-Test-RCA`)
2. ✅ Indices populated with data:
   - `test-execution-logs`
   - `host-metrics`
3. ✅ JinaAI reranker configured (`jina-reranker-v3`)

---

## Step 1: Access Agent Builder

### Navigate to Agent Builder

1. Log in to your Elastic Cloud console
2. Select your project: **Flaky-Test-RCA**
3. In the left sidebar, click **"Agent Builder"** (or search for "Agent Builder" in Kibana)
4. Click **"Create new agent"**

**Screenshot Location:** Kibana → Management → Agent Builder

---

## Step 2: Configure Basic Agent Settings

### Agent Name
```
Elastic AI Agent
```

### Description
```
Production-grade RCA assistant for flaky test analysis. Correlates test failures with system metrics to provide actionable root cause analysis.
```

### Knowledge Base Selection

Select **"Use Elasticsearch indices"**

**Indices to search:**
- `test-execution-logs`
- `host-metrics`

**Why these indices?**
- `test-execution-logs`: Contains test execution results, errors, timing
- `host-metrics`: Contains CPU, memory, disk I/O metrics

---

## Step 3: Configure System Prompt

The system prompt defines the agent's persona and behavior. Copy and paste this exact prompt:

```
You are an expert SRE and test automation specialist with deep knowledge of:
- Flaky test analysis and root cause identification
- System resource correlation (CPU, memory, disk I/O)
- CI/CD pipeline troubleshooting
- Performance degradation patterns

Your primary responsibilities:
1. Analyze test execution failures by correlating test logs with host metrics
2. Identify resource bottlenecks and performance constraints
3. Detect flakiness patterns across multiple test runs
4. Provide actionable, data-driven recommendations

Always:
- Cite specific data points from test-execution-logs and host-metrics
- Calculate flakiness percentages when relevant
- Correlate failures with resource usage spikes
- Provide clear, structured responses

Response format:
1. Summary of the issue
2. Data analysis (with specific metrics)
3. Root cause identification
4. Actionable recommendations
```

**Why this prompt matters:**
- Sets expert SRE/QA persona
- Defines clear responsibilities
- Ensures citations from indices
- Standardizes response format

---

## Step 4: Enable RAG (Retrieval-Augmented Generation)

### RAG Configuration

1. **Enable RAG:** ✅ Check "Retrieve from Elasticsearch indices"

2. **Retrieval Settings:**
   - **Documents to retrieve:** 50-100 (default: 50)
   - **Search method:** Hybrid (BM25 + vector search)

3. **Enable Reranker:** ✅ Check "Use reranker"

4. **Select Reranker:**
   - From dropdown, select: `jina-reranker-v3`

5. **Reranker Settings:**
   - **Top N documents:** 10
   - **Relevance threshold:** 0.5

**How RAG Works:**
```
User Query → ES|QL Search (50 docs) → JinaAI Reranker (top 10) → LLM Generation
```

---

## Step 5: Configure Advanced Settings

### Response Length
- **Max tokens:** 1500 (for detailed RCA reports)

### Temperature
- **Value:** 0.3 (lower = more factual, less creative)

### Citations
- ✅ **Enable source citations:** Always cite documents

### Conversation Memory
- **Turns to remember:** 5 (maintains context across queries)

---

## Step 6: Test the Agent

### Sample Test Queries

#### Query 1: Basic Failure Analysis
```
Why did cart_checkout_flow fail on jenkins-agent-41?
```

**Expected Response:**
```
Summary: The cart_checkout_flow test failed due to a timeout error while waiting for the payment gateway response.

Data Analysis:
- Test: cart_checkout_flow
- Host: jenkins-agent-41
- Status: Failed
- Error: "Timeout waiting for payment gateway"
- Duration: 3,200ms (vs. baseline 1,800ms)

Resource Correlation:
- CPU Usage: 92.5% (threshold: 80%)
- Memory Usage: 88.3% (threshold: 75%)
- Disk I/O: 450.2 MB/s (spike detected)

Root Cause: Resource exhaustion on jenkins-agent-41 caused the payment gateway to time out.

Recommendations:
1. Scale jenkins-agent-41 (increase CPU/memory)
2. Distribute tests to jenkins-agent-42 (5% failure rate)
3. Implement retry logic for payment gateway calls
```

#### Query 2: Flakiness Detection
```
Show me all tests with >30% flakiness rate
```

**Expected Response:**
```
Flaky Tests Analysis:

1. cart_checkout_flow
   - Pass: 8 | Fail: 12
   - Flakiness: 60.0%
   - Affected Hosts: jenkins-agent-41
   - Primary Issue: Resource constraints

2. api_response_time
   - Pass: 15 | Fail: 5
   - Flakiness: 25.0%
   - Affected Hosts: jenkins-agent-38
   - Primary Issue: Network latency spikes

Recommendation: Investigate jenkins-agent-41 resource allocation.
```

#### Query 3: Performance Comparison
```
Compare test performance across different hosts
```

**Expected Response:**
```
Host Performance Comparison:

jenkins-agent-41:
- Total Tests: 45
- Failures: 27 (60%)
- Avg CPU: 89.2%
- Avg Memory: 84.1%
- Status: ⚠️ Resource Bottleneck

jenkins-agent-42:
- Total Tests: 50
- Failures: 3 (6%)
- Avg CPU: 48.5%
- Avg Memory: 61.2%
- Status: ✅ Healthy

Recommendation: Migrate cart_checkout_flow to jenkins-agent-42
```

---

## Step 7: Verify Agent Functionality

### Verification Checklist

- [ ] Agent responds to queries in <5 seconds
- [ ] Agent cites specific documents from indices
- [ ] Agent uses JinaAI reranker (check relevance scores)
- [ ] Agent provides actionable recommendations
- [ ] Agent maintains conversation context

### How to Check Reranker Usage

1. Open browser DevTools (F12)
2. Network tab → Filter: "rerank"
3. Send a query to the agent
4. Verify `POST _inference/rerank/jina-reranker-v3/_rerank` request

**Expected Response:**
```json
{
  "reranked_results": [
    {
      "index": 0,
      "score": 0.92,
      "document": "Timeout waiting for payment gateway on jenkins-agent-41"
    },
    {
      "index": 1,
      "score": 0.88,
      "document": "CPU usage 92.5% during cart_checkout_flow failure"
    }
  ]
}
```

---

## Step 8: Share the Agent

### Option 1: Public Link (No Login Required)

1. Click **"Share agent"** button
2. Select **"Public access"**
3. Copy the public URL (format: `https://[project-id].kb.[region].gcp.elastic.cloud/app/agent_builder/conversations/[agent-id]`)
4. Add this link to your README.md:
   ```markdown
   👉 **[Try the Agent](YOUR_PUBLIC_URL)**
   ```

### Option 2: Embed in Application

Use Elastic's Agent API to embed in your CI/CD dashboard:

```javascript
const response = await fetch('https://[project-id].kb.[region].gcp.elastic.cloud/api/agent_builder/chat', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    agent_id: 'YOUR_AGENT_ID',
    message: 'Why did cart_checkout_flow fail?'
  })
});
```

---

## Common Agent Builder Issues

### Issue 1: Agent returns "No results found"

**Cause:** Indices are empty or not properly selected

**Solution:**
1. Verify data in indices:
   ```json
   GET /test-execution-logs/_search
   {
     "size": 5
   }
   ```
2. Re-select indices in Agent Builder settings

### Issue 2: Agent doesn't use reranker

**Cause:** Reranker not properly configured

**Solution:**
1. Verify reranker exists:
   ```json
   GET _inference/rerank/jina-reranker-v3
   ```
2. Re-select reranker in Agent Builder → Advanced Settings

### Issue 3: Agent provides generic responses

**Cause:** System prompt too vague or RAG disabled

**Solution:**
1. Update system prompt with specific instructions
2. Ensure RAG is enabled in settings
3. Increase "Documents to retrieve" to 50+

---

## Best Practices

### 1. System Prompt Optimization

**Do:**
- Be specific about data sources
- Define clear responsibilities
- Request structured outputs
- Emphasize citation requirements

**Don't:**
- Use vague instructions like "be helpful"
- Ask for creative responses (lower temperature for factual data)
- Ignore citation requirements

### 2. Reranker Configuration

**Optimal Settings:**
- Initial retrieval: 50-100 documents
- Reranker top_n: 5-10 documents
- Relevance threshold: 0.5+

**Why?**
- More initial docs = better coverage
- Reranking reduces to most relevant
- Threshold filters out noise

### 3. Conversation Context

**Tips:**
- Enable conversation memory (5+ turns)
- Reference previous queries naturally
- Build on prior context for deeper analysis

**Example:**
```
User: "Why did cart_checkout_flow fail?"
Agent: [Provides analysis]
User: "What about other tests on the same host?"
Agent: [Uses context from previous query about jenkins-agent-41]
```

---

## Advanced: Custom ES|QL in Agent

Agent Builder can execute custom ES|QL queries. Example:

**User:** "Show me tests with CPU > 90%"

**Agent internally executes:**
```esql
FROM test-execution-logs, host-metrics
| WHERE test-execution-logs.host == host-metrics.host
  AND host-metrics.cpu_usage > 90
  AND test-execution-logs.status == "failed"
| STATS failure_count = COUNT(*) BY test-execution-logs.test_name
| SORT failure_count DESC
```

---

## Monitoring Agent Performance

### Metrics to Track

1. **Response Time:** Target <5 seconds
2. **Relevance Score:** JinaAI scores >0.7
3. **User Satisfaction:** Track thumbs up/down
4. **Query Success Rate:** % of queries with actionable responses

### How to Access Metrics

1. Kibana → Agent Builder → Your Agent → Metrics
2. View:
   - Average response time
   - Total queries
   - Reranker hit rate
   - User feedback

---

## Next Steps

✅ **Agent created and tested**

Now:
1. Share public link in README.md
2. Create demo video showing agent in action
3. Document use cases (see USE_CASES.md)
4. Submit to Elastic Blogathon

---

## References

- [Elasticsearch Agent Builder Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/agent-builder.html)
- [RAG Best Practices](https://www.elastic.co/blog/retrieval-augmented-generation)
- [JinaAI Reranker API](https://jina.ai/reranker/)

---

**Created by:** Pavan Kumar  
**Last Updated:** February 2026  
**Elastic Blogathon 2026**
