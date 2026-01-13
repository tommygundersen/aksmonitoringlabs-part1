# Chapter 3: PromQL Basics + Alerting

## 📋 Overview

In this chapter, you will learn to write PromQL queries and create actionable alerts. This is where junior developers become intermediate - understanding not just how to read dashboards, but how to write queries that answer specific questions and create alerts that signal real problems, not just noise.

## 🎯 Learning Objectives

After completing this chapter, you will:

- Write PromQL queries for CPU throttling and memory pressure
- Query pod availability metrics
- Understand PromQL operators and functions
- Create meaningful alerts (e.g., "Pod down for 5 minutes")
- Configure alert rules in Azure Monitor
- Trigger alerts intentionally to verify they work
- Understand alerting best practices (avoiding alert fatigue)

## ⏱️ Estimated Time

25-30 minutes

## 📝 Step-by-Step Instructions

### Step 1: PromQL Fundamentals

Before writing queries, let's understand PromQL basics.

#### PromQL Query Types:

1. **Instant Vector**: A set of time series at a single point in time
   ```promql
   node_cpu_seconds_total
   ```

2. **Range Vector**: A set of time series over a time range
   ```promql
   node_cpu_seconds_total[5m]
   ```

3. **Scalar**: A single numeric value
   ```promql
   100
   ```

#### Common Functions:

- `rate()`: Calculate per-second rate over a time range (for counters)
- `sum()`: Add values across dimensions
- `avg()`: Average values across dimensions
- `max()` / `min()`: Maximum / minimum values
- `increase()`: Total increase over a time range

#### Common Operators:

- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Comparison: `==`, `!=`, `>`, `<`, `>=`, `<=`
- Logical: `and`, `or`, `unless`

Let's put this into practice.

### Step 2: Writing Queries for CPU Throttling

CPU throttling happens when a container hits its CPU limit. This degrades performance.

#### First, Create a Throttling Scenario:

Before we can see throttling in action, we need to create a workload that will actually get throttled:

```bash
# Create a CPU-intensive deployment with low limits (will cause throttling)
cat <<EOF > cpu-throttling-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-throttled
  namespace: demo-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cpu-throttled
  template:
    metadata:
      labels:
        app: cpu-throttled
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--cpu", "4", "--timeout", "600s"]  # Request 4 CPUs
        resources:
          requests:
            cpu: "500m"
            memory: "128Mi"
          limits:
            cpu: "500m"  # But limit to 0.5 CPU - will throttle!
            memory: "256Mi"
EOF

kubectl apply -f cpu-throttling-demo.yaml

# Verify pods are running
kubectl get pods -n demo-apps -l app=cpu-throttled

# Wait 2-3 minutes for metrics to start flowing
echo "Waiting for throttling metrics to appear..."
```

> **💡 Why this causes throttling**: The stress container tries to use 4 CPU cores, but the CPU limit is only 500m (0.5 cores). Kubernetes will constantly throttle this container, which is perfect for our monitoring demo.

#### Now Query for CPU Throttling:

After waiting 2-3 minutes, open **Grafana → Explore** and try these queries:

> **⚠️ First, check if throttling metrics are available:**
> 
> Before running the throttling queries, let's verify the metrics exist. In Grafana Explore, try:
> ```promql
> # Check if throttling metrics exist
> container_cpu_cfs_throttled_seconds_total
> ```
> 
> **If this returns no data**, the throttling metrics might not be collected by Azure Managed Prometheus. This is a known limitation - not all cAdvisor metrics are exported by default.
>
> **Alternative approach**: Check CPU usage against limits instead:
> ```promql
> # CPU usage as percentage of limit
> sum by (namespace, pod, container) (
>   rate(container_cpu_usage_seconds_total{namespace="demo-apps", container!="", pod=~"cpu-throttled.*"}[5m])
> ) / sum by (namespace, pod, container) (
>   kube_pod_container_resource_limits{namespace="demo-apps", resource="cpu", pod=~"cpu-throttled.*"}
> ) * 100
> ```
> This shows CPU usage relative to limits. If it's consistently near 100%, the container is likely being throttled.

#### Query 1: Identify CPU Throttling (if metrics available)

```promql
# CPU throttling rate by container
sum by (namespace, pod, container) (
  rate(container_cpu_cfs_throttled_seconds_total[5m])
) > 0
```

**What this does:**
- `container_cpu_cfs_throttled_seconds_total`: Counter of throttled CPU time
- `rate(...[5m])`: Per-second rate over 5 minutes
- `sum by (...)`: Group by namespace/pod/container
- `> 0`: Only show containers that are being throttled

**Interpretation:**
- Value of 0.1 means throttled 10% of the time
- Value of 1.0 means throttled 100% of the time (very bad!)
- The cpu-throttled containers should show significant throttling if the stress test is working

> **⚠️ No data returned?** The `container_cpu_cfs_throttled_*` metrics may not be available in Azure Managed Prometheus. Skip to the alternative query below or move to Step 3.

#### Alternative Query: CPU Usage vs Limits

If throttling metrics aren't available, use this query to detect potential throttling:

```promql
# CPU usage as percentage of limit - high values indicate throttling
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{namespace="demo-apps", container!=""}[5m])
) / sum by (namespace, pod) (
  kube_pod_container_resource_limits{namespace="demo-apps", resource="cpu"}
) * 100
```

**Interpretation:**
- Values consistently at or near 100% indicate the container is hitting its CPU limit and likely being throttled
- The cpu-throttled pods should show ~100% (or higher if summed across containers)
- Normal pods (simple-web, memory-stress) should be well below their limits

> **Note**: Pods with multiple containers can show >100% when summed by pod (e.g., a 2-container pod with 500m limits each could show 200% if both hit their limits). For per-container accuracy, group by `container` instead of `pod`.

#### Query 2: Actual CPU Usage (Alternative)

```promql
# Compare actual CPU usage for throttled vs non-throttled pods
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="demo-apps", container!=""}[5m])
)
```

You should see:
- `cpu-throttled-*`: Around 0.5 cores (the limit)
- Other pods: Lower, variable usage

This confirms the cpu-throttled pods are hitting their limit.

### Step 3: Writing Queries for Memory Pressure

Memory pressure indicates a pod is close to its memory limit.

> **First, verify the metrics exist:**
> ```promql
> # Check if memory limit metrics are available
> kube_pod_container_resource_limits{resource="memory", namespace="demo-apps"}
> ```
> This should show memory limits for your containers.

#### Query 1: Memory Usage as Percentage of Limit

```promql
# Memory usage as % of limit
sum by (namespace, pod) (
  container_memory_working_set_bytes{namespace="demo-apps", container!=""}
) / sum by (namespace, pod) (
  kube_pod_container_resource_limits{namespace="demo-apps", resource="memory"}
) * 100
```

**What to look for:**
- > 80%: Pod is under memory pressure
- > 90%: High risk of OOMKill
- > 100%: Should already be OOMKilled (if limit is enforced)

> **Note**: We use `kube_pod_container_resource_limits` (from kube-state-metrics) instead of `container_spec_memory_limit_bytes` (cAdvisor metric) as it's more reliably available in Azure Managed Prometheus.

#### Query 2: Pods Using More Than 80% of Memory Limit

```promql
# Pods with high memory usage
(
  sum by (namespace, pod) (
    container_memory_working_set_bytes{namespace="demo-apps", container!=""}
  ) / sum by (namespace, pod) (
    kube_pod_container_resource_limits{namespace="demo-apps", resource="memory"}
  )
) > 0.8
```

This will only show pods that are using more than 80% of their memory limit.

#### Query 3: Actual Memory Usage (Simple View)

If the above queries don't work, use this simpler query to see memory usage:

```promql
# Memory usage by pod in MB
sum by (namespace, pod) (
  container_memory_working_set_bytes{namespace="demo-apps", container!=""}
) / 1024 / 1024
```

This shows actual memory usage without comparing to limits.

**Create memory pressure to test:**

```bash
# Create a memory-intensive workload
cat <<EOF > memory-pressure-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-pressure
  namespace: demo-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-pressure
  template:
    metadata:
      labels:
        app: memory-pressure
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "400M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"  # Using ~80% of limit
            cpu: "200m"
EOF

kubectl apply -f memory-pressure-demo.yaml

# Wait 2-3 minutes, then run the memory pressure queries
```

### Step 4: Querying Pod Availability

Pod availability is critical for SLAs and SLOs.

#### Query 1: Pods Not in Running State

```promql
# Count of pods not running by namespace
count by (namespace) (
  kube_pod_status_phase{phase!="Running"}
) 
  unless 
count by (namespace) (
  kube_pod_status_phase{phase="Running"}
)
```

> **Note**: This query avoids overcounting by excluding namespaces where Running pods exist. For a simpler view of non-Running pods:
> ```promql
> kube_pod_status_phase{phase=~"Pending|Failed|Unknown"}
> ```

#### Query 2: Pod Availability Percentage

```promql
# Percentage of pods in Running state
(
  count(kube_pod_status_phase{namespace="demo-apps", phase="Running"})
  /
  count(kube_pod_status_phase{namespace="demo-apps"})
) * 100
```

#### Query 3: Pods Restarted in Last Hour

```promql
# Pods that have restarted recently
increase(kube_pod_container_status_restarts_total{namespace="demo-apps"}[1h]) > 0
```

#### Query 4: Pod Uptime

```promql
# Pod uptime in hours
(time() - kube_pod_start_time{namespace="demo-apps"}) / 3600
```

### Step 5: Understanding Aggregation and Grouping

Aggregation is key to writing useful queries.

#### Example: Total CPU Usage Across All Pods

```promql
# Sum all pod CPU usage
sum(
  rate(container_cpu_usage_seconds_total{namespace="demo-apps", container!=""}[5m])
)
```

#### Example: Average Memory per Pod

```promql
# Average memory per pod
avg(
  container_memory_working_set_bytes{namespace="demo-apps", container!=""}
)
```

#### Example: Max CPU Usage per Node

```promql
# Max CPU usage by any pod on each node
max by (node) (
  rate(container_cpu_usage_seconds_total{namespace="demo-apps"}[5m])
)
```

### Step 6: Creating Your First Alert - Pod Down for 5 Minutes

Now let's create an alert that fires when a pod is down for more than 5 minutes.

#### Create Prometheus Alert Rule

In Azure, Prometheus alerts are managed through Azure Monitor Alert Rules.

```bash
# First, let's create an action group to receive notifications
export ACTION_GROUP_NAME="ag-aksmon-${STUDENT_INITIALS}"
export YOUR_EMAIL="your.email@example.com"  # Replace with your actual email

# Create action group (email notification)
az monitor action-group create \
  --resource-group $RESOURCE_GROUP \
  --name $ACTION_GROUP_NAME \
  --short-name "AKSAlerts" \
  --action email Email $YOUR_EMAIL

# Get action group ID
export ACTION_GROUP_ID=$(az monitor action-group show \
  --resource-group $RESOURCE_GROUP \
  --name $ACTION_GROUP_NAME \
  --query id \
  --output tsv)

echo "Action Group ID: $ACTION_GROUP_ID"
```

> **Important**: Replace `your.email@example.com` with your actual email address before running the command. The syntax is: `--action email <receiver-name> <email-address>`

You'll receive a confirmation email - be sure to click the activation link!

#### Understanding Prometheus Alerting Options

For Prometheus/Azure Managed Prometheus, there are two main alerting approaches:

1. **Grafana Managed Alerts** (Recommended for this lab) - Create alerts directly in Grafana UI
2. **Azure Monitor Prometheus Rule Groups** - Create alerts via Azure CLI/Portal (more complex)

We'll use Grafana since it's more visual and easier to learn.

### Step 7: Creating Alerts in Grafana

Let's create Prometheus-based alerts using Grafana's alerting feature.

#### Alert 1: Pod Not Running

1. In Grafana, go to **Alerting** (bell icon) → **Alert rules**
2. Click **+ New alert rule**
3. Configure the alert:

**Section 1: Set query and alert condition**
- Query: Select "Azure Managed Prometheus"
- Enter this PromQL query:
  ```promql
  count(kube_pod_status_phase{namespace="demo-apps", phase=~"Pending|Failed|Unknown"}) > 0
  ```
- Set threshold: **IS ABOVE** `0`
- This will trigger when any pod in demo-apps is in a non-Running, non-Succeeded state

**Section 2: Set evaluation behavior**
- Folder: Create new folder "AKS Monitoring" or use "General Alerting"
- Evaluation group: Create new "Demo Apps Alerts"
- Evaluation interval: `1m` (check every minute)
- Pending period: `5m` (wait 5 minutes before firing)

**Section 3: Add details**
- Alert name: `Pod Not Running in demo-apps`
- Description: `Alert when a pod in demo-apps namespace is not in Running state for 5 minutes`

**Section 4: Notifications**
- For now, skip this section (Grafana notifications require additional setup)
- We'll observe the alert firing in the Grafana UI

4. Click **Save rule and exit**

#### Test the Alert

Let's trigger this alert by scaling a deployment to zero:

```bash
# Scale simple-web to zero replicas
kubectl scale deployment simple-web -n demo-apps --replicas=0

# Wait about 1-2 minutes, then check Grafana
```

In Grafana:
1. Go to **Alerting** → **Alert rules**
2. Wait 1-2 minutes
3. The alert should show **Pending** status (waiting for 5 minutes)
4. After 5 minutes, it will change to **Firing**

**Restore the deployment:**
```bash
# Scale back to 3 replicas
kubectl scale deployment simple-web -n demo-apps --replicas=3

# Wait for alert to resolve (1-3 minutes)
# Check Alerting → Alert rules - status should change from Firing → Normal
```

The alert resolution process:
1. Pods start running (~30 seconds)
2. Prometheus scrapes updated pod status (~30-60 seconds)
3. Grafana re-evaluates alert query (~1 minute)
4. Alert resolves to Normal state

Total time: **2-3 minutes** from scaling up to alert resolution.

#### Alert 2: High Memory Usage

Let's create another alert for high memory usage:

1. In Grafana → **Alerting** → **Alert rules** → **+ New alert rule**
2. Configure:

**Query:**
```promql
(
  sum by (namespace, pod) (
    container_memory_working_set_bytes{namespace="demo-apps"}
  ) 
  / 
  sum by (namespace, pod) (
    kube_pod_container_resource_limits{namespace="demo-apps", resource="memory"}
  )
) > 0.85
```

**Alert name:** `High Memory Usage in demo-apps`

**Description:** `Alert when pod memory usage exceeds 85% of limits`

**Evaluation:**
- Interval: `1m`
- Pending: `3m`

3. **Save rule and exit**

#### Alert 3: Frequent Pod Restarts

Let's create one more alert for detecting crashlooping pods:

1. **Alerting** → **Alert rules** → **+ New alert rule**

**Query:**
```promql
increase(kube_pod_container_status_restarts_total{namespace="demo-apps"}[15m]) > 3
```

**Alert name:** `Frequent Pod Restarts in demo-apps`

**Description:** `Alert when a pod restarts more than 3 times in 15 minutes`

**Evaluation:**
- Interval: `5m`
- Pending: `0m` (fire immediately)

3. **Save rule and exit**

### Step 8: Observing Alerts

Check your alerts in Grafana:

1. Go to **Alerting** → **Alert rules**
2. You should see all 3 alerts in **Normal** state (green)

Alert states:
- **Normal** (green): Everything OK
- **Pending** (yellow): Condition met but waiting for pending period
- **Firing** (red): Alert is actively firing

### Step 9: Testing Memory Alert

Let's trigger the high memory alert:

```bash
# Deploy a pod that uses ~88% of its memory limit
cat <<EOF > high-memory-trigger.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-memory-trigger
  namespace: demo-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: high-memory-trigger
  template:
    metadata:
      labels:
        app: high-memory-trigger
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "450M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"  # Will use ~88% of limit (450/512)
            cpu: "200m"
EOF

kubectl apply -f high-memory-trigger.yaml

# Wait 3-4 minutes for metrics to appear and alert to fire
```

Check in Grafana:
1. **Alerting** → **Alert rules**
2. After ~1-2 minutes: "High Memory Usage" should show **Pending**
3. After 3 minutes total: It should change to **Firing** (red)

**Cleanup:**
```bash
kubectl delete deployment high-memory-trigger -n demo-apps

# Wait for alert to resolve
# This takes 1-3 minutes:
# 1. Prometheus scrapes updated metrics (~30-60 sec)
# 2. Grafana re-evaluates alert (~1 min based on evaluation interval)
# 3. Alert transitions from Firing → Normal
```

Check alert status:
1. Go to **Alerting** → **Alert rules** in Grafana
2. Wait 1-3 minutes
3. "High Memory Usage" should change from **Firing** (red) → **Normal** (green)

> **Note**: Grafana-managed alerts (created in Grafana UI) only appear in Grafana, not in Azure Portal. To see alerts in Azure Monitor → Alerts, you must create them using Azure Monitor alert rules (via CLI or Portal). Both approaches work with Prometheus data, but the management interface differs.

### Step 10: Alert Best Practices - Avoiding Alert Fatigue

Creating alerts that matter is a skill. Here are key principles:

#### ✅ DO:

1. **Alert on symptoms, not causes**
   - ✅ Good: "API response time > 2s for 5 minutes"
   - ❌ Bad: "CPU usage > 80%"

2. **Use appropriate time windows (pending periods)**
   - Don't alert on transient spikes
   - Use 3-5 minute pending periods to avoid noise

3. **Set realistic thresholds**
   - Base on actual baselines, not arbitrary numbers
   - 80% memory might be fine for some workloads

4. **Make alerts actionable**
   - Include runbook links in descriptions
   - Clear title that explains the problem

5. **Use severity correctly**
   - **Critical**: Service is down, customer impact
   - **Warning**: Potential issue, no customer impact yet
   - **Info**: Informational, FYI only

#### ❌ DON'T:

1. **Don't alert on everything**
   - Focus on what impacts users or SLOs
   
2. **Don't set thresholds too low**
   - This creates alert fatigue
   
3. **Don't create alerts without testing**
   - Always verify alerts can fire and resolve

4. **Don't ignore alerts**
   - If you're ignoring it, delete it

#### Example: Good vs Bad Alerts

**Bad Alert:**
```promql
# Alert on any pod restart
increase(kube_pod_container_status_restarts_total[5m]) > 0
```
- Too noisy (single restart might be expected)
- No context on impact

**Good Alert:**
```promql
# Alert on frequent restarts indicating a problem
increase(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 3
```
- Threshold accounts for normal restarts
- Focused on production namespace
- 15-minute window filters transient issues

### Advanced: Email Notifications (Optional)

To receive email notifications from Grafana alerts, you need to configure a contact point:

1. In Grafana → **Alerting** → **Contact points**
2. **+ Add contact point**
3. Name: `Email Notifications`
4. Integration: **Email**
5. Addresses: Enter your email
6. **Note**: This requires Grafana to have SMTP configured

For Azure Managed Grafana, email notifications require:
- Azure Monitor action groups integration (complex setup)
- OR custom SMTP configuration
- OR webhook integrations (Teams, Slack, etc.)

This is outside the scope of this introductory lab, but in production you would configure this.

## 🎓 Key Takeaways
## 🎓 Key Takeaways

By now you should understand:

1. **PromQL Basics**: How to write queries with `rate()`, `sum()`, aggregations
2. **CPU Usage Queries**: Monitor CPU usage vs limits to detect throttling
3. **Memory Pressure Queries**: Identify pods at risk of OOMKill
4. **Pod Availability Queries**: Monitor running state and restarts
5. **Grafana Alerting**: Create visual alert rules with PromQL queries
6. **Alert Testing**: Trigger alerts intentionally to verify functionality
7. **Best Practices**: Create actionable alerts that reduce noise

**The skill you've developed**: You can now create alerts that signal real problems, not just noise. This is what separates junior from intermediate engineers.

## ✅ Verification Checklist

Ensure you've completed:

- [ ] Wrote queries for CPU usage vs limits
- [ ] Wrote queries for memory pressure using working_set_bytes
- [ ] Wrote queries for pod availability and restarts
- [ ] Created "Pod Not Running" alert in Grafana
- [ ] Created "High Memory Usage" alert
- [ ] Created "Frequent Pod Restarts" alert
- [ ] Triggered the high memory alert with stress deployment
- [ ] Verified alert states (Normal → Pending → Firing → Normal)
- [ ] Understood alert best practices

## 🧹 Cleanup for This Chapter

```bash
# Remove test deployments if any are still running
kubectl delete deployment high-memory-trigger -n demo-apps --ignore-not-found

# Keep alerts in Grafana - we'll use them in later chapters
```

## 🐛 Troubleshooting

### Issue: Alert doesn't fire

**Solution**:
- Verify query returns data in Grafana Explore first
- Check evaluation interval and pending period settings
- Ensure metrics are actually being collected (check in Explore with same query)

### Issue: Query returns "no data"

**Solution**:
- Verify namespace and label selectors are correct
- Check time range in Grafana (last 15 minutes)
- Ensure Prometheus is scraping: `kubectl get pods -n kube-system | grep ama-metrics`
- For memory queries: Confirm resource limits are set on deployments

### Issue: Alert stuck in "Pending"

**Solution**:
- This is normal - pending period must elapse before firing
- If stuck too long, verify the condition is still true in Explore

## 📚 Additional Resources

- [PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Alerting Guide](https://grafana.com/docs/grafana/latest/alerting/)
- [Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)

---

## ⏭️ Next Steps

Excellent progress! You can now write meaningful queries and create actionable alerts. In [Chapter 4: Metrics Collection & Visualization](../chapter-4-metrics-comparison/README.md), you'll learn the important conceptual differences between platform metrics and Prometheus metrics.

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
