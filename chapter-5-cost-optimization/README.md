# Chapter 5: Cost Optimization for Metrics Ingestion

## 📋 Overview

In this final chapter, you will learn how to optimize observability costs - a skill that is rarely taught but extremely valuable in professional environments. Uncontrolled metrics ingestion can quickly become expensive, and knowing how to balance cost with observability needs gives you "senior-level" skills.

## 🎯 Learning Objectives

After completing this chapter, you will:

- Identify the top 5 most expensive metrics in your environment
- Understand what drives metrics costs (cardinality, scrape frequency, retention)
- Adjust scrape intervals and retention policies
- Filter out unnecessary metrics
- Calculate cost savings before and after optimization
- Implement cost-effective observability strategies
- Know how to make informed decisions about metrics collection

## ⏱️ Estimated Time

15-20 minutes

## 📝 Step-by-Step Instructions

### Step 1: Understanding What Drives Metrics Costs

Metrics costs are driven by three main factors:

```
Cost = (Number of Metrics) × (Cardinality) × (Scrape Frequency) × (Retention Period)
```

Let's break down each factor:

#### 1. Number of Metrics
- Each unique metric name (e.g., `node_cpu_seconds_total`)
- More metrics = higher cost

#### 2. Cardinality
- Number of unique label combinations for a metric
- Example: `node_cpu_seconds_total{instance="node1", mode="user"}` vs `{instance="node2", mode="system"}`
- High cardinality = exponentially higher cost

#### 3. Scrape Frequency
- How often Prometheus collects metrics (default: 30s)
- More frequent = more samples = higher cost

#### 4. Retention Period
- How long metrics are stored (default: 18 months)
- Azure Managed Prometheus retention is currently fixed at 18 months (cannot be reduced per workspace)
- Longer retention = higher storage cost

### Step 2: Current Cost Baseline

Let's establish your current metrics ingestion rate and estimated cost.

#### Query Current Sample Ingestion Rate:

In Grafana → Explore:

```promql
# Count scrape targets (proxy for active time series)
count(up)
```

> **Note**: Azure Managed Prometheus doesn't support `{__name__=~".+"}` regex queries. We'll use scrape target count as an approximation. For exact metrics, check Azure Portal → Azure Monitor Workspace → Metrics.

This returns the number of scrape targets. Let's say this returns `10` targets.

**Estimate time series based on targets:**
- Typical AKS cluster with Container Insights: ~500-1000 time series per target
- For 10 targets: approximately **5,000-10,000 active time series**

**Estimate sample rate:**
- With default 30-second scrape interval: `5000 time series × (60s / 30s) = 10,000 samples/minute`
- That's approximately **167 samples/second**

**Calculate monthly ingestion:**
```
10,000 samples/min × 60 min × 24 hours × 30 days
= 432,000,000 samples/month
= 432 million samples/month
```

**Estimate cost (Azure pricing):**
- Azure Managed Prometheus: approximately **$0.50 per million samples ingested**
- Cost: 432 × $0.50 = **~$216/month**

> **Important**: Pricing varies significantly by region and commitment tier. Check [Azure Monitor pricing](https://azure.microsoft.com/pricing/details/monitor/) for your region. Use Azure Cost Management for actual costs.

#### Alternative: Check Azure Portal for Actual Usage

For accurate ingestion metrics:
1. Go to **Azure Portal** → **Azure Monitor Workspace**
2. Select your workspace
3. Go to **Metrics** → **Ingestion** to see actual samples ingested

#### Query Total Number of Metrics:

```promql
# Count specific metric families to estimate coverage
count(container_memory_working_set_bytes)
+ count(container_cpu_usage_seconds_total) 
+ count(kube_pod_info)
+ count(node_cpu_seconds_total)
```

This gives you a sense of key metrics being collected.

> **For comprehensive metrics inventory**: Use Azure Portal → Azure Monitor Workspace → Metrics to browse all available metrics.

#### Query High-Cardinality Metrics:

```promql
# Check cardinality of specific metrics
count(container_network_receive_bytes_total)
```

Then manually check other common high-cardinality metrics:
- `container_network_*`
- `kube_pod_labels` 
- `node_network_*`

> **Note**: Azure Managed Prometheus doesn't support `{__name__=~"regex"}` queries. Check specific metrics individually or use Azure Portal to view metrics inventory.

### Step 3: Identifying Top 5 Most Expensive Metrics

Let's find which metrics are consuming the most resources.

#### Check Cardinality of Common Metrics:

Manually check cardinality of typical high-cardinality metrics:

```promql
# Container network metrics (often high cardinality)
count(container_network_receive_bytes_total)
```

```promql
# Pod info metrics
count(kube_pod_info)
```

```promql  
# Node network metrics
count(node_network_receive_bytes_total)
```

```promql
# Container CPU metrics (per container)
count(container_cpu_usage_seconds_total)
```

> **Note**: Not all Kubernetes metrics are available in Azure Managed Prometheus. API server and etcd metrics may not be exposed.

**Typical high-cardinality culprits in Azure Managed Prometheus:**
- `container_network_*`: Many network interface labels
- `kube_pod_info`: One time series per pod
- `node_network_*`: Per-interface metrics
- `container_cpu_*` / `container_memory_*`: Per-container granularity

> **Note**: API server (`apiserver_*`) and etcd (`etcd_*`) metrics are typically not available in Azure Managed Prometheus for AKS.

#### Create Dashboard to Track Costs:

In Grafana, create a new dashboard:

**Panel 1: Common Metric Cardinality**

Create multiple queries in one panel:

```promql
# Query A: Container network metrics
count(container_network_receive_bytes_total)
```

```promql
# Query B: Pod info
count(kube_pod_info)
```

```promql
# Query C: Node network metrics  
count(node_network_receive_bytes_total)
```

- Visualization: **Bar chart** or **Stat**
- Title: "Cardinality of High-Volume Metrics"

**Panel 2: Scrape Target Count**

```promql
count(up)
```

- Visualization: **Gauge**
- Title: "Active Scrape Targets"

**Panel 3: Key Metrics Health Check**

```promql
# Verify key metrics are present
count(container_memory_working_set_bytes) > 0
```

- Visualization: **Stat**  
- Title: "Container Metrics Available"

> **Note**: For actual ingestion metrics and costs, check:
> - Azure Portal → Azure Monitor Workspace → Metrics → Ingestion (for sample counts)
> - Azure Cost Management → Cost Analysis (for actual spending)

Save this dashboard as "Cost Monitoring".

### Step 4: Adjusting Scrape Intervals

The default scrape interval is 30 seconds. Let's adjust it to reduce costs.

#### View Current Scrape Configuration:

```bash
# Check if custom metrics configuration exists
kubectl get configmap -n kube-system ama-metrics-settings-configmap -o yaml

# If it doesn't exist yet, that's normal - we'll create it below
```

#### Create ConfigMap to Adjust Scrape Intervals:

Azure Managed Prometheus supports per-target scrape interval configuration:

```bash
cat <<EOF > metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ama-metrics-settings-configmap
  namespace: kube-system
data:
  prometheus-collector-settings: |
    default-scrape-settings-enabled: true
    default-targets-scrape-interval-settings:
      # Adjust scrape intervals for predefined targets
      kubelet: "60s"          # default: 30s (pods, nodes, containers)
      cadvisor: "60s"         # default: 30s (container metrics)
      kubestate: "60s"        # default: 30s (Kubernetes state)
      nodeexporter: "60s"     # default: 30s (node metrics)
EOF

# Apply the configuration
kubectl apply -f metrics-config.yaml

# Restart ama-metrics pods to pick up new config
kubectl rollout restart daemonset ama-metrics-node -n kube-system
kubectl rollout restart deployment ama-metrics -n kube-system

# Wait for pods to restart
kubectl rollout status daemonset ama-metrics-node -n kube-system
kubectl rollout status deployment ama-metrics -n kube-system
```

> **Impact**: Changing from 30s to 60s across all targets reduces sample count by ~50%, significantly cutting costs!

> **Important**: This configures scrape intervals per target/job, not a global cluster setting. Each target (kubelet, cadvisor, etc.) is configured individually.

#### Trade-offs:

| Scrape Interval   | Pros                                 | Cons                        |
| ----------------- | ------------------------------------ | --------------------------- |
| **15s**           | High granularity, catch short spikes | 2x cost vs 30s              |
| **30s** (default) | Good balance                         | Standard cost               |
| **60s**           | 50% cost reduction                   | May miss short-lived issues |
| **120s**          | 75% cost reduction                   | Significant loss of detail  |

**Recommendation**: 
- Production critical apps: 30s
- Development/test: 60s
- Non-critical workloads: 120s

### Step 5: Using Minimal Ingestion Profile

Azure Managed Prometheus supports a "minimal ingestion profile" that collects only essential metrics, significantly reducing costs.

#### Option 1: Enable Minimal Ingestion Profile (Recommended)

This profile collects only the most commonly used metrics:

```bash
cat <<EOF > metrics-minimal-profile.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ama-metrics-settings-configmap
  namespace: kube-system
data:
  prometheus-collector-settings: |
    default-scrape-settings-enabled: true
    default-targets-scrape-interval-settings:
      kubelet: "60s"
      cadvisor: "60s"
      kubestate: "60s"
      nodeexporter: "60s"
    
    # Enable minimal ingestion profile
    minimal_ingestion_profile: true
EOF

kubectl apply -f metrics-minimal-profile.yaml

# Restart metrics collection
kubectl rollout restart daemonset ama-metrics-node -n kube-system
kubectl rollout restart deployment ama-metrics -n kube-system
```

> **Impact**: Minimal ingestion profile typically reduces metrics by 50-70%, significantly cutting costs while retaining essential observability.

#### Option 2: Use Keep Lists for Specific Metrics

If you need more control, specify exactly which metrics to keep per target:

```bash
cat <<EOF > metrics-keep-list.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ama-metrics-settings-configmap
  namespace: kube-system
data:
  prometheus-collector-settings: |
    default-scrape-settings-enabled: true
    default-targets-scrape-interval-settings:
      kubelet: "60s"
      cadvisor: "60s"
      kubestate: "60s"
      nodeexporter: "60s"
    
    # Specify which metrics to keep (drops all others)
    default-targets-metrics-keep-list:
      kubelet: |
        kubelet_volume_stats_used_bytes
        kubelet_node_name
      cadvisor: |
        container_cpu_usage_seconds_total
        container_memory_working_set_bytes
        container_network_receive_bytes_total
        container_network_transmit_bytes_total
      kubestate: |
        kube_pod_info
        kube_pod_status_phase
        kube_pod_container_status_restarts_total
        kube_pod_container_resource_limits
        kube_node_status_condition
      nodeexporter: |
        node_cpu_seconds_total
        node_memory_MemAvailable_bytes
        node_filesystem_avail_bytes
EOF

kubectl apply -f metrics-keep-list.yaml

# Restart metrics collection
kubectl rollout restart daemonset ama-metrics-node -n kube-system
kubectl rollout restart deployment ama-metrics -n kube-system
```

> **Note**: Keep lists define what to **include** - all other metrics from that target are dropped.

#### Verify Reduced Metrics:

```bash
# Wait for pods to restart
kubectl rollout status daemonset ama-metrics-node -n kube-system

# In Grafana, check scrape target count (should be same)
# count(up)

# But individual metric counts should be lower
# count(container_cpu_usage_seconds_total)
```

> **Impact**: Using keep lists with minimal ingestion can reduce ingestion by 20-40% or more depending on which metrics you need.

### Step 6: Reducing High-Cardinality Labels (Default Targets vs Custom Metrics)

High-cardinality labels increase time series count and drive ingestion costs.

#### Understanding Azure Managed Prometheus Limitations:

**Important**: For AKS default targets (kubelet, cadvisor, kube-state-metrics, node-exporter):

✅ **Supported approaches:**
- Metric allowlists (keep lists) - already covered in Step 5
- Minimal ingestion profile
- Scrape interval adjustments

❌ **Not supported:**
- `metric_relabel_configs` for default targets
- `labeldrop` / `labelkeep` for default targets
- Regex-based label filtering for default targets

> **Why?** Azure restricts this to protect built-in dashboards, recording rules, and alerts shipped with the managed experience.

#### Option A: Enable Minimal Ingestion Profile (Recommended)

The easiest way to reduce label cardinality is to use Microsoft's minimal ingestion profile:

```bash
cat <<EOF > minimal-ingestion.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ama-metrics-settings-configmap
  namespace: kube-system
data:
  prometheus-collector-settings: |
    default-scrape-settings-enabled: true
    minimal-ingestion-profile: true
    
    default-targets-scrape-interval-settings:
      kubelet: "60s"
      cadvisor: "60s"
      kubestate: "60s"
      nodeexporter: "60s"
EOF

kubectl apply -f minimal-ingestion.yaml
kubectl rollout restart daemonset ama-metrics-node -n kube-system
kubectl rollout restart deployment ama-metrics -n kube-system
```

**What minimal ingestion profile removes:**
- Verbose kube-state-metrics labels
- Histogram buckets
- High-cardinality edge metrics

> **Impact**: Can reduce ingestion by 50-70% while keeping essential monitoring capabilities.

#### Option B: Label Filtering for Custom Application Metrics Only

If you have custom application metrics (your own exporters), you can apply label filtering:

```yaml
# Custom scrape job with label filtering (for your own apps only)
apiVersion: azmonitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
  namespace: demo-apps
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
  - port: metrics
    interval: 60s
    # Label filtering works for custom metrics
    metricRelabelings:
    - action: labeldrop
      regex: "pod_uid|container_id|controller_revision_hash"
```

> **Rule**: AKS default metrics = filter by allowlist only. Custom metrics = full Prometheus relabeling allowed.

#### Verify Impact:

```bash
# Check scrape targets still exist
# In Grafana:
count(up)

# Verify specific metrics have fewer labels
# In Grafana:
count(container_cpu_usage_seconds_total)
```

### Step 7: Adjusting Retention Period

**Note**: Retention for Azure Managed Prometheus is currently fixed at 18 months and cannot be adjusted per workspace. However, you can:

1. **Use recording rules** to pre-aggregate old data (reduces query cost and dashboard complexity, not ingestion cost)
2. **Export to cold storage** (Azure Storage) for long-term retention
3. **Delete old data** using Azure Monitor API (if supported in your region)

### Step 8: Calculate Cost Savings

Let's calculate savings from our optimizations.

#### Before Optimization:

```promql
# Baseline: Count scrape targets
count(up)
```

Let's say: **10 scrape targets**

Estimate for typical AKS cluster:
- Time series per target: ~800 average
- Total time series: 10 × 800 = **8,000 time series**

With 30-second scrape interval:
- Samples per minute: 8,000 × 2 = **16,000 samples/min**
- Monthly samples: 16,000 × 60 × 24 × 30 = 691 million
- Estimated cost (at ~$0.50/M samples): **~$345/month**

#### After Optimization:

Apply all optimizations:
1. Scrape interval: 30s → 60s (**-50% samples**)
2. Minimal ingestion profile or keep lists (**-50-70% metrics**)
3. Combined effect: Fewer metrics collected less frequently

**After optimizations:**
- Scrape interval change: 30s → 60s (**-50% samples**)
- Minimal ingestion profile: (**-60% metrics** - typical)
- Combined reduction: 8,000 × 0.4 = **3,200 time series**

**New estimate:**
- Time series: 8,000 × 0.4 = **3,200 time series**

With 60-second scrape interval:
- Samples per minute: 3,200 × 1 = **3,200 samples/min**
- Monthly samples: 3,200 × 60 × 24 × 30 = 138 million
- Estimated cost (at ~$0.50/M samples): **~$69/month**

**Savings: ~$276/month (80% reduction!)**

> **Important**: These are order-of-magnitude estimates. Always verify actual costs using Azure Cost Management. Pricing varies by region and commitment tier.

#### Create Savings Dashboard:

In Grafana:

**Panel 1: Scrape Targets Over Time**

```promql
# Query scrape target count
count(up)
```

Add annotations for when you applied optimizations.

**Panel 2: Estimated Cost Trend**

Since Azure Managed Prometheus doesn't expose internal metrics, monitor this in **Azure Portal**:
1. Go to Azure Monitor Workspace → **Metrics**
2. Select metric: **Samples Ingested**
3. View cost trends over time

### Step 9: Monitoring the Impact

After applying optimizations, monitor to ensure you haven't lost critical observability.

#### Checklist: What to Verify

```bash
# 1. Verify scrape interval changed
kubectl get configmap ama-metrics-settings-configmap -n kube-system -o yaml | grep interval

# 2. Check scrape target count
# In Grafana:
count(up)

# 3. Verify your dashboards still work
# Navigate through your dashboards and ensure data is present

# 4. Test alerts still fire
# Trigger a test alert (from Chapter 3) and verify it fires
```

#### Create Optimization Impact Dashboard:

Create a dashboard with these panels:

1. **Scrape Targets Over Time**
   ```promql
   count(up)
   ```

2. **Key Metrics Count**
   ```promql
   count(container_memory_working_set_bytes)
   ```

3. **Network Metrics Cardinality**
   ```promql
   count(container_network_receive_bytes_total)
   ```

> **For actual cost tracking**: Use Azure Portal → Azure Monitor Workspace → Metrics → Ingestion to view real ingestion data and cost.

Save as "Cost Optimization Impact".

### Step 10: Best Practices for Cost-Effective Observability

Let's consolidate learnings into actionable best practices.

#### ✅ DO:

1. **Start with defaults, optimize based on actual needs**
   - Don't prematurely optimize
   - Observe what you actually use

2. **Use recording rules for frequently-queried aggregations**
   - Pre-aggregate at collection time
   - Reduces query cost and dashboard complexity (not ingestion cost)

3. **Adjust scrape intervals by namespace**
   - Critical: 30s
   - Non-critical: 60-120s

4. **Filter metrics at the source**
   - Drop before storing (most efficient)
   - Don't collect what you don't use

5. **Monitor your metrics costs**
   - Set up cost dashboards
   - Alert on unexpected cost increases

6. **Use labels wisely**
   - Avoid high-cardinality labels (e.g., timestamps, UUIDs)
   - Keep only labels you actually filter by

#### ❌ DON'T:

1. **Don't collect everything "just in case"**
   - Storage and query costs add up

2. **Don't use extremely high cardinality labels**
   - Example: Don't use customer_id as a label if you have millions of customers

3. **Don't set scrape interval too low unless necessary**
   - 15s is rarely needed
   - 30s is usually sufficient

4. **Don't keep metrics you never query**
   - Review unused metrics quarterly
   - Drop if not used in 90 days

5. **Don't forget to clean up test resources**
   - Temporary deployments should be removed

#### Cost Optimization Checklist:

```
Optimization Opportunities:
□ Scrape interval optimized per workload type
□ High-cardinality metrics identified and filtered
□ Unused metrics dropped
□ Recording rules created for common aggregations
□ Label cardinality reviewed and optimized
□ Cost monitoring dashboard created
□ Quarterly review process established
□ Team trained on cost-aware metrics collection
```

### Step 11: Long-Term Cost Management Strategy

Establish processes for ongoing cost management:

#### 1. Monthly Review Process

```
Monthly Metrics Review:
1. Check cost dashboard
2. Identify new high-cardinality metrics
3. Review metrics usage (are we querying it?)
4. Adjust scrape intervals if needed
5. Document changes
```

#### 2. Alert on Unexpected Metric Growth

Create an alert for unexpected changes:

```promql
# Alert if scrape target count increases by >20%
(
  count(up)
  /
  (count(up offset 1d) > 0)
) > 1.2
```

> **Note**: For production cost alerts, use Azure Monitor cost alerts on the Azure Monitor Workspace resource directly.

#### 3. Tag and Track by Team

Use labels to track which team's metrics are most expensive:

```promql
# Cost by namespace (proxy for team)
topk(10,
  sum by (namespace) (
    count(kube_pod_info)
  )
)
```

> **Note**: This estimates time series per namespace. For more accurate tracking, sum multiple representative metrics: `count(container_cpu_usage_seconds_total) + count(kube_deployment_status_replicas) + count(node_cpu_seconds_total)`

## 🎓 Key Takeaways

By now you should understand:

1. **What drives costs**: Metrics count × cardinality × scrape frequency × retention
2. **How to identify expensive metrics**: High cardinality is the main culprit
3. **Optimization techniques**:
   - Adjust scrape intervals
   - Drop unnecessary metrics
   - Optimize label cardinality
   - Use recording rules
4. **Calculate ROI**: 50-70% cost reduction is achievable without losing visibility
5. **Cost-aware observability**: Balance cost with monitoring needs
6. **Continuous improvement**: Regular reviews and optimization

**The skill you've developed**: You can now control observability costs while maintaining effective monitoring - a senior-level capability.

## ✅ Verification Checklist

Ensure you've completed:

- [ ] Calculated current baseline metrics ingestion rate
- [ ] Identified top 5 most expensive metrics
- [ ] Adjusted scrape interval from 30s to 60s
- [ ] Configured metric drop rules for unnecessary metrics
- [ ] Optimized high-cardinality labels
- [ ] Calculated cost savings before and after optimization
- [ ] Created cost monitoring dashboard
- [ ] Verified dashboards and alerts still function after optimization
- [ ] Documented optimization decisions
- [ ] Established monthly review process

## 🧹 Final Cleanup

Now that the lab is complete, clean up all resources:

```bash
# Delete the entire resource group (includes AKS, Prometheus, Grafana, Log Analytics)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

# Verify deletion started
az group list --output table | grep $RESOURCE_GROUP

# Delete local files (optional)
cd ..
rm -rf aksmonitoringlabs-part1/
```

> **Important**: This will delete ALL resources created during the lab. Ensure you've saved any dashboards, queries, or configurations you want to keep.

## 🐛 Troubleshooting

### Issue: ConfigMap changes not taking effect

**Solution**: Restart the metrics collection pods:
```bash
kubectl rollout restart daemonset ama-metrics-node -n kube-system
kubectl rollout restart deployment ama-metrics-ksm -n kube-system
kubectl rollout status daemonset ama-metrics-node -n kube-system
```

### Issue: Sample rate not decreasing after changes

**Solution**: Wait 5-10 minutes for changes to propagate, then verify config:
```bash
kubectl describe configmap ama-metrics-settings-configmap -n kube-system
```

### Issue: Dashboards missing data after optimization

**Solution**: 
- Check if you dropped a metric that a dashboard uses
- Review drop rules and ensure critical metrics are not filtered
- Revert specific drop rules if needed

## 📚 Additional Resources

- [Azure Monitor Pricing](https://azure.microsoft.com/pricing/details/monitor/)
- [Prometheus Best Practices: Metrics and Labels](https://prometheus.io/docs/practices/naming/)
- [Reducing Prometheus Costs](https://last9.io/blog/reduce-prometheus-costs/)
- [High Cardinality in Prometheus](https://www.robustperception.io/cardinality-is-key)

---

## 🎉 Congratulations!

You've completed the AKS Monitoring & Observability lab! You now have practical skills in:

✅ **Azure Monitor & Container Insights**: Finding issues when AKS misbehaves
✅ **Prometheus & Grafana**: Reading and building cloud-native dashboards
✅ **PromQL & Alerting**: Writing queries and creating actionable alerts
✅ **Metrics Strategy**: Choosing the right metric for the right use case
✅ **Cost Optimization**: Controlling observability costs effectively

These skills are highly valued in the market. You're now equipped to:
- Monitor production Kubernetes clusters
- Debug performance issues
- Create meaningful alerts
- Build informative dashboards
- Optimize monitoring costs

**Next steps to continue learning:**
- Practice writing more PromQL queries
- Build custom dashboards for real applications
- Implement these patterns in your own projects
- Share your knowledge with your team

Thank you for completing this lab! 🚀

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
