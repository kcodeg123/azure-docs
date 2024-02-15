---
title: Set up Private Link for a Chaos Studio agent-based experiment [Preview]
description: Understand the steps to set up a chaos experiment using private link for agent-based experiments
services: chaos-studio
author: nikhilkaul-msft
ms.topic: how-to
ms.date: 12/04/2023
ms.author: abbyweisberg
ms.reviewer: nikhilkaul
ms.service: chaos-studio
ms.custom: ignite-fall-2023
---

# How-to: Configure Private Link for Agent-Based experiments [Preview]

This guide explains the steps needed to configure Private Link for a Chaos Studio **Agent-based** Experiment [Preview]. The current user experience is based on the private endpoints support enabled as part of public preview of the private endpoints feature. Expect this experience to evolve with time as the feature is enhanced to GA quality, as it is currently in **preview**. 

---
## Prerequisites

1. An Azure account with an active subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
2. First define your agent-based experiment by following the steps found [here](chaos-studio-tutorial-agent-based-portal.md).

> [!NOTE]
> If the target resource was created using the portal, then the chaos agent VM extension will be austomatically installed on the host VM. If the target is enabled using the CLI, then follow the Chaos Studio documentation to install the VM extension first on the virtual machine. Until you complete the private endpoint setup, the VM extension will be reporting an unhealthy state. This is expected.

<br/>

## Limitations

- You'll need to use our **2023-10-27-preview REST API** to create and use private link for agent-based experiments ONLY. There's **no** support for private link for agent-based experiments in our GA-stable REST API until H1 2024. 

- The entire end-to-end for this flow requires some use of the CLI. The current end-to-end experience cannot be done from the Azure portal currently. 

- The **Chaos Studio Private Accesses (CSPA)** resource type has a **strict 1:1 mapping of Chaos Target:CSPA Resource (abstraction for private endpoint).**.** We only allow **5 CSPA resources to be created per Subscription** to maintain the expected experience for all of our customers.  
 
## Step 1: Create a Chaos Studio Private Access (CSPA) resource

To use Private endpoints for agent-based chaos experiments, you need to create a new resource type called **Chaos Studio Private Accesses**. CSPA is the resource against which the private endpoints are created.

> [!NOTE]
> Currently this resource can **only be created from the CLI**. See the example code for how to do this:

 ```AzCLI
az rest --verbose --skip-authorization-header --header "Authorization=Bearer $accessToken" --uri-parameters api-version=2023-10-27-preview --method PUT --uri "https://centraluseuap.management.azure.com/subscriptions/<subscriptionID>/resourceGroups/<resourceGroupName>/providers/Microsoft.Chaos/privateAccesses/<CSPAResourceName>?api-version=2023-10-27-preview" --body ' 

{ 

    "location": "<resourceLocation>", 

    "properties": { 

        "id": "<CSPAResourceName>", 

        "name": "<CSPAResourceName>", 

        "location": "<resourceLocation>", 

        "type": "Microsoft.Chaos/privateAccesses", 

        "resourceId": "subscriptions/<subscriptionID>/resourceGroups/<resourceGroupName>/providers/Microsoft.Chaos/privateAccesses/<CSPAResourceName>" 

    } 

}'
 ```

| Name |Required | Type | Description |
|-|-|-|-|
|subscriptionID|True|String|GUID that represents an Azure subscription ID|
|resourceGroupName|True|String|String that represents an Azure resource group|
|CSPAResourceName|True|String|String that represents the name you want to give your Chaos Studio Private Access Resource|
|resourceLocation|True|String|Location you want the resource to be hosted (must be a support region by Chaos Studio)|


## Step 2: Create your Virtual Network, Subnet, and Private Endpoint

[Set up your desired Virtual Network, Subnet, and Endpoint](../private-link/create-private-endpoint-portal.md) for the experiment if you haven't already.

Make sure you attach it to the same VM's VNET. Screenshots provide examples of creating the VNET, Subnet and Private Endpoint. It's important to note that you need to set the "Resource Type" to "Microsoft.Chaos/privateAccesses" as seen in the screenshot. 

[![Screenshot of resource tab of private endpoint creation.](images/resource-private-endpoint.png)](images/resource-private-endpoint.png#lightbox)

[![Screenshot of VNET tab of private endpoint creation.](images/resource-vnet-cspa.png)](images/resource-vnet-cspa.png#lightbox)


## Step 3: Map the agent host VM to the CSPA resource

Find the Target "Resource ID" by making a GetTarget call:

```AzCLI
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{parentProviderNamespace}/{parentResourceType}/{parentResourceName}/providers/Microsoft.Chaos/targets/{targetName}?api-version=2023-10-27-preview
```

<br/>

The GET command returns a large response. Note this response. We use this response and modify it before running a "PUT Target" command to map the two resources. 

<br/>

Invoke a "PUT Target" command using this response. You need to append **TWO ADDITIONAL FIELDS** to the body of the PUT command before running it.

These extra fields are shown below:

```
"privateAccessId": "subscriptions/<subID>/...
"allowPublicAccess": false

},
```

Here's an example block for what the "PUT Target" command should look like and the fields that you would need to fill out:

> [!NOTE]
> The body should be copied from the previous GET command. You'll need to manually append the "privateAccessID" and "allowPublicAccess" fields. 

```AzCLI

az rest --verbose --skip-authorization-header --header "Authorization=Bearer $accessToken" --uri-parameters api-version=2023-10-27-preview --method PUT --uri "https://management.azure.com/subscriptions/<subscriptionID>/resourceGroups/<resourceGroup>/providers/Microsoft.Compute/virtualMachines/<VMSSname>/providers/Microsoft.Chaos/targets/Microsoft-Agent?api-version=2023-10-27-preview " --body ' {
    "id": "/subscriptions/<subscriptionID>/resourceGroups/<resourceGroupName>/providers/microsoft.compute/virtualmachines/<VMSSName>/providers/Microsoft.Chaos/targets/Microsoft-Agent",
    "type": "Microsoft.Chaos/targets",
    "name": "Microsoft-Agent",
    "location": "<resourceLocation>",
    "properties": {
        "agentProfileId": "<from target resource>",
        "identities": [
            {
                "type": "AzureManagedIdentity",
                "clientId": "<clientID>",
                "tenantId": "<tenantID>"
            }
        ],
        "agentTenantId": "CHAOSSTUDIO",
        "privateAccessId": "subscriptions/<subscriptionID>/resourceGroups/<resourceGroupName>/providers/Microsoft.Chaos/privateAccesses/<CSPAresourceName>",
        "allowPublicAccess": false
    }} '

```

> [!NOTE]
> The PrivateAccessID should exactly match the "resourceID" used to create the CSPA resource in Step 1.

## Step 4: Restart the Azure Chaos Agent service in the VM

After making all the required changes to the host, restart the Azure Chaos Agent Service in the VM

### Windows

[![Screenshot of restarting Windows VM.](images/restart-windows-vm.png)](images/restart-windows-vm.png#lightbox)

### Linux

For Linux, run the following command from the CLI:

```
Systemctl restart azure-chaos-agent
```

[![Screenshot of restarting Linux VM.](images/restart-linux-vm.png)](images/restart-linux-vm.png#lightbox)

## Step 5: Run your Agent-based experiment using private endpoints

After the restart, the Chaos agent should be able to communicate with the Agent Communication data plane service and the agent registration to the data plane should be successful. After successful registration, the agent will be able to heartbeat its status and you can go ahead and run the chaos agent-based experiments using private endpoints!

