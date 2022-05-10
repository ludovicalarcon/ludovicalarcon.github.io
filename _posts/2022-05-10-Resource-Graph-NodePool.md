---
layout: post
title: Get node pool information on all AKS clusters with Azure Resource Graph
img: ResourceGraph.png
tags: [Azure, AKS, Resource Graph]
---

Like me you could have the need to know sku, node count and powerstate of the node pools of all your AKS clusters.

`Azure Resource Graph` looks the right tools to do so.
According to official [documentation](https://docs.microsoft.com/en-us/azure/governance/resource-graph/overview)
> Azure Resource Graph is an Azure service designed to extend Azure Resource Management by providing efficient and performant resource exploration with the ability to query at scale across a given set of subscriptions so that you can effectively govern your environment.

 Azure Resource Graph's query language is based on `KQL` (Kusto Query Language).  
 For more details, I recommend you to go through the [documentation](https://docs.microsoft.com/en-us/azure/governance/resource-graph/concepts/query-language)

 It took me several attempts and documentation reading before I could get what I wanted.
 ```
 Resources
 | where type == "microsoft.contaunerservice/managedclusters"
 | extend properties.agentPoolProfiles
 | project subscriptionId, name, nodePool = properties.agentPoolProfiles
 | mv-expand nodePool
 | project subscriptionId, name, sku = nodePool.vmSize, count = nodePool.['count'], powerState = nodePool.powerState.code 
 ```

You can now have for AKS cluster you have:
 - how many nodes your node pool has
 - what's the sku of underlying VMs
 - the powerstate of the node pool
 
With only one query!

`Azure Resource Graph` is very powerful