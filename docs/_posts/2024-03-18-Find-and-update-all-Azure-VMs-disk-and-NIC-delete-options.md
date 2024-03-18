---
layout: post
title:  "Find and update all Azure VMs' disk and NIC delete options"
date:   2024-03-18 00:00:00 +0000
categories: Azure
---

In Azure you can set a default action when deleting a virtual machine. This could be to delete the disks and NICs along with the VM, or to keep them.

Today we will look into how to find the current setting for all machines and update it as needed.

## Find the setting from the portal
For any VM, you can find this setting by heading to the JSON View and look for the `deleteOptions` property under `storageProfile` > `osDisk` or `dataDisks` and under `networkProfile` > `networkInterfaces` > `properties` > `deleteOptions`.

![alt text](/assets/2024-03-18-Find-and-update-all-Azure-VMs-disk-and-NIC-delete-options/VM-JSON-View.png)

## Find the setting using PowerShell

Let's find the current setting for all VMs in a subscription. We will use PowerShell for this.

{% highlight PowerShell %}
Get-AzVM | Select-Object Name, @{label='OSDiskDeleteOption';expression={$_.StorageProfile.OsDisk.DeleteOption}}, @{label='NICDeleteOption';expression={$_.NetworkProfile.NetworkInterfaces[0].DeleteOption}} | Format-Table
{% endhighlight %}

This is great, but limited to one subscription. We can leverage Azure Resource Graph to query this across all subscriptions.

## Find the setting using Azure Resource Graph Explorer
We can use the below KQL query,

{% highlight KQL %}
resources
| where ['type'] == 'microsoft.compute/virtualmachines'
| extend DiskDeleteOption = properties.storageProfile.osDisk.deleteOption
| extend NICDeleteOption = properties.networkProfile.networkInterfaces[0].properties.deleteOption
| project ['id'], name, resourceGroup, subscriptionId, DiskDeleteOption, NICDeleteOption
{% endhighlight %}

## Update the setting using PowerShell
Now how do we use the above query to update all VMs?

If we are targeting a single VM, we can use the below PowerShell command,

{% highlight PowerShell %}
$vm = Get-AzVM -ResourceGroupName "MyResourceGroup" -Name "MyVM"
$vm.StorageProfile.OsDisk.DeleteOption = "Delete"
$vm.NetworkProfile.NetworkInterfaces[0].DeleteOption = "Delete"
$vm | Update-AzVM
{% endhighlight %}

We can expand on the above command to update all VMs using the below PowerShell script, let us say we want to switch from `Delete` to `Detach`,
{% highlight PowerShell %}
$kqlQeury = @"
resources
| where ['type'] == 'microsoft.compute/virtualmachines'
| extend DiskDeleteOption = properties.storageProfile.osDisk.deleteOption
| extend NICDeleteOption = properties.networkProfile.networkInterfaces[0].properties.deleteOption
| where DiskDeleteOption == 'Delete' or NICDeleteOption  == 'Delete'
| project ['id'], name, resourceGroup, subscriptionId, DiskDeleteOption, NICDeleteOption
"@
$vmsWithDeleteOption = Search-AzGraph -Query $kqlQeury
$vmsWithDeleteOption | ForEach-Object {$_.name;$null = Set-AzContext -SubscriptionId $_.subscriptionId;$vm = Get-azvm -ResourceId $_.id; $vm.StorageProfile.OsDisk.DeleteOption = 'Detach'; $vm | Update-AzVM -NoWait}
{% endhighlight %}

This script will go through all VMs and update the delete options as needed. Bear in mind that this is just an example and could be optimized in different ways, such as checking if we are already in the right context, sorting by subscription,
or even switching to REST API so we can use parallel processing without the need to switch contexts. If your organization has a policy to keep or delete disks and NICs, you can enforce this using an Azure Policy with a remediation task.

In conclusion, we are able to query and update the delete options to quickly change the behavior. We have different tools for the job so select the proper one according to how many VMs and how complex the environment is.
