# ES|QL Queries for Flaky Test RCA

This document contains all ES|QL queries used in the Elastic Flaky Test RCA Agent for analyzing test failures and correlating with system metrics.

---

## 1. Basic Failed Test Analysis

### Query: Find all failed tests

```esql
FROM test-execution-logs
| WHERE status == "failed"
| STATS count = COUNT(*) BY test_name
| SORT count DESC
```

**Purpose:** Identify which tests fail most frequently.

**Expected Output:**
```
test_name              | count
-----------------------|------
cart_checkout_flow     | 12
user_login_test        | 5
payment_processing     | 3
```

---

## 2. Host-Specific Failure Correlation

### Query: Correlate failures with host metrics

```esql
FROM test-execution-logs, host-metrics
| WHERE test-execution-logs.status == "failed"
  AND test-execution-logs.host == host-metrics.host
  AND ABS(TO_LONG(test-execution-logs.execution_time) - TO_LONG(host-metrics.timestamp)) < 300000
| STATS 
    avg_cpu = AVG(host-metrics.cpu_usage),
    avg_memory = AVG(host-metrics.memory_usage),
    failure_count = COUNT(*)
  BY test-execution-logs.host
| WHERE avg_cpu > 80 OR avg_memory > 75
| SORT failure_count DESC
```

**Purpose:** Find hosts with high resource usage during test failures.

**Key Points:**
- Joins two indices on `host` field
- Time window: ±5 minutes (300,000 ms)
- Filters for resource bottlenecks

**Expected Output:**
```
host              | avg_cpu | avg_memory | failure_count
------------------|---------|------------|---------------
jenkins-agent-41  | 92.5    | 88.3       | 15
jenkins-agent-38  | 85.2    | 79.1       | 8
```

---

## 3. Flaky Test Pattern Detection

### Query: Identify flaky tests (pass/fail on same host)

```esql
FROM test-execution-logs
| STATS 
    passed = COUNT_IF(status == "passed"),
    failed = COUNT_IF(status == "failed"),
    total = COUNT(*)
  BY test_name, host
| WHERE passed > 0 AND failed > 0
| EVAL flakiness_rate = ROUND((failed * 100.0 / total), 2)
| SORT flakiness_rate DESC
```

**Purpose:** Calculate flakiness percentage per test per host.

**Expected Output:**
```
test_name          | host             | passed | failed | flakiness_rate
-------------------|------------------|--------|--------|----------------
cart_checkout_flow | jenkins-agent-41 | 8      | 12     | 60.00%
api_response_time  | jenkins-agent-42 | 15     | 5      | 25.00%
```

---

## 4. Time-Based Failure Analysis

### Query: Failures by hour of day

```esql
FROM test-execution-logs
| WHERE status == "failed"
| EVAL hour = DATE_EXTRACT(execution_time, "hour")
| STATS failure_count = COUNT(*) BY hour
| SORT hour ASC
```

**Purpose:** Detect if failures correlate with specific times (e.g., peak load).

**Expected Output:**
```
hour | failure_count
-----|---------------
10   | 25
11   | 32
14   | 18
```

---

## 5. Error Message Pattern Mining

### Query: Group failures by error message

```esql
FROM test-execution-logs
| WHERE status == "failed"
| STATS 
    occurrences = COUNT(*),
    affected_hosts = COUNT_DISTINCT(host)
  BY error_message
| WHERE occurrences > 3
| SORT occurrences DESC
```

**Purpose:** Find common error patterns across multiple failures.

**Expected Output:**
```
error_message                          | occurrences | affected_hosts
---------------------------------------|-------------|----------------
Timeout waiting for payment gateway    | 15          | 3
Database connection pool exhausted     | 8           | 2
Null pointer exception in cart module  | 5           | 4
```

---

## 6. Duration Anomaly Detection

### Query: Tests with abnormal execution times

```esql
FROM test-execution-logs
| STATS 
    avg_duration = AVG(duration_ms),
    max_duration = MAX(duration_ms),
    min_duration = MIN(duration_ms)
  BY test_name
| EVAL duration_variance = max_duration - min_duration
| WHERE duration_variance > 5000
| SORT duration_variance DESC
```

**Purpose:** Identify tests with inconsistent execution times.

**Expected Output:**
```
test_name          | avg_duration | max_duration | duration_variance
-------------------|--------------|--------------|-------------------
cart_checkout_flow | 2100         | 8500         | 7300
api_load_test      | 1800         | 6200         | 5100
```

---

## 7. Build-Specific Failure Rate

### Query: Failure rate per build

```esql
FROM test-execution-logs
| STATS 
    total_tests = COUNT(*),
    failed_tests = COUNT_IF(status == "failed")
  BY build_id
| EVAL failure_rate = ROUND((failed_tests * 100.0 / total_tests), 2)
| WHERE failure_rate > 10
| SORT failure_rate DESC
```

**Purpose:** Identify problematic builds.

**Expected Output:**
```
build_id  | total_tests | failed_tests | failure_rate
----------|-------------|--------------|-------------
build-1523| 45          | 18           | 40.00%
build-1520| 50          | 8            | 16.00%
```

---

## 8. Comprehensive RCA Query (Production-Grade)

### Query: Full correlation analysis

```esql
FROM test-execution-logs, host-metrics
| WHERE test-execution-logs.status == "failed"
  AND test-execution-logs.host == host-metrics.host
  AND ABS(TO_LONG(test-execution-logs.execution_time) - TO_LONG(host-metrics.timestamp)) < 300000
| EVAL 
    cpu_category = CASE(
      host-metrics.cpu_usage > 90, "Critical",
      host-metrics.cpu_usage > 70, "High",
      "Normal"
    ),
    memory_category = CASE(
      host-metrics.memory_usage > 85, "Critical",
      host-metrics.memory_usage > 65, "High",
      "Normal"
    )
| STATS 
    failure_count = COUNT(*),
    avg_duration = AVG(test-execution-logs.duration_ms),
    avg_cpu = AVG(host-metrics.cpu_usage),
    avg_memory = AVG(host-metrics.memory_usage),
    unique_errors = COUNT_DISTINCT(test-execution-logs.error_message)
  BY test-execution-logs.test_name, cpu_category, memory_category
| WHERE cpu_category == "Critical" OR memory_category == "Critical"
| SORT failure_count DESC
```

**Purpose:** Production-grade RCA combining all factors.

**Expected Output:**
```
test_name          | cpu_category | memory_category | failure_count | avg_cpu | avg_memory | unique_errors
-------------------|--------------|-----------------|---------------|---------|------------|---------------
cart_checkout_flow | Critical     | Critical        | 12            | 92.5    | 88.3       | 3
api_timeout_test   | Critical     | High            | 5             | 91.2    | 72.5       | 2
```

---

## 9. Reranker-Optimized Query

### Query: Generate documents for JinaAI reranking

```esql
FROM test-execution-logs
| WHERE status == "failed"
| EVAL rca_document = CONCAT(
    "Test: ", test_name,
    " | Error: ", error_message,
    " | Host: ", host,
    " | Duration: ", TO_STRING(duration_ms), "ms",
    " | Time: ", TO_STRING(execution_time)
  )
| KEEP rca_document, test_name, host, execution_time
| LIMIT 20
```

**Purpose:** Format failure data for semantic reranking.

**Usage with Reranker:**
```json
POST _inference/rerank/jina-reranker-v3/_rerank
{
  "query": "Why did cart_checkout_flow fail with timeout?",
  "documents": [/* results from above query */],
  "top_n": 5
}
```

---

## 10. Real-Time Monitoring Query

### Query: Recent failures (last 1 hour)

```esql
FROM test-execution-logs
| WHERE status == "failed"
  AND execution_time >= NOW() - 1 hour
| STATS recent_failures = COUNT(*) BY test_name, host
| SORT recent_failures DESC
```

**Purpose:** Live dashboard for CI/CD monitoring.

---

## Query Performance Tips

1. **Index Optimization:**
   - Ensure `host`, `status`, `execution_time` are properly indexed
   - Use keyword type for exact matches

2. **Time Range Filtering:**
   - Always add time filters to reduce data scanned
   - Use `NOW()` for relative time queries

3. **Join Optimization:**
   - Keep time window small (5-10 minutes max)
   - Use `STATS` with `BY` clause efficiently

4. **Reranker Integration:**
   - Limit initial query to 50-100 documents
   - Let JinaAI reranker reduce to top 5-10

---

## Integration with Agent Builder

These queries are automatically invoked by the Agent Builder when users ask:
- "Why did [test_name] fail?"
- "Show me flaky tests"
- "Correlate failures with system metrics"
- "What's the RCA for build [build_id]?"

The agent uses RAG to:
1. Execute relevant ES|QL query
2. Pass results to JinaAI reranker
3. Generate natural language RCA report

---

**Last Updated:** January 2026  
**ES|QL Version:** Elasticsearch 8.17+
