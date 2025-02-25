---

title: Enable Container insights | Microsoft Docs
description: This article describes how to enable and configure Container insights so that you can understand how your container is performing and what performance-related issues have been identified. 
ms.topic: conceptual
ms.date: 06/30/2020

---

# Enable Container insights

This article provides an overview of the options that are available for setting up Container insights to monitor the performance of workloads that are deployed to Kubernetes environments and hosted on:

- [Azure Kubernetes Service (AKS)](../../aks/index.yml)  
- [Azure Arc-enabled Kubernetes cluster](../../azure-arc/kubernetes/overview.md)
   - [Azure Stack](/azure-stack/user/azure-stack-kubernetes-aks-engine-overview) or on-premises
   - [AKS engine](https://github.com/Azure/aks-engine)
   - [Azure Red Hat OpenShift](../../openshift/intro-openshift.md) version 4.x  
   - [Red Hat OpenShift](https://docs.openshift.com/container-platform/4.3/welcome/index.html) version 4.x  


You can enable Container insights for a new deployment or for one or more existing deployments of Kubernetes by using any of the following supported methods:

- The Azure portal
- Azure PowerShell
- The Azure CLI
- [Terraform and AKS](/azure/developer/terraform/create-k8s-cluster-with-tf-and-aks)

For any non-AKS kubernetes cluster, you will need to first connect your cluster to [Azure Arc](../../azure-arc/kubernetes/overview.md) before enabling monitoring.

[!INCLUDE [updated-for-az](../../../includes/updated-for-az.md)]

## Prerequisites

Before you start, make sure that you've met the following requirements:

> [!IMPORTANT]
> Log Analytics Containerized Linux Agent (replicaset pod) makes API calls to all the Windows nodes on Kubelet Secure Port (10250) within the cluster to collect Node and Container Performance related Metrics. 
Kubelet secure port (:10250) should be opened in the cluster's virtual network for both inbound and outbound for Windows Node and container performance related metrics collection to work.
>
> If you have a Kubernetes cluster with Windows nodes, then please review and configure the Network Security Group and Network Policies to make sure the Kubelet secure port (:10250) is opened for both inbound and outbound in cluster's virtual network.


- You have a Log Analytics workspace.

   Container insights supports a Log Analytics workspace in the regions that are listed in [Products available by region](https://azure.microsoft.com/global-infrastructure/services/?regions=all&products=monitor).

   You can create a workspace when you enable monitoring for your new AKS cluster, or you can let the onboarding experience create a default workspace in the default resource group of the AKS cluster subscription. 
   
   If you choose to create the workspace yourself, you can create it through: 
   - [Azure Resource Manager](../logs/resource-manager-workspace.md)
   - [PowerShell](../logs/powershell-workspace-configuration.md?toc=%2fpowershell%2fmodule%2ftoc.json)
   - [The Azure portal](../logs/quick-create-workspace.md) 
   
   For a list of the supported mapping pairs to use for the default workspace, see [Region mapping for Container insights](container-insights-region-mapping.md).

- You are a member of the *Log Analytics contributor* group for enabling container monitoring. For more information about how to control access to a Log Analytics workspace, see [Manage workspaces](../logs/manage-access.md).

- You are a member of the [*Owner* group](../../role-based-access-control/built-in-roles.md#owner) on the AKS cluster resource.

   [!INCLUDE [log-analytics-agent-note](../../../includes/log-analytics-agent-note.md)]

- To view the monitoring data, you need to have [*Log Analytics reader*](../logs/manage-access.md#manage-access-using-azure-permissions) role in the Log Analytics workspace, configured with Container insights.

- Prometheus metrics aren't collected by default. Before you [configure the agent](container-insights-prometheus-integration.md) to collect the metrics, it's important to review the [Prometheus documentation](https://prometheus.io/) to understand what data can be scraped and what methods are supported.
- An AKS cluster can be attached to a Log Analytics workspace in a different Azure subscription in the same Azure AD Tenant. This cannot currently be done with the Azure Portal, but can be done with Azure CLI or Resource Manager template.

## Supported configurations

Container insights officially supports the following configurations:

- Environments: Azure Red Hat OpenShift, Kubernetes on-premises, and the AKS engine on Azure and Azure Stack. For more information, see [the AKS engine on Azure Stack](/azure-stack/user/azure-stack-kubernetes-aks-engine-overview).
- The versions of Kubernetes and support policy are the same as those [supported in Azure Kubernetes Service (AKS)](../../aks/supported-kubernetes-versions.md).
- We recommend connecting your cluster to [Azure Arc](../../azure-arc/kubernetes/overview.md) and enabling monitoring through Container Insights via Azure Arc.

## Network firewall requirements

The following table lists the proxy and firewall configuration information that's required for the containerized agent to communicate with Container insights. All network traffic from the agent is outbound to Azure Monitor.

|Agent resource|Port |
|--------------|------|
| `*.ods.opinsights.azure.com` | 443 |
| `*.oms.opinsights.azure.com` | 443 |
| `dc.services.visualstudio.com` | 443 |
| `*.monitoring.azure.com` | 443 |
| `login.microsoftonline.com` | 443 |

The following table lists the proxy and firewall configuration information for Azure China 21Vianet:

|Agent resource|Port |Description | 
|--------------|------|-------------|
| `*.ods.opinsights.azure.cn` | 443 | Data ingestion |
| `*.oms.opinsights.azure.cn` | 443 | OMS onboarding |
| `dc.services.visualstudio.com` | 443 | For agent telemetry that uses Azure Public Cloud Application Insights |

The following table lists the proxy and firewall configuration information for Azure US Government:

|Agent resource|Port |Description | 
|--------------|------|-------------|
| `*.ods.opinsights.azure.us` | 443 | Data ingestion |
| `*.oms.opinsights.azure.us` | 443 | OMS onboarding |
| `dc.services.visualstudio.com` | 443 | For agent telemetry that uses Azure Public Cloud Application Insights |

## Components

Your ability to monitor performance relies on a containerized Log Analytics agent for Linux that's specifically developed for Container insights. This specialized agent collects performance and event data from all nodes in the cluster, and the agent is automatically deployed and registered with the specified Log Analytics workspace during deployment. 

The  agent version is microsoft/oms:ciprod04202018 or later, and it's represented by a date in the following format: *mmddyyyy*.

>[!NOTE]
>With the general availability of Windows Server support for AKS, an AKS cluster with Windows Server nodes has a preview agent installed as a daemonset pod on each individual Windows server node to collect logs and forward it to Log Analytics. For performance metrics, a Linux node that's automatically deployed in the cluster as part of the standard deployment collects and forwards the data to Azure Monitor on behalf all Windows nodes in the cluster.

When a new version of the agent is released, it's automatically upgraded on your managed Kubernetes clusters that are hosted on Azure Kubernetes Service (AKS). To track which versions are released, see [agent release announcements](https://github.com/microsoft/docker-provider/tree/ci_feature_prod).

> [!NOTE]
> If you've already deployed an AKS cluster, you've enabled monitoring by using either the Azure CLI or a provided Azure Resource Manager template, as demonstrated later in this article. You can't use `kubectl` to upgrade, delete, redeploy, or deploy the agent.
>
> The template needs to be deployed in the same resource group as the cluster.

To enable Container insights, use one of the methods that's described in the following table:

| Deployment state | Method | Description |
|------------------|--------|-------------|
| New Kubernetes cluster | [Create an AKS cluster by using the Azure CLI](../../aks/kubernetes-walkthrough.md#create-aks-cluster)| You can enable monitoring for a new AKS cluster that you create by using the Azure CLI. |
| | [Create an AKS cluster by using Terraform](container-insights-enable-new-cluster.md#enable-using-terraform)| You can enable monitoring for a new AKS cluster that you create by using the open-source tool Terraform. |
| | [Create an OpenShift cluster by using an Azure Resource Manager template](container-insights-azure-redhat-setup.md#enable-for-a-new-cluster-using-an-azure-resource-manager-template) | You can enable monitoring for a new OpenShift cluster that you create by using a preconfigured Azure Resource Manager template. |
| | [Create an OpenShift cluster by using the Azure CLI](/cli/azure/openshift#az_openshift_create) | You can enable monitoring when you deploy a new OpenShift cluster by using the Azure CLI. |
| Existing AKS cluster | [Enable monitoring of an AKS cluster by using the Azure CLI](container-insights-enable-existing-clusters.md#enable-using-azure-cli) | You can enable monitoring for an AKS cluster that's already deployed by using the Azure CLI. |
| |[Enable for AKS cluster using Terraform](container-insights-enable-existing-clusters.md#enable-using-terraform) | You can enable monitoring for an AKS cluster that's already deployed by using the open-source tool Terraform. |
| | [Enable for AKS cluster from Azure Monitor](container-insights-enable-existing-clusters.md#enable-from-azure-monitor-in-the-portal)| You can enable monitoring for one or more AKS clusters that are already deployed from the multi-cluster page in Azure Monitor. |
| | [Enable from AKS cluster](container-insights-enable-existing-clusters.md#enable-directly-from-aks-cluster-in-the-portal)| You can enable monitoring directly from an AKS cluster in the Azure portal. |
| | [Enable for AKS cluster using an Azure Resource Manager template](container-insights-enable-existing-clusters.md#enable-using-an-azure-resource-manager-template)| You can enable monitoring for an AKS cluster by using a preconfigured Azure Resource Manager template. |
| Existing non-AKS Kubernetes cluster | [Enable for non-AKS Kubernetes cluster by using the Azure CLI](container-insights-enable-arc-enabled-clusters.md#create-extension-instance-using-azure-cli). | You can enable monitoring for your Kubernetes clusters that are hosted outside of Azure and enabled with Azure Arc, this includes hybrid, OpenShift, and multi-cloud using Azure CLI. |
| | [Enable for non-AKS Kubernetes cluster using an Azure Resource Manager template](container-insights-enable-arc-enabled-clusters.md#create-extension-instance-using-azure-resource-manager) | You can enable monitoring for your clusters enabled with Arc by using a preconfigured Azure Resource Manager template. |
| | [Enable for non-AKS Kubernetes cluster from Azure Monitor](container-insights-enable-arc-enabled-clusters.md#create-extension-instance-using-azure-portal) | You can enable monitoring for one or more clusters enabled with Arc that are already deployed from the multicluster page in Azure Monitor. |

## Next steps

Now that you've enabled monitoring, you can begin analyzing the performance of your Kubernetes clusters that are hosted on Azure Kubernetes Service (AKS), Azure Stack, or another environment. To learn how to use Container insights, see [View Kubernetes cluster performance](container-insights-analyze.md).