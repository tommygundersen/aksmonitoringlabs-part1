# Chapter 1: Azure Monitor + Container Insights

## 📋 Overview

In this chapter, you will learn how to use Azure Monitor and Container Insights to observe your AKS cluster's health and performance. This is the foundation of observability in Azure - the default monitoring solution available in almost all Azure environments.

> **⏰ Important: Data Latency**  
> Container Insights collects performance data every 1-3 minutes and logs continuously, but there's typically a **2-3 minute delay** between an event occurring in your cluster and when it appears in the Azure Portal. For real-time troubleshooting, use `kubectl` commands. For analysis and trends, use Container Insights.

## 🎯 Learning Objectives

After completing this chapter, you will:

- Understand how Container Insights collects data from AKS
- Navigate Azure Monitor and Container Insights dashboards
- Find node CPU pressure, pod restarts, and OOMKills
- Use Activity Logs and Diagnostic Settings
- Create and observe a crashloop scenario
- Know where to look when AKS behaves unexpectedly

## ⏱️ Estimated Time

25-30 minutes

## 📝 Step-by-Step Instructions

### Step 1: Verify Container Insights is Enabled

Container Insights was enabled during cluster creation in Chapter 0. Let's verify it's working.

```bash
# Check monitoring addon status
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "addonProfiles.omsagent.enabled" \
  --output tsv

# Should return: true

# Check the monitoring agents are running
kubectl get pods -n kube-system | grep ama-logs
```

You should see `ama-logs` pods running on each node. These agents collect logs, events, and performance data from your cluster and send them to Log Analytics.

### Step 2: Navigate to Container Insights in Azure Portal

1. Open Azure Portal: https://portal.azure.com
2. Navigate to your AKS cluster: `aks-mon-<YOUR_INITIALS>`
3. In the left menu, under **Monitoring**, click **Insights**
4. You should see the Container Insights dashboard

> **Note**: If you just enabled Container Insights in Chapter 0, it may take 5-10 minutes for data to appear in the dashboard. If you see "No data available", wait a few minutes and refresh.

> **Take a moment to explore**: Look at the different tabs:
> - **Cluster**: Overall cluster health
> - **Nodes**: Individual node performance
> - **Controllers**: Deployment/StatefulSet/DaemonSet health
> - **Containers**: Individual container metrics

### Step 3: Explore Node Metrics

Let's understand what metrics are available at the node level.

> **Note**: While the dashboard becomes visible after 5-10 minutes, CPU and memory metrics data may take **15-20 minutes** to start appearing. If you see nodes listed but no CPU/memory data, continue with the next steps and the data will populate as you progress through the lab.

#### In Azure Portal (Container Insights):

1. Click on the **Nodes** tab
2. You should see your 3 nodes listed
3. Look at the following metrics:
   - **CPU Usage %**: Current CPU utilization
   - **Memory Working Set %**: Current memory usage
   - **Disk Usage %**: Disk consumption

#### Using kubectl:

```bash
# View node resource usage
kubectl top nodes

# Example output:
# NAME                                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# aks-nodepool1-12345678-vmss000000  150m         7%     2048Mi          30%
# aks-nodepool1-12345678-vmss000001  180m         9%     2304Mi          34%
# aks-nodepool1-12345678-vmss000002  120m         6%     1920Mi          28%
```

### Step 4: Finding Node CPU Pressure

Let's create a scenario where a node experiences CPU pressure.

```bash
# Create a CPU stress deployment
cat <<EOF > cpu-stress.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-stress
  namespace: demo-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cpu-stress
  template:
    metadata:
      labels:
        app: cpu-stress
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--cpu", "2", "--timeout", "300s"]  # 5 minutes of CPU stress
        resources:
          requests:
            cpu: "500m"
            memory: "128Mi"
          limits:
            cpu: "1000m"
            memory: "256Mi"
EOF

# Apply the CPU stress
kubectl apply -f cpu-stress.yaml

# Watch the pods start
kubectl get pods -n demo-apps -w
```

#### Observe CPU Pressure in Container Insights:

1. Go back to Azure Portal → Container Insights → **Nodes** tab
2. Wait 2-3 minutes for metrics to update
3. Look for increased CPU usage on nodes
4. Click on a node to see detailed metrics
5. Observe the CPU trend line showing the spike

#### Query CPU Metrics in Log Analytics:

```bash
# Get your Log Analytics Workspace ID
echo $LOG_ANALYTICS_WORKSPACE_ID
```

In Azure Portal:
1. Navigate to your Log Analytics Workspace: `log-aksmon-<YOUR_INITIALS>`
2. Go to **Logs** in the left menu
3. Run this query:

```kql
// Node CPU usage over time
Perf
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

> **Understanding the query**:
> - `Perf`: Performance counter table
> - `ObjectName == "K8SNode"`: Filter to node-level metrics
> - `CounterName == "cpuUsageNanoCores"`: Specific CPU metric
> - `bin(TimeGenerated, 5m)`: 5-minute intervals
> - `render timechart`: Visualize as time series

### Step 5: Finding Pod Restarts

Let's look at how to identify pods that are restarting frequently.

#### Using kubectl:

```bash
# View all pods with restart count
kubectl get pods -n demo-apps -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,STATUS:.status.phase

# View pods sorted by restart count (requires jq)
kubectl get pods -n demo-apps -o json | jq -r '.items[] | "\(.status.containerStatuses[0].restartCount // 0) \(.metadata.name) \(.status.phase)"' | sort -rn
```

#### Using Container Insights:

1. Go to Azure Portal → Your AKS cluster → **Monitoring** → **Insights**
2. Click on the **Containers** tab
3. Look at the **Restarts** column
4. Click on any container to see detailed restart history

#### Query Pod Restarts in Log Analytics:

```kql
// Pods with restart counts
KubePodInventory
| where TimeGenerated > ago(24h)
| summarize MaxRestarts = max(PodRestartCount) by Name, Namespace
| where MaxRestarts > 0
| order by MaxRestarts desc
| take 20
```

### Step 6: Detecting OOMKills (Out-of-Memory Kills)

OOMKills happen when a container exceeds its memory limit. Let's create one intentionally.

```bash
# Create an OOMKill scenario by modifying our memory stress app
cat <<EOF > memory-oom.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-oom
  namespace: demo-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-oom
  template:
    metadata:
      labels:
        app: memory-oom
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]  # Allocate 150MB
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "100Mi"  # Limit is lower than allocation - will cause OOM
            cpu: "200m"
EOF

# Apply
kubectl apply -f memory-oom.yaml

# Watch the pod - it will restart repeatedly due to OOMKill
kubectl get pods -n demo-apps -l app=memory-oom -w
```

After 1-2 minutes, you should see the pod restarting with `OOMKilled` reason.

#### Verify OOMKill in kubectl:

```bash
# Describe the pod to see OOMKilled status
kubectl describe pod -n demo-apps -l app=memory-oom | grep -A 5 "Last State"

# You should see something like:
#     Last State:     Terminated
#       Reason:       OOMKilled
#       Exit Code:    137
```

#### Find OOMKills in Container Insights:

1. Go to Container Insights → **Containers** tab
2. Click on the `memory-oom` container
3. Look at the **Status** and **Restart** information
4. The status reason should show `OOMKilled`

#### Query OOMKills in Log Analytics:

```kql
// Find OOMKilled containers
KubePodInventory
| where TimeGenerated > ago(24h)
| where ContainerStatusReason == "OOMKilled"
| project TimeGenerated, Namespace, Name, ContainerName, ContainerStatusReason
| order by TimeGenerated desc
```

> **📚 Important Learning Point**: The `memory-oom` container likely appears in `KubePodInventory` (showing OOMKilled status) but **not** in the `Perf` table. Why? Because it crashes so quickly that Container Insights never gets a chance to scrape its metrics. Container Insights scrapes performance data every 1-3 minutes, so containers that fail within seconds won't have any performance metrics recorded.

This is normal and happens frequently with:
- Containers that crash immediately on startup
- Containers that hit OOM limits very quickly
- Pods in CrashLoopBackOff that never fully start

#### Check which containers DO have performance metrics:

```kql
// See which containers have actual performance data
Perf
| where TimeGenerated > ago(24h)
| where ObjectName == "K8SContainer"
| where CounterName == "memoryRssBytes"
| distinct InstanceName
```

You should see the `stress` containers from Chapter 0 here, but likely not `memory-oom`.

#### Query memory usage for the stress containers:

```kql
// Recent memory usage for stress containers (these DO have metrics)
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SContainer"
| where CounterName == "memoryRssBytes"
| where InstanceName contains "stress"
| project TimeGenerated, InstanceName, MemoryUsageMB = CounterValue / 1024 / 1024
| order by TimeGenerated desc
| take 100
```

**Key Takeaway**: For detecting fast-failing containers and OOMKills, rely on `KubePodInventory` and `KubeEvents` tables, not the `Perf` table. Performance metrics are only available for containers that run long enough to be scraped.

> **Note**: This detailed query may not return data immediately, as the `Perf` table metrics collection can have a longer delay (5-10 minutes). The first simple query above is more reliable for recent OOMKills. Use this detailed query for historical analysis when more data is available.

### Step 7: Creating and Observing a CrashLoop Scenario

Now let's simulate a real-world scenario: an application that crashes on startup.

```bash
# Update the crashloop-demo deployment to actually crash
cat <<EOF > crashloop-app-broken.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashloop-demo
  namespace: demo-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: crashloop-demo
  template:
    metadata:
      labels:
        app: crashloop-demo
    spec:
      containers:
      - name: alpine
        image: alpine:3.18
        # This command will fail, causing the container to crash
        command: ["sh", "-c", "echo 'Starting application...'; exit 1"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF

# Apply the broken deployment
kubectl apply -f crashloop-app-broken.yaml

# Watch the crashloop happen
kubectl get pods -n demo-apps -l app=crashloop-demo -w
```

You'll see the pods cycling through states:
- `Running` → `Error` → `CrashLoopBackOff` → `Running` → ...

#### Observe CrashLoop Signals:

**1. Using kubectl:**

```bash
# See the CrashLoopBackOff status
kubectl get pods -n demo-apps -l app=crashloop-demo

# Describe pod to see events
kubectl describe pod -n demo-apps -l app=crashloop-demo | grep -A 10 Events

# Check logs
kubectl logs -n demo-apps -l app=crashloop-demo --tail=20
```

**2. Using Container Insights:**

1. Go to Container Insights → **Containers** tab
2. Filter by `crashloop-demo`
3. Notice the high restart count
4. Click on the container to see:
   - Status: `CrashLoopBackOff`
   - Restart Count: Increasing rapidly
   - Status Reason: `Error`

**3. Query in Log Analytics:**

First, let's explore what data actually exists for crashlooping pods:

```kql
// Explore what data exists for crashloop-demo pods
KubePodInventory
| where TimeGenerated > ago(1h)
| where Name contains "crashloop-demo"
| distinct PodStatus, ContainerStatus, ContainerStatusReason, PodRestartCount
```

> **📚 Important**: Pods in CrashLoopBackOff typically have `PodStatus = "Running"` in KubePodInventory, even though they're constantly restarting. The key indicator is a high `PodRestartCount`, not the status field.

Find crashlooping pods by restart count:

```kql
// Find crashlooping pods by restart count
KubePodInventory
| where TimeGenerated > ago(1h)
| where PodRestartCount > 0  // Pods that have restarted
| summarize 
    LatestRestartCount = max(PodRestartCount),
    LatestStatus = arg_max(TimeGenerated, PodStatus, ContainerStatus)
    by Name, Namespace
| where LatestRestartCount >= 3  // Focus on pods with multiple restarts
| order by LatestRestartCount desc
```

More detailed query with events:

```kql
// Crashloop correlation with events
let CrashLoopPods = KubePodInventory
| where TimeGenerated > ago(1h)
| where PodRestartCount >= 3
| distinct Name, Namespace;
KubeEvents
| where TimeGenerated > ago(1h)
| join kind=inner CrashLoopPods on $left.Name == $right.Name
| where Reason in ("BackOff", "Failed", "Killing", "Started")
| project TimeGenerated, Namespace, Name, Reason, Message
| order by TimeGenerated desc
```

### Step 8: Querying Control Plane and Activity Logs

> **Note**: Diagnostic settings were enabled in Chapter 0, so control plane logs and Activity Logs should now be available in Log Analytics. If you just completed Chapter 0, data may still be arriving (5-10 minutes delay is normal).

#### Check Available Log Tables:

```kql
// List all AKS-related tables
search *
| where TimeGenerated > ago(1h)
| where $table startswith "AKS" or $table == "AzureActivity"
| distinct $table
```

You should see tables like `AKSControlPlane`, `AKSAudit`, and `AzureActivity`.

#### Query Kubernetes API Server Logs:

```kql
// Kubernetes API server logs (resource-specific table)
AKSControlPlane
| where Category == "kube-apiserver"
| where TimeGenerated > ago(1h)
| take 50
| order by TimeGenerated desc
```

#### Query Kubernetes Audit Logs:

```kql
// Kubernetes audit logs (resource-specific table)
AKSAudit
| where TimeGenerated > ago(1h)
| take 50
| order by TimeGenerated desc
```

#### Query Activity Logs:

Activity Logs show Azure Resource Manager operations on your AKS cluster.

```kql
// Recent AKS operations
AzureActivity
| where ResourceProvider == "Microsoft.ContainerService"
| where ResourceGroup == "<YOUR_RESOURCE_GROUP>"
| project TimeGenerated, OperationName, ActivityStatus, Caller, Properties
| order by TimeGenerated desc
| take 50
```

You should see operations like cluster creation, configuration changes, and scale operations.

> **⚠️ Troubleshooting**: If these tables are empty or don't exist, verify that diagnostic settings were properly enabled in Chapter 0. You can check in Azure Portal → AKS cluster → Diagnostic settings. Wait 5-10 minutes after enabling before expecting data.

### Step 9: Understanding Data Sources and Costs

Let's understand what data Container Insights collects and what costs you'll incur.

#### What Microsoft Azure Container Insights Collects — and What You Pay For

**Platform Metrics (Azure Monitor Metrics):**
- Basic CPU %, Memory %, Network counters
- **Stored separately from Log Analytics**
- **Do NOT count toward Log Analytics 5 GB free tier**
- Billed per time series (negligible cost for most scenarios)

**Log Analytics Data (5 GB/month free tier):**
- **5 GB/month per Log Analytics workspace** (all sources combined)
- This is a general Log Analytics allowance, **not specific to Container Insights**

**What Container Insights Collects by Default (counts toward 5 GB):**

**Always Collected (unless explicitly disabled):**
- Pod & container inventory (`KubeNodeInventory`, `KubePodInventory`)
- Kubernetes events (`KubeEvents`)
- Performance counters (`Perf` - detailed metrics in Log Analytics)
- Container logs (`ContainerLog`)

**Optional (requires diagnostic settings):**
- Control plane logs (`AzureDiagnostics`)

> **⚠️ Important**: Platform metrics visible in Azure Portal (CPU %, Memory %) are **NOT** stored in Log Analytics and don't count toward the 5 GB free tier. Only logs and detailed performance data sent to Log Analytics count toward the quota. The free tier applies to **all data** in the Log Analytics workspace, not just Container Insights data.

#### Query to Check Data Ingestion:

> **Note**: The `Usage` table may not be available immediately in a newly created Log Analytics workspace. Usage data is typically updated once per day. If you get an error about the table not being found, check back after 24 hours, or view usage in the Azure Portal instead.

**To view usage in Azure Portal:**
1. Navigate to your Log Analytics Workspace: `log-aksmon-<YOUR_INITIALS>`
2. Go to **Usage and estimated costs** in the left menu
3. View data ingestion by solution and table

**If the Usage table is available, try this query:**

```kql
// Data ingestion by table (last 7 days)
Usage
| where TimeGenerated > ago(7d)
| where IsBillable == true
| summarize TotalGB = sum(Quantity) / 1000 by DataType
| order by TotalGB desc
```

**Alternative query using a different approach:**

```kql
// Estimate data volume by counting records in key tables
union KubePodInventory, Perf, ContainerLog, KubeEvents
| where TimeGenerated > ago(1d)
| summarize RecordCount = count() by Type
| order by RecordCount desc
```

Estimate cost:
```kql
// Estimated cost (assuming $2.30/GB after free tier)
Usage
| where TimeGenerated > ago(7d)
| where IsBillable == true
| summarize TotalGB = sum(Quantity) / 1000
| extend EstimatedCost = iff(TotalGB > 5, (TotalGB - 5) * 2.30, 0.0)
| project TotalGB, EstimatedCost
```

## 🎓 Key Takeaways

By now you should understand:

1. **Where data comes from**: The `ama-logs` agent collects logs, events, and performance data from nodes and containers and sends them to Log Analytics
2. **What's free vs. paid**: First 5GB/month is free, then pay-per-GB ingestion
3. **Where to look when things go wrong**:
   - High CPU: Container Insights → Nodes tab
   - Pod restarts: Container Insights → Containers tab
   - OOMKills: Container Insights → Containers tab + Log Analytics queries
   - CrashLoops: kubectl describe + Container Insights + KubeEvents table
4. **How to query logs**: Use KQL in Log Analytics for detailed investigations
5. **Control plane visibility**: Query `AKSControlPlane`, `AKSAudit`, and `AzureActivity` tables for cluster operations

## ✅ Verification Checklist

Ensure you've completed:

- [ ] Verified Container Insights is enabled
- [ ] Navigated Container Insights dashboard (Cluster, Nodes, Controllers, Containers)
- [ ] Created CPU pressure and observed it in metrics
- [ ] Created OOMKill scenario and found it in logs
- [ ] Created CrashLoop scenario and analyzed the signals
- [ ] Queried control plane logs and Activity Logs
- [ ] Understood data sources and cost implications

## 🧹 Cleanup for This Chapter

```bash
# Remove the stress applications
kubectl delete deployment cpu-stress memory-oom -n demo-apps

# Fix the crashloop demo (or leave it for observation)
# kubectl delete deployment crashloop-demo -n demo-apps
```

## 🐛 Troubleshooting

### Issue: No data showing in Container Insights

**Solution**: 
- Wait 5-10 minutes after cluster creation
- Verify ama-logs pods are running: `kubectl get pods -n kube-system | grep ama-logs`
- Check agent logs: `kubectl logs -n kube-system <ama-logs-pod-name>`

### Issue: Cannot access Log Analytics Workspace

**Solution**: Ensure you have at least Reader permissions on the workspace:
```bash
az role assignment create \
  --assignee $(az account show --query user.name -o tsv) \
  --role "Log Analytics Reader" \
  --scope $LOG_ANALYTICS_WORKSPACE_ID
```

## 📚 Additional Resources

- [Container Insights Overview](https://docs.microsoft.com/azure/azure-monitor/containers/container-insights-overview)
- [KQL Quick Reference](https://docs.microsoft.com/azure/data-explorer/kql-quick-reference)
- [Azure Monitor Pricing](https://azure.microsoft.com/pricing/details/monitor/)

---

## ⏭️ Next Steps

Great work! You now know where to look when AKS behaves unexpectedly. In [Chapter 2: Azure Managed Prometheus + Grafana](../chapter-2-prometheus-grafana/README.md), you'll learn about the modern cloud-native monitoring stack with Prometheus and Grafana.

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
