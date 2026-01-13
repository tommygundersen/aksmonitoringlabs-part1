# Chapter 0: Setup and Prerequisites

## 📋 Overview

In this chapter, you will set up the necessary infrastructure and tools for the AKS Monitoring lab. This includes creating an AKS cluster, installing required tools, and deploying sample applications that we'll monitor throughout the lab.

## 🎯 Learning Objectives

- Set up your Azure environment with proper naming conventions
- Create an AKS cluster with monitoring capabilities
- Deploy sample applications for monitoring scenarios
- Verify all prerequisites are in place

## ⏱️ Estimated Time

15-20 minutes

## 🔧 Prerequisites

Before you begin, ensure you have:

- Azure subscription with Contributor or Owner access
- Azure CLI installed (version 2.50.0 or later)
- kubectl installed
- A terminal/command prompt
- Text editor (VS Code recommended)

## 📝 Step-by-Step Instructions

### Step 1: Set Your Student Initials

To avoid naming conflicts, each student should set their unique initials. Replace `xyz` with your own initials (2-3 characters).

```bash
# Set your initials (use lowercase, 2-3 characters)
export STUDENT_INITIALS="xyz"

# Verify
echo "Your initials: $STUDENT_INITIALS"
```

> **Note**: If you're using PowerShell on Windows, use:
> ```powershell
> $env:STUDENT_INITIALS="xyz"
> echo "Your initials: $env:STUDENT_INITIALS"
> ```

### Step 2: Set Azure Subscription

```bash
# Login to Azure
az login

# List your subscriptions
az account list --output table

# Set your subscription (replace with your subscription ID)
az account set --subscription "YOUR_SUBSCRIPTION_ID"

# Verify
az account show --output table
```

### Step 3: Define Resource Names

```bash
# Define common variables
export RESOURCE_GROUP="rg-aksmon-${STUDENT_INITIALS}"
export LOCATION="swedencentral"
export AKS_CLUSTER_NAME="aks-mon-${STUDENT_INITIALS}"
export LOG_ANALYTICS_WORKSPACE="log-aksmon-${STUDENT_INITIALS}"

# Verify all variables
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "AKS Cluster: $AKS_CLUSTER_NAME"
echo "Log Analytics Workspace: $LOG_ANALYTICS_WORKSPACE"
```

> **PowerShell users**:
> ```powershell
> $env:RESOURCE_GROUP="rg-aksmon-$env:STUDENT_INITIALS"
> $env:LOCATION="swedencentral"
> $env:AKS_CLUSTER_NAME="aks-mon-$env:STUDENT_INITIALS"
> $env:LOG_ANALYTICS_WORKSPACE="log-aksmon-$env:STUDENT_INITIALS"
> ```

### Step 4: Create Resource Group

```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Verify
az group show --name $RESOURCE_GROUP --output table
```

### Step 5: Create Log Analytics Workspace

We'll create the Log Analytics Workspace upfront so we can enable Container Insights during AKS cluster creation.

```bash
# Create Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE \
  --location $LOCATION

# Get the workspace ID for later use
export LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE \
  --query id \
  --output tsv)

echo "Log Analytics Workspace ID: $LOG_ANALYTICS_WORKSPACE_ID"
```

### Step 6: Create AKS Cluster

Create an AKS cluster with monitoring enabled from the start:

```bash
# Create AKS cluster with Container Insights enabled
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_ID \
  --generate-ssh-keys \
  --network-plugin azure \
  --network-policy azure

# This will take 5-10 minutes
```

> **Why these settings?**
> - `--node-count 3`: Three nodes to demonstrate load distribution
> - `--node-vm-size Standard_D4s_v3`: Sufficient resources for monitoring workloads
> - `--enable-addons monitoring`: Enables Container Insights from the start
> - `--network-plugin azure`: Required for some advanced monitoring scenarios

### Step 7: Connect to AKS Cluster

```bash
# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Verify connection
kubectl get nodes

# You should see 3 nodes in Ready state
```

### Step 8: Install kubectl plugins (optional but recommended)

```bash
# Install krew (kubectl plugin manager) if not already installed
# For Linux/macOS:
# (
#   set -x; cd "$(mktemp -d)" &&
#   OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
#   ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
#   KREW="krew-${OS}_${ARCH}" &&
#   curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
#   tar zxvf "${KREW}.tar.gz" &&
#   ./"${KREW}" install krew
# )

# Install useful plugins
# kubectl krew install ctx ns top
```

### Step 9: Create Sample Namespace

```bash
# Create namespace for demo applications
kubectl create namespace demo-apps

# Set as default namespace (optional)
kubectl config set-context --current --namespace=demo-apps

# Verify
kubectl get namespace demo-apps
```

### Step 10: Deploy Sample Applications

We'll deploy a few different application types to monitor:

#### Application 1: Simple Web Application

```bash
# Create deployment manifest
cat <<EOF > simple-web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-web
  namespace: demo-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: simple-web
  template:
    metadata:
      labels:
        app: simple-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: simple-web-svc
  namespace: demo-apps
spec:
  type: LoadBalancer
  selector:
    app: simple-web
  ports:
  - port: 80
    targetPort: 80
EOF

# Apply
kubectl apply -f simple-web-app.yaml
```

#### Application 2: Memory-Intensive Application (for OOM testing)

```bash
# Create memory stress application
cat <<EOF > memory-stress-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-stress
  namespace: demo-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-stress
  template:
    metadata:
      labels:
        app: memory-stress
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"  # This will cause OOM if we increase stress
            cpu: "200m"
EOF

# Apply
kubectl apply -f memory-stress-app.yaml
```

#### Application 3: CrashLoop Application (for Chapter 1)

This will be used later to demonstrate crashloop detection.

```bash
# Create crashloop application (initially working)
cat <<EOF > crashloop-app.yaml
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
        command: ["sh", "-c", "echo 'Application started successfully'; sleep 3600"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF

# Apply
kubectl apply -f crashloop-app.yaml
```

### Step 11: Verify Deployments

```bash
# Check all pods are running
kubectl get pods -n demo-apps

# Check services
kubectl get svc -n demo-apps

# Wait for all pods to be Ready
kubectl wait --for=condition=ready pod -l app=simple-web -n demo-apps --timeout=120s
kubectl wait --for=condition=ready pod -l app=memory-stress -n demo-apps --timeout=120s
kubectl wait --for=condition=ready pod -l app=crashloop-demo -n demo-apps --timeout=120s
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
crashloop-demo-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
crashloop-demo-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
memory-stress-xxxxxxxxxx-xxxxx    1/1     Running   0          1m
simple-web-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
simple-web-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
simple-web-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
```

### Step 12: Enable Diagnostic Settings for Monitoring

To ensure data is available when you start Chapter 1, let's enable diagnostic settings now. This will give logs time to populate while you verify the setup.

#### Part 1: Enable AKS Control Plane Logs

```bash
# Create diagnostic settings for AKS with resource-specific tables
az monitor diagnostic-settings create \
  --resource $(az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query id -o tsv) \
  --name "aks-diagnostics" \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID \
  --export-to-resource-specific true \
  --logs '[
    {"category": "kube-apiserver", "enabled": true},
    {"category": "kube-controller-manager", "enabled": true},
    {"category": "kube-scheduler", "enabled": true},
    {"category": "kube-audit", "enabled": true},
    {"category": "cluster-autoscaler", "enabled": true}
  ]' \
  --metrics '[
    {"category": "AllMetrics", "enabled": true}
  ]'
```

#### Part 2: Enable Activity Logs Export

```bash
# Get subscription ID (if not already set)
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Create diagnostic settings for Activity Logs (subscription-level)
az monitor diagnostic-settings subscription create \
  --subscription $SUBSCRIPTION_ID \
  --name "activity-log-to-workspace" \
  --location $LOCATION \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID \
  --logs '[
    {"category": "Administrative", "enabled": true},
    {"category": "Security", "enabled": true},
    {"category": "ServiceHealth", "enabled": true},
    {"category": "Alert", "enabled": true},
    {"category": "Policy", "enabled": true},
    {"category": "Autoscale", "enabled": true},
    {"category": "ResourceHealth", "enabled": true}
  ]'
```

> **💡 Why enable this now?**
> - Diagnostic logs take 5-10 minutes to start flowing to Log Analytics
> - Activity logs can take even longer to appear
> - By enabling these settings now, data will be available when you need it in Chapter 1
> - Resource-specific tables (`AKSControlPlane`, `AKSAudit`, `AzureActivity`) provide better schemas than legacy `AzureDiagnostics` table

> **Note**: These logs do incur costs. For this lab, costs should be minimal, but for production environments, be selective about which log categories you enable.

### Step 13: Verify Container Insights is Collecting Data

```bash
# Check that the monitoring agent (ama-logs) is running
kubectl get pods -n kube-system | grep ama-logs

# You should see ama-logs-* pods running on each node
```

## ✅ Verification Checklist

Before proceeding to Chapter 1, verify:

- [ ] AKS cluster is created and running
- [ ] Log Analytics Workspace is created
- [ ] kubectl can connect to the cluster
- [ ] All 3 nodes are in Ready state
- [ ] Namespace `demo-apps` exists
- [ ] All 6 demo application pods are running
- [ ] ama-logs pods are running in kube-system namespace
- [ ] Diagnostic settings are enabled for AKS control plane logs
- [ ] Activity log export is configured

> **⏰ Important**: After completing setup, wait **5-10 minutes** before starting Chapter 1. This allows monitoring data to start flowing to Log Analytics. You can use this time for a break or to review the [Azure Monitor documentation](https://docs.microsoft.com/azure/azure-monitor/).

## 🧹 Cleanup (DO NOT RUN YET - only after completing all chapters)

When you've completed all chapters and want to clean up:

```bash
# Delete the entire resource group
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

# Verify deletion (optional - will show error when fully deleted)
az group show --name $RESOURCE_GROUP
```

## 🐛 Troubleshooting

### Issue: AKS cluster creation fails

**Solution**: Check subscription quotas and ensure you have enough capacity:
```bash
az vm list-usage --location $LOCATION --output table
```

### Issue: kubectl cannot connect

**Solution**: Ensure credentials are properly configured:
```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --overwrite-existing
kubectl config current-context
```

### Issue: Pods are not starting

**Solution**: Check pod events and logs:
```bash
kubectl describe pod <pod-name> -n demo-apps
kubectl logs <pod-name> -n demo-apps
```

## 📚 Additional Resources

- [AKS Documentation](https://docs.microsoft.com/azure/aks/)
- [Azure CLI Reference](https://docs.microsoft.com/cli/azure/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## ⏭️ Next Steps

Congratulations! Your environment is now set up. Proceed to [Chapter 1: Azure Monitor + Container Insights](../chapter-1-azure-monitor/README.md) to start monitoring your AKS cluster.

---

## ⚠️ Disclaimer

**Educational/lab purposes only.** Calculations and/or statements may contain errors.
