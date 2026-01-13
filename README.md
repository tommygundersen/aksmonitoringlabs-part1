# AKS Monitoring & Observability - Hands-on Lab

Welcome to the Azure Kubernetes Service (AKS) Monitoring and Observability Lab! This comprehensive lab will guide you through essential monitoring techniques, metrics collection, alerting strategies, and cost optimization for AKS clusters using Azure-native and cloud-native tools.

## 🎯 Learning Objectives

By completing this lab, you will gain hands-on experience with:

- **Azure Monitor Integration**: Configure Container Insights and understand default observability in Azure
- **Managed Prometheus & Grafana**: Deploy and use Azure Managed Prometheus with Grafana dashboards
- **PromQL & Alerting**: Write meaningful PromQL queries and create actionable alerts
- **Metrics Strategy**: Understand the difference between platform metrics and Prometheus metrics
- **Cost Optimization**: Learn how to control and optimize observability costs

## 📚 Lab Structure

This lab is organized into the following chapters:

- **[Chapter 0: Setup and Prerequisites](chapter-0-setup/README.md)**
  - Setting your student initials
  - Creating resource group and AKS cluster
  - Installing required tools
  - Deploying sample applications

- **[Chapter 1: Azure Monitor + Container Insights](chapter-1-azure-monitor/README.md)**
  - Connecting AKS to Log Analytics Workspace
  - Enabling Container Insights
  - Understanding Activity Logs and Diagnostic Settings
  - Finding node CPU pressure, pod restarts, and OOMKills
  - Creating a crashloop scenario and observing signals

- **[Chapter 2: Azure Managed Prometheus + Grafana](chapter-2-prometheus-grafana/README.md)**
  - Enabling Azure Managed Prometheus
  - Connecting to Azure Managed Grafana
  - Importing pre-built dashboards
  - Understanding core Prometheus metrics
  - Exploring node_cpu_seconds_total and container_memory_working_set_bytes

- **[Chapter 3: PromQL Basics + Alerting](chapter-3-promql-alerting/README.md)**
  - Writing PromQL queries for CPU throttling and memory pressure
  - Querying pod availability metrics
  - Creating meaningful alerts (e.g., "Pod down for 5 minutes")
  - Triggering alerts intentionally
  - Understanding alerting best practices

- **[Chapter 4: Metrics Collection & Visualization](chapter-4-metrics-comparison/README.md)**
  - Understanding platform metrics vs. Prometheus metrics
  - Comparing Azure Monitor CPU % with Prometheus CPU metrics
  - Discussing sampling, granularity, and data freshness
  - Choosing the right metric for the right use case

- **[Chapter 5: Cost Optimization for Metrics Ingestion](chapter-5-cost-optimization/README.md)**
  - Identifying top 5 most expensive metrics
  - Adjusting scrape intervals and retention policies
  - Calculating cost before and after optimization
  - Implementing cost-effective observability strategies

## ⏱️ Estimated Time

- **Total Lab Duration**: 2-3 hours
- Chapter 0 (Setup): 15-20 minutes
- Chapter 1 (Azure Monitor): 25-30 minutes
- Chapter 2 (Prometheus & Grafana): 30-35 minutes
- Chapter 3 (PromQL & Alerting): 25-30 minutes
- Chapter 4 (Metrics Comparison): 15-20 minutes
- Chapter 5 (Cost Optimization): 15-20 minutes

## 🔧 Prerequisites

- Azure subscription with appropriate permissions
- Basic knowledge of Kubernetes concepts (pods, deployments, services)
- Familiarity with Azure CLI and kubectl
- Understanding of basic monitoring concepts
- Text editor (VS Code recommended)

## 🚀 Getting Started

1. Start with [Chapter 0: Setup and Prerequisites](chapter-0-setup/README.md)
2. Complete each chapter in sequence
3. Keep your AKS cluster running throughout the lab
4. Each chapter builds on previous concepts

## 💡 Tips for Success

- **Set your initials**: You'll use student initials for resource naming to avoid conflicts
- **One cluster for all**: Use the same AKS cluster throughout all chapters
- **Monitor as you go**: Keep Azure Portal and Grafana dashboards open to observe changes in real-time
- **Take your time with PromQL**: PromQL can be challenging - don't rush through Chapter 3
- **Save your queries**: Keep track of useful PromQL queries for future reference
- **Ask for help**: Don't hesitate to ask your instructor if you get stuck

## 📝 Notes

- All students in the same subscription should use unique initials for naming
- Some resources (especially Managed Prometheus and Grafana) may take several minutes to provision
- Metrics may take 1-2 minutes to appear after deployment
- Keep your Azure CLI session active throughout the lab
- Cost calculations in Chapter 5 are based on current Azure pricing (January 2026)

## 🧹 Cleanup

After completing all chapters, refer to the cleanup section in Chapter 0 to remove all resources and avoid unnecessary charges.

**Important**: Azure Monitor costs can accumulate quickly. Make sure to delete resources when you're done with the lab.

## 📖 Additional Resources

- [Azure Monitor Documentation](https://docs.microsoft.com/azure/azure-monitor/)
- [Container Insights Overview](https://docs.microsoft.com/azure/azure-monitor/containers/container-insights-overview)
- [Azure Managed Prometheus](https://docs.microsoft.com/azure/azure-monitor/essentials/prometheus-metrics-overview)
- [Azure Managed Grafana](https://docs.microsoft.com/azure/managed-grafana/overview)
- [PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)

---

Ready to begin? Head over to [Chapter 0: Setup and Prerequisites](chapter-0-setup/README.md) to get started!
