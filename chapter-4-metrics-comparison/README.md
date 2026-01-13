# Chapter 4: Metrics Collection & Visualization

## 📋 Overview

In this chapter, you will learn the critical conceptual differences between Azure platform metrics and Prometheus metrics. Understanding when to use which metric source is essential for making informed decisions about monitoring strategies. This knowledge is often overlooked but extremely valuable in real-world scenarios.

## 🎯 Learning Objectives

After completing this chapter, you will:

- Understand the difference between platform metrics and Prometheus metrics
- Compare Azure Monitor CPU % with Prometheus CPU metrics
- Understand sampling intervals and granularity
- Know which metric to use for different use cases
- Understand the cost implications of different metric sources
- Make informed decisions about metric collection strategies

## ⏱️ Estimated Time

15-20 minutes

## 📝 Step-by-Step Instructions

### Step 1: Understanding the Two Metric Pipelines

Azure provides two distinct paths for collecting metrics from AKS:

```
┌─────────────────────────────────────────────────────────────┐
│                     AKS Cluster                              │
│                                                              │
│  ┌──────────┐                                               │
│  │  Node    │                                               │
│  │ (Kubelet)│                                               │
│  └─────┬────┘                                               │
│        │                                                     │
│        └──▶ Pod                                             │
│              └──▶ Container                                 │
│                                                              │
└───────┬──────────────────────────────────────┬──────────────┘
        │                                      │
        │                                      │
        ▼                                      ▼
┌───────────────┐     ┌─────────────────────────────────┐
│  AMA-Logs     │     │  AMA-Metrics (Prometheus)        │
│  Agent        │     │  - Node Exporter                 │
│               │     │  - cAdvisor                      │
│  Collects:    │     │  - kube-state-metrics            │
│  - Logs       │     │                                  │
│  - Events     │     │  Collects:                       │
└───────┬───────┘     │  - Container metrics             │
        │             │  - Node metrics                  │
        │             │  - Kubernetes state              │
        │             └────────────┬────────────────────┘
        │                          │
        ▼                          ▼
┌───────────────┐     ┌─────────────────────────────────┐
│ Log Analytics │     │ Azure Monitor Workspace          │
│ Workspace     │     │ (Prometheus)                     │
│               │     │                                  │
│ - Logs (KQL)  │     │ - Metrics (PromQL)               │
│               │     │ - Cloud-Native Metrics           │
└───────────────┘     └─────────────────────────────────┘
        
                ┌─────────────────────────────────┐
                │  Azure Monitor Metrics           │
                │  (Separate Pipeline)             │
                │                                  │
                │ Platform Metrics:                │
                │ - CPU %                          │
                │ - Memory %                       │
                │ - Network bytes                  │
                │                                  │
                │ ✓ No ingestion cost              │
                │ ✓ Not in Log Analytics           │
                │ ✓ Pre-aggregated by Azure        │
                └─────────────────────────────────┘
```

**Key Insights from Diagram:**
- **Node → Pod → Container** is the correct hierarchy
- **AMA-Logs** collects logs/events → Log Analytics (KQL queries)
- **AMA-Metrics** collects Prometheus metrics → Azure Monitor Workspace (PromQL queries)
- **Platform Metrics** flow directly to Azure Monitor Metrics (separate pipeline, no ingestion cost)

### Step 2: Platform Metrics vs Prometheus Metrics

Let's understand the key differences:

| Aspect             | Platform Metrics (Azure Monitor)              | Prometheus Metrics                             |
| ------------------ | --------------------------------------------- | ---------------------------------------------- |
| **Source**         | Azure Resource Provider (ARM)                 | Prometheus exporters (cAdvisor, node-exporter) |
| **Collection**     | Direct to Azure Monitor Metrics               | AMA-Metrics agent → Azure Monitor Workspace    |
| **Aggregation**    | Pre-aggregated by Azure                       | Raw samples, aggregate in queries              |
| **Sampling**       | 1-minute intervals (fixed)                    | 15-30 seconds (configurable)                   |
| **Retention**      | 31-93 days (extendable to 730 days)           | 18 months (fixed per Azure Monitor Workspace)  |
| **Granularity**    | Lower (averaged)                              | Higher (raw values)                            |
| **Cardinality**    | Lower (fewer labels)                          | Higher (many labels)                           |
| **Query Language** | KQL (Kusto) for logs; native for metrics      | PromQL                                         |
| **Best For**       | Azure Portal dashboards, basic monitoring     | Advanced queries, alerting, custom dashboards  |
| **Cost Model**     | Per time series + API calls (negligible cost) | Per million samples ingested                   |

**Key Cost Insight**: Platform metrics have negligible cost compared to log-based telemetry. They don't count against your Log Analytics ingestion quota and are billed separately under Azure Monitor Metrics pricing (per time series, not per GB).

### Step 3: Comparing CPU Metrics

Let's compare CPU metrics from both sources side-by-side.

#### Platform Metrics: CPU Usage %

In Azure Portal:
1. Navigate to your AKS cluster
2. Go to **Monitoring** → **Metrics**
3. Add metric:
   - Metric Namespace: "Container Insights"
   - Metric: "CPU Usage Percentage"
   - Aggregation: Average
   - Time range: Last 1 hour
   - Time granularity: 1 minute

This shows you CPU usage as a percentage, averaged over 1-minute intervals.

#### Prometheus Metrics: node_cpu_seconds_total

In Grafana → Explore:

```promql
# Average CPU usage across all nodes (as percentage)
avg(
  rate(node_cpu_seconds_total{mode!="idle"}[5m])
) * 100
```

This shows you CPU usage calculated from raw Prometheus samples.

### Step 4: Side-by-Side Comparison

Let's create a dashboard that shows both metrics together.

#### Create Comparison Dashboard in Grafana:

1. Go to Grafana → Dashboards → New Dashboard
2. Add visualization

**Panel 1: Prometheus CPU Metrics**

Azure Platform Metrics aren't directly queryable in Grafana - they're best viewed in Azure Portal. In Grafana, we'll compare Prometheus metrics with Container Insights data from Log Analytics.

**Query (Prometheus data source):**
```promql
# Prometheus: Node CPU usage by node
avg by (instance) (
  rate(node_cpu_seconds_total{mode!="idle"}[5m])
) * 100
```

- Visualization: Time series
- Title: "Prometheus: Node CPU Usage %"
- Legend: `{{instance}}`

**Panel 2: Container Insights Data from Log Analytics**

Add another panel using the Azure Monitor data source.

**Query (Azure Monitor Logs data source):**

```kql
// Container Insights: CPU usage by node (1-minute aggregation)
Perf
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 1m), Computer
| order by TimeGenerated asc
| render timechart
```

> **Note**: The counter uses nanocores (billionths of a CPU core). To convert to percentage for a node with N cores: `(nanocores / (N * 1,000,000,000)) * 100`. For comparison purposes, the raw nanocores value works fine.

### Step 5: Understanding Sampling and Granularity

Let's understand how sampling affects what you see.

#### Sampling Intervals:

**Platform Metrics (Azure Monitor):**
- Collected every 60 seconds
- Pre-aggregated (averaged) by Azure
- You see: "Average CPU over 60 seconds"

**Prometheus Metrics:**
- Scraped every 15-30 seconds (default: 30s)
- Raw samples stored
- You see: "CPU at each scrape interval"

#### Practical Example:

Imagine CPU usage like this:
```
Time:  00:00   00:15   00:30   00:45   01:00
CPU:    10%     90%     10%     10%     10%
```

**What you see:**
- **Azure Platform Metric (1-min average)**: ~30% (averaged over entire minute)
- **Prometheus (15s samples)**: 10%, 90%, 10%, 10%, 10% (actual spikes visible)

**Impact**: Prometheus catches short-lived spikes that platform metrics average out.

#### Query to See Sampling Difference:

In Grafana, compare these two queries:

```promql
# High resolution: per-scrape CPU samples
rate(node_cpu_seconds_total{mode!="idle"}[1m])
```

vs

```promql
# Lower resolution: averaged over 5 minutes
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```

The first shows more detail (and variability), the second is smoother.

### Step 6: Understanding Data Freshness

How quickly do you see new data?

#### Test Data Freshness:

```bash
# Deploy a new pod
kubectl run freshness-test --image=nginx:1.25 -n demo-apps

# Immediately query both sources
```

**In Azure Portal (Platform Metrics):**
1. Go to Container Insights → Containers
2. Look for `freshness-test` pod
3. Note the delay: ~2-3 minutes

**In Grafana (Prometheus):**
```promql
kube_pod_info{pod="freshness-test"}
```

Note the delay: ~30-60 seconds

**Conclusion**: Prometheus metrics appear faster due to more frequent scraping.

### Step 7: Choosing the Right Metric

Now let's understand when to use which metric source.

#### Use Platform Metrics (Azure Monitor / Container Insights) When:

✅ **Viewing in Azure Portal**
- Stakeholders prefer Azure Portal
- Integrating with other Azure services

✅ **Long-term trend analysis**
- Historical analysis over months
- Capacity planning

✅ **Cost-sensitive environments**
- First 5GB/month is free
- Lower cardinality = lower cost

✅ **Basic monitoring needs**
- Simple dashboards
- Standard alerts

#### Use Prometheus Metrics When:

✅ **Need high granularity**
- Catching short-lived spikes
- Detailed performance analysis

✅ **Advanced queries required**
- Complex aggregations
- Multi-dimensional filtering

✅ **Standardization across platforms**
- Multi-cloud or hybrid deployments
- Kubernetes-native tooling

✅ **Custom metrics**
- Application-specific metrics
- ServiceMonitor / PodMonitor patterns

✅ **Grafana dashboards preferred**
- Team is familiar with PromQL
- Rich visualization needs

### Step 8: Practical Use Case Examples

Let's work through real scenarios.

#### Scenario 1: "Is my node overloaded?"

**Quick check (Platform Metrics):**
- Azure Portal → Container Insights → Nodes
- Look at CPU % column
- Good for: Quick glance, high-level view

**Detailed analysis (Prometheus):**
```promql
# Per-node CPU breakdown by mode (user, system, etc.)
sum by (instance, mode) (
  rate(node_cpu_seconds_total[5m])
)
```
- Good for: Understanding what's consuming CPU (user vs system)

#### Scenario 2: "Show me a dashboard for management"

**Use Platform Metrics:**
- Azure Portal → Container Insights
- Built-in views, familiar interface
- No extra tools needed

#### Scenario 3: "Alert on CPU usage vs limits"

**Use Prometheus:**
```promql
# CPU usage vs limits (throttling alternative)
(
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{namespace="demo-apps"}[5m])
  )
  / 
  sum by (namespace, pod) (
    kube_pod_container_resource_limits{namespace="demo-apps", resource="cpu"}
  )
) > 0.8
```
- Platform metrics don't expose CPU limits or detailed container metrics
- Prometheus is required for CPU usage vs limits monitoring

> **Note**: Azure Managed Prometheus doesn't include `container_cpu_cfs_throttled_seconds_total`. Use CPU usage vs limits as an alternative to detect potential throttling.

#### Scenario 4: "Historical analysis: CPU trends over 3 months"

**Use Container Insights Data:**
- Lower storage cost for long retention
- Pre-aggregated data sufficient for trends
- KQL in Log Analytics:
  ```kql
  Perf
  | where TimeGenerated > ago(90d)
  | where CounterName == "cpuUsageNanoCores"
  | where ObjectName == "K8SNode"
  | summarize avg(CounterValue) by bin(TimeGenerated, 1d), Computer
  | render timechart
  ```

### Step 9: Cost Implications

Let's understand the cost difference.

#### Container Insights (Log Analytics) Cost:

```kql
// In Log Analytics Workspace
Usage
| where TimeGenerated > ago(7d)
| where IsBillable == true
| where DataType in ("Perf", "KubePodInventory", "KubeNodeInventory")
| summarize TotalGB = sum(Quantity) / 1000 by DataType
| extend CostPerGB = 2.30  // Approximate (varies by region)
| extend EstimatedCost = iff(TotalGB > 5, (TotalGB - 5) * CostPerGB, 0.0)
| project DataType, TotalGB, EstimatedCost
```

**Typical cost:** $70-200/month after the 5 GB free tier (per-GB ingestion model)

#### Prometheus Metrics Cost:

Prometheus costs are based on samples ingested. Here are typical costs by cluster size:

| Cluster Type     | Samples/sec | Samples/min    | Monthly Cost |
| ---------------- | ----------- | -------------- | ------------ |
| Small (dev/test) | 1-3k/s      | ~100k/min      | $150-$300    |
| Medium prod      | 3-8k/s      | ~300-500k/min  | $300-$900    |
| Large prod       | 10-20k/s    | ~600k-1.2M/min | $900-$2,500  |

> **⚠️ These are order-of-magnitude estimates.** Actual cost depends on scrape interval, label cardinality, region, and commitment tier. Use Azure Cost Management for precise costs.

> **Why higher cost?** Prometheus collects more metrics, at higher frequency (15-30 sec vs 1 min), with more labels (higher cardinality).

#### Query Current Prometheus Sample Rate:

In Grafana Explore:

```promql
# Samples ingested per second
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

Use this to estimate your monthly cost based on the table above.

### Step 10: Hybrid Approach - Best of Both Worlds

Most organizations use both for different purposes:

**Recommended Strategy:**

#### 1. Keep Container Insights enabled (Platform Metrics & basic telemetry)

Use this for:
- **Azure Portal native experience**
  - AKS Insights, VM insights, resource health
  
- **Low-cost, long-term retention**
  - Aggregated metrics with predictable cost
  
- **Baseline operational monitoring**
  - Node availability, pod health, capacity signals
  
- **Platform-native alerting**
  - Autoscale signals, simple alerts

✅ Metrics are stored in Azure Monitor Metrics  
✅ Do not consume Log Analytics GB ingestion  
✅ Very low, predictable cost

#### 2. Enable Prometheus (Cloud-native metrics – Managed Prometheus)

Use this for:
- **Grafana dashboards**
  - SLO / SLA views, RED/USE metrics
  
- **Advanced alerting**
  - PromQL-based alerts, burn rates
  
- **Developer self-service**
  - Custom app metrics, labels, dimensions
  
- **High-resolution analysis**
  - 15s–30s granularity where needed

#### 3. Optimize Prometheus collection (Chapter 5)
   - Adjust scrape intervals
   - Filter unnecessary metrics
   - Balance cost vs granularity

## 🎓 Key Takeaways

By now you should understand:

1. **Two metric pipelines**: Platform metrics (Azure Monitor) and Prometheus metrics serve different purposes
2. **Sampling matters**: Platform metrics (60s) vs Prometheus (30s) affects spike detection
3. **Granularity trade-offs**: Higher granularity = better insights, but higher cost
4. **Choose the right tool**:
   - Azure Portal & basic needs → Platform metrics
   - Grafana & advanced queries → Prometheus metrics
5. **Cost implications**: Prometheus provides more data but costs more
6. **Hybrid approach**: Use both for different use cases

## ✅ Verification Checklist

Ensure you've completed:

- [ ] Understood the two metric collection pipelines
- [ ] Compared CPU metrics in Azure Portal vs Grafana
- [ ] Explored sampling and granularity differences
- [ ] Tested data freshness for both sources
- [ ] Learned when to use platform metrics vs Prometheus
- [ ] Worked through practical use case scenarios
- [ ] Understood cost implications of each approach
- [ ] Recognized the value of a hybrid strategy

## 🐛 Troubleshooting

### Issue: Metrics don't match between sources

**Explanation**: This is expected!
- Platform metrics are pre-aggregated over 60s
- Prometheus samples are more frequent (30s)
- Different aggregation methods
- This is not a bug - it's the nature of the two systems

### Issue: Prometheus shows more spikes

**Explanation**: This is correct!
- Prometheus captures higher frequency samples
- Short-lived spikes are visible
- Platform metrics average them out
- Both are "correct" - just different granularities

## 📚 Additional Resources

- [Azure Monitor Metrics Overview](https://docs.microsoft.com/azure/azure-monitor/essentials/data-platform-metrics)
- [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/)
- [Container Insights Cost Optimization](https://docs.microsoft.com/azure/azure-monitor/containers/container-insights-cost)
- [Azure Monitor Pricing](https://azure.microsoft.com/pricing/details/monitor/)

---

## ⏭️ Next Steps

Excellent! You now understand which metric to use and why. In [Chapter 5: Cost Optimization for Metrics Ingestion](../chapter-5-cost-optimization/README.md), you'll learn how to control and reduce observability costs - a skill that's rarely taught but extremely valuable.

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
