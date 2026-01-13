# Chapter 2: Azure Managed Prometheus + Grafana

## 📋 Overview

In this chapter, you will set up Azure Managed Prometheus and Azure Managed Grafana - the modern, cloud-native monitoring stack for AKS. This is becoming the new standard for Kubernetes observability in Azure and is a highly sought-after skill in the market.

## 🎯 Learning Objectives

After completing this chapter, you will:

- Enable Azure Managed Prometheus for your AKS cluster
- Connect Azure Managed Grafana to your Prometheus workspace
- Import and explore pre-built Kubernetes dashboards
- Understand core Prometheus metrics like `node_cpu_seconds_total` and `container_memory_working_set_bytes`
- Differentiate between Azure-native metrics and cloud-native Prometheus metrics
- Navigate Grafana dashboards confidently

## ⏱️ Estimated Time

30-35 minutes

## 📝 Step-by-Step Instructions

### Step 1: Understanding Azure Managed Prometheus vs Container Insights

Before we begin, let's understand the difference:

| Feature        | Container Insights         | Azure Managed Prometheus        |
| -------------- | -------------------------- | ------------------------------- |
| Data Store     | Log Analytics (Logs)       | Prometheus (Time Series)        |
| Query Language | KQL (Kusto)                | PromQL                          |
| Visualization  | Azure Portal               | Grafana                         |
| Use Case       | Azure-native monitoring    | Cloud-native, portable          |
| Cardinality    | Lower                      | Higher (more labels)            |
| Retention      | 30-730 days (configurable) | 18 months (fixed per workspace) |
| Cost Model     | Per GB ingested            | Per million samples             |

> **Both are valuable**: Container Insights for Azure-native tooling, Prometheus for cloud-native workflows.

### Step 2: Create Azure Monitor Workspace (for Prometheus)

Azure Managed Prometheus requires an Azure Monitor Workspace - this is different from Log Analytics.

```bash
# Define Azure Monitor Workspace name
export AZURE_MONITOR_WORKSPACE="amw-${STUDENT_INITIALS}-prometheus"

# Create Azure Monitor Workspace
az resource create \
  --resource-group $RESOURCE_GROUP \
  --namespace microsoft.monitor \
  --resource-type accounts \
  --name $AZURE_MONITOR_WORKSPACE \
  --location $LOCATION \
  --properties '{}'

# Get the workspace resource ID
export AZURE_MONITOR_WORKSPACE_ID=$(az resource show \
  --resource-group $RESOURCE_GROUP \
  --namespace microsoft.monitor \
  --resource-type accounts \
  --name $AZURE_MONITOR_WORKSPACE \
  --query id \
  --output tsv)

echo "Azure Monitor Workspace ID: $AZURE_MONITOR_WORKSPACE_ID"
```

> **Note**: The workspace is created instantly, but it will take 5-10 minutes after enabling Prometheus in Step 3 before metrics start appearing in Grafana.

### Step 3: Enable Managed Prometheus on AKS

Now let's configure AKS to send metrics to the Azure Monitor Workspace.

```bash
# Enable Prometheus metrics collection on AKS
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id $AZURE_MONITOR_WORKSPACE_ID

# This will take 3-5 minutes
```

#### Verify Prometheus is collecting metrics:

```bash
# Check that ama-metrics pods are running
kubectl get pods -n kube-system | grep ama-metrics

# You should see:
# - ama-metrics-* (DaemonSet on each node)
# - ama-metrics-ksm-* (kube-state-metrics)
```

The `ama-metrics` DaemonSet scrapes Prometheus metrics from default targets:
- **Node exporter**: Node-level metrics (CPU, memory, disk, network)
- **cAdvisor**: Container-level metrics (resource usage per container)
- **kube-state-metrics**: Kubernetes object state (pods, deployments, services)

> **Note**: These default targets use Azure-managed scrape configurations (30-second intervals by default). You can customize scrape intervals and add custom targets in Chapter 5.

### Step 4: Create Azure Managed Grafana Instance

```bash
# Define Grafana instance name
export GRAFANA_NAME="grafana-${STUDENT_INITIALS}"

# Create Azure Managed Grafana
az grafana create \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# This will take 5-7 minutes
echo "Grafana instance creation started..."
```

While Grafana is being created, let's understand what you're getting:

**Azure Managed Grafana provides:**
- Fully managed Grafana service
- No infrastructure to maintain
- Built-in Azure AD authentication
- High availability
- Automatic updates
- Integration with Azure Monitor Workspace

### Step 5: Configure Permissions and Link Prometheus to Grafana

```bash
# Get your user Object ID
export USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Assign Grafana Admin role to yourself
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Grafana Admin" \
  --scope $(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)

# Get the Prometheus query endpoint from Azure Monitor Workspace
export PROMETHEUS_QUERY_ENDPOINT=$(az monitor account show \
  --name $AZURE_MONITOR_WORKSPACE \
  --resource-group $RESOURCE_GROUP \
  --query metrics.prometheusQueryEndpoint \
  --output tsv)

echo "Prometheus Query Endpoint: $PROMETHEUS_QUERY_ENDPOINT"

# Delete existing data source if it exists (to avoid conflicts)
az grafana data-source delete \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --data-source "Azure Managed Prometheus" 2>/dev/null || true

# Link the Azure Monitor Workspace as a data source in Grafana
az grafana data-source create \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --definition '{
    "name": "Azure Managed Prometheus",
    "type": "prometheus",
    "access": "proxy",
    "url": "'"$PROMETHEUS_QUERY_ENDPOINT"'",
    "jsonData": {
      "httpMethod": "POST",
      "azureCredentials": {
        "authType": "msi"
      }
    },
    "isDefault": true
  }'

echo "Data source created successfully!"
```

> **Note**: The `prometheusQueryEndpoint` includes a unique identifier in the hostname (e.g., `-d3a9agh0euhnbkcx`). This is the correct format. If the data source already exists, we delete it first to ensure it's recreated with the correct URL.

### Step 6: Access Grafana Dashboard

```bash
# Get Grafana endpoint URL
export GRAFANA_ENDPOINT=$(az grafana show \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --query properties.endpoint \
  --output tsv)

echo "Grafana URL: $GRAFANA_ENDPOINT"
echo ""
echo "Copy and paste the URL above into your browser"
```

> **Example output**: `https://grafana-togunder4-a5gkeed7fzeaaffs.cse.grafana.azure.com`

**To access Grafana:**

1. Copy the Grafana endpoint URL from the output above
2. Open it in your browser
3. Sign in with your Azure AD credentials
4. You should see the Grafana home page

### Step 6.5: Verify Prometheus Data Source Connection

Before importing dashboards, let's verify that the Prometheus data source is working correctly.

**In Grafana UI:**

1. Click **Connections** (plug icon) in the left sidebar
2. Click **Data sources**
3. You should see "Azure Managed Prometheus" listed
4. Click on it
5. Scroll to the bottom and click **Save & test**
6. You should see a green checkmark: "Data source is working"

> **⚠️ If you get an error**: The data source might not be configured correctly. Try these steps:
> 
> **Option 1: Delete and recreate the data source via CLI**
> ```bash
> # Delete the existing data source
> az grafana data-source delete \
>   --name $GRAFANA_NAME \
>   --resource-group $RESOURCE_GROUP \
>   --data-source "Azure Managed Prometheus"
> 
> # Get the correct Prometheus endpoint
> export PROMETHEUS_QUERY_ENDPOINT=$(az monitor account show \
>   --name $AZURE_MONITOR_WORKSPACE \
>   --resource-group $RESOURCE_GROUP \
>   --query metrics.prometheusQueryEndpoint \
>   --output tsv)
> 
> echo "Using endpoint: $PROMETHEUS_QUERY_ENDPOINT"
> 
> # Recreate the data source with correct URL
> az grafana data-source create \
>   --name $GRAFANA_NAME \
>   --resource-group $RESOURCE_GROUP \
>   --definition '{
>     "name": "Azure Managed Prometheus",
>     "type": "prometheus",
>     "access": "proxy",
>     "url": "'"$PROMETHEUS_QUERY_ENDPOINT"'",
>     "jsonData": {
>       "httpMethod": "POST",
>       "azureCredentials": {
>         "authType": "msi"
>       }
>     },
>     "isDefault": true
>   }'
> ```
> Then refresh the Grafana page and test the data source again.
>
> **Option 2: Fix via Grafana UI**
> 1. Get the correct endpoint URL:
>    ```bash
>    az monitor account show \
>      --name $AZURE_MONITOR_WORKSPACE \
>      --resource-group $RESOURCE_GROUP \
>      --query metrics.prometheusQueryEndpoint \
>      --output tsv
>    ```
> 2. Copy the full URL (it should include the hostname with a unique identifier)
> 3. In Grafana data source settings, update the URL field with the correct endpoint
> 4. Ensure "Managed Identity" authentication is selected
> 5. Click **Save & test**

**Verify metrics are flowing:**

After the data source shows "working", let's verify metrics are available:

1. In Grafana, click **Explore** (compass icon) in the left sidebar
2. Ensure "Azure Managed Prometheus" is selected as the data source (top dropdown)
3. In the query field, enter: `up`
4. Click **Run query** (or press Shift+Enter)
5. You should see a table/graph with results showing scraped targets

> **If you see "No data"**: Metrics may still be populating. Remember from Chapter 0 that it can take 5-10 minutes for Prometheus metrics to start flowing. Wait a few minutes and try again.

### Step 7: Import Pre-Built Kubernetes Dashboard

> **⚠️ Important**: Before proceeding, ensure you completed Step 6.5 and verified that:
> - The data source shows "Data source is working" (green checkmark)
> - The `up` query in Explore returns data
> 
> If not, the dashboard import will fail with "An error occurred within the plugin".

Azure provides pre-built dashboards for AKS. Let's import the main Kubernetes cluster monitoring dashboard.

> **💡 Recommendation**: Use Option A (Grafana UI) as it's more reliable. Option B (CLI) can sometimes fail with generic errors.

> **⚠️ Dashboard ID Note**: Dashboard IDs from Grafana.com (like 15759) can change or become unavailable over time. If `15759` doesn't work, search "Kubernetes" on grafana.com/dashboards and look for recent, highly-rated Kubernetes monitoring dashboards. The import process is the same regardless of ID.

#### Option A: Import from Grafana UI (Recommended)

1. In Grafana, click on **Dashboards** (left sidebar, four-squares icon)
2. Click **New** → **Import**
3. In the "Import via grafana.com" field, enter dashboard ID: `15759`
4. Click **Load**
5. In the "Prometheus" dropdown, select your data source: **"Azure Managed Prometheus"**
6. Click **Import**

You should now see the "Kubernetes / Views / Global" dashboard with live data!

#### Option B: Import via Azure CLI (Alternative)

```bash
# Import pre-configured Kubernetes dashboard
az grafana dashboard import \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --definition 15759
```

> **⚠️ Note**: If the CLI command fails with "An error occurred", use Option A (UI import) instead. The CLI import can be unreliable for Grafana.com dashboards.

#### Import Additional Useful Dashboards:

Once you have the first dashboard working, import these additional dashboards using the same UI process:

- **15760**: Kubernetes / Views / Namespaces
- **15761**: Kubernetes / Views / Pods  
- **15757**: Kubernetes / System / API Server
- **15758**: Kubernetes / System / Kubelet

> **Tip**: For each dashboard, use Dashboards → New → Import, enter the ID, load it, select "Azure Managed Prometheus" as the data source, and import.

### Step 8: Explore the Dashboard

Now that you have a dashboard, let's explore what you're looking at.

#### Navigate to the Dashboard:

1. Click **Dashboards** → **Browse**
2. Open "Kubernetes / Views / Global"
3. Set the time range to "Last 1 hour" (top right)

#### What You're Seeing:

**Cluster Overview Panel:**
- **Cluster CPU Usage**: Total CPU used across all nodes
- **Cluster Memory Usage**: Total memory used
- **Pod Count**: Number of running pods
- **Container Count**: Total containers

**Node Metrics:**
- CPU usage per node
- Memory usage per node
- Disk I/O per node
- Network traffic per node

**Pod Metrics:**
- CPU usage per pod
- Memory usage per pod
- Network traffic per pod
- Restart counts

### Step 9: Understanding Core Prometheus Metrics

Let's dive deep into two fundamental metrics.

#### Metric 1: `node_cpu_seconds_total`

This is a **counter** metric that tracks total CPU time in seconds.

**In Grafana:**
1. Click on any CPU chart
2. Click **Edit** (pencil icon)
3. Look at the query - you'll see something like:

```promql
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```

**What this means:**
- `node_cpu_seconds_total`: Counter of CPU seconds
- `{mode!="idle"}`: Exclude idle time (only active CPU)
- `rate(...[5m])`: Calculate rate over 5-minute window
- Result: CPU cores actively used per second

**Why it matters:**
- This is THE metric for CPU monitoring
- Shows actual CPU consumption, not just requests/limits
- Used for capacity planning and autoscaling

**Try this query yourself:**
1. Go to **Explore** (compass icon in left sidebar)
2. Select "Azure Managed Prometheus" data source
3. Enter query:

```promql
# Total CPU usage by node
sum by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))
```

#### Metric 2: `container_memory_working_set_bytes`

This is a **gauge** metric that shows current memory usage.

**Query to try:**

```promql
# Memory usage by container
sum by (namespace, pod, container) (
  container_memory_working_set_bytes{container!=""}
)
```

**What this means:**
- `container_memory_working_set_bytes`: Current memory in use
- `{container!=""}`: Exclude pause containers
- `sum by (namespace, pod, container)`: Group by namespace/pod/container
- Result: Memory in bytes per container

**Why it matters:**
- This is what Kubernetes uses for OOMKill decisions
- This is the **active memory** that cannot be easily reclaimed (excludes page cache)
- Working set = active memory + kernel memory

**Try querying memory for your demo apps:**

```promql
# Memory working set for simple-web pods
container_memory_working_set_bytes{namespace="demo-apps", pod=~"simple-web.*"}
```

> **📚 Note**: You may see references to `container_memory_usage_bytes` in older documentation, but this metric is **not available** in Azure Managed Prometheus. Always use `container_memory_working_set_bytes` for memory monitoring in AKS, as it represents the memory that Kubernetes tracks for OOMKill decisions.

### Step 10: Create Your Own Dashboard Panel

Let's create a custom dashboard to monitor our demo applications.

#### Create a new dashboard:

1. Click **Dashboards** → **New Dashboard**
2. Click **Add visualization**
3. Select "Azure Managed Prometheus" data source

#### Panel 1: Demo Apps CPU Usage

In the query editor, enter:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="demo-apps", container!=""}[5m])
)
```

Configure the panel:
- Set visualization type: **Time series** (default)
- Panel title: "Demo Apps CPU Usage" (in the right sidebar under "Panel options")
- Unit: In right sidebar → **Standard options** → **Unit** → select "cores"
- Click **Save** (top right corner) or **Back to dashboard** to return to the dashboard

> **Note**: In newer Grafana versions, you may not see an "Apply" button. Instead, click **Save** in the top right to save the dashboard, or use the **Back to dashboard** arrow to return without saving yet.

#### Panel 2: Demo Apps Memory Usage

From the dashboard, click **Add** → **Visualization**:

In the query editor:

```promql
sum by (pod) (
  container_memory_working_set_bytes{namespace="demo-apps", container!=""}
) / 1024 / 1024
```

Configure the panel:
- Visualization: **Time series**
- Title: "Demo Apps Memory Usage"
- Unit: **Data** → **mebibytes (MiB)**
- Click **Save** or **Back to dashboard**

#### Panel 3: Pod Restart Count

From the dashboard, click **Add** → **Visualization**:

In the query editor:

```promql
sum by (pod) (
  kube_pod_container_status_restarts_total{namespace="demo-apps"}
)
```

Configure the panel:
- Visualization: **Stat** (change from Time series)
- Title: "Pod Restarts"
- Click **Save** or **Back to dashboard**

#### Save Your Dashboard:

1. Click **Save dashboard** (floppy disk icon, top right)
2. Dashboard name: "Demo Apps Monitoring"
3. Click **Save**

You now have a custom dashboard monitoring your demo applications!

### Step 11: Explore Additional Metrics

Try these queries in the **Explore** view to understand more metrics:

#### Network Traffic:

```promql
# Network receive bytes per pod
sum by (pod) (
  rate(container_network_receive_bytes_total{namespace="demo-apps"}[5m])
)
```

#### Disk I/O:

```promql
# Disk read bytes per node
sum by (instance) (
  rate(node_disk_read_bytes_total[5m])
)
```

#### Available Memory:

```promql
# Available memory per node (in GB)
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

#### Pod Count by Namespace:

```promql
# Count of running pods per namespace
count by (namespace) (
  kube_pod_status_phase{phase="Running"}
)
```

### Step 12: Understanding Metrics Cardinality

One key advantage of Prometheus is high cardinality - many labels for fine-grained filtering.

**Example: CPU metric with all labels:**

```promql
node_cpu_seconds_total
```

Click on a result to see all labels:
- `instance`: Node name
- `job`: Scrape job
- `mode`: CPU mode (idle, user, system, etc.)
- `cluster`: Cluster name
- `region`: Azure region

**You can filter by any combination:**

```promql
# CPU for specific node in user mode
node_cpu_seconds_total{instance="aks-nodepool1-12345678-vmss000000", mode="user"}
```

This flexibility is powerful but requires understanding PromQL.

## 🎓 Key Takeaways

By now you should understand:

1. **Azure Managed Prometheus** is the modern standard for AKS monitoring
2. **Grafana dashboards** provide powerful, customizable visualization
3. **Core metrics**:
   - `node_cpu_seconds_total`: CPU consumption (use with `rate()`)
   - `container_memory_working_set_bytes`: Memory that matters for OOMKills
4. **Prometheus vs Container Insights**: Different tools for different needs
5. **High cardinality**: Prometheus allows fine-grained filtering with labels

## ✅ Verification Checklist

Ensure you've completed:

- [ ] Created Azure Monitor Workspace for Prometheus
- [ ] Enabled Managed Prometheus on AKS cluster
- [ ] Created Azure Managed Grafana instance
- [ ] Linked Prometheus data source to Grafana
- [ ] Imported at least one pre-built Kubernetes dashboard
- [ ] Explored node and pod metrics in dashboards
- [ ] Understood `node_cpu_seconds_total` metric
- [ ] Understood `container_memory_working_set_bytes` metric
- [ ] Created a custom dashboard with 3 panels
- [ ] Experimented with PromQL queries in Explore view

## 🐛 Troubleshooting

### Issue: No metrics showing in Grafana

**Solution**:
1. Wait 5-10 minutes after enabling Prometheus
2. Verify ama-metrics pods are running:
   ```bash
   kubectl get pods -n kube-system | grep ama-metrics
   ```
3. Check data source configuration in Grafana (Settings → Data Sources)

### Issue: Cannot access Grafana URL

**Solution**: Ensure you have Grafana Admin role:
```bash
az role assignment create \
  --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Grafana Admin" \
  --scope $(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)
```

### Issue: Dashboard import fails

**Solution**: Import manually:
1. Go to https://grafana.com/grafana/dashboards/15759
2. Download JSON
3. In Grafana: Dashboards → Import → Upload JSON file

### Issue: Queries return no data

**Solution**: 
- Check time range (top right in Grafana)
- Verify namespace/pod names match your deployment
- Use Metrics Explorer to see available metrics

## 📚 Additional Resources

- [Azure Managed Prometheus Documentation](https://docs.microsoft.com/azure/azure-monitor/essentials/prometheus-metrics-overview)
- [Azure Managed Grafana Documentation](https://docs.microsoft.com/azure/managed-grafana/overview)
- [Grafana Dashboard Gallery](https://grafana.com/grafana/dashboards/)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)

---

## ⏭️ Next Steps

Excellent work! You can now read and interpret Prometheus dashboards. In [Chapter 3: PromQL Basics + Alerting](../chapter-3-promql-alerting/README.md), you'll learn to write your own PromQL queries and create meaningful alerts.

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
