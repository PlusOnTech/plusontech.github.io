---
layout: post
title:  "Fix Inconsistent Tag Names In Azure"
date:   2024-03-13 00:00:00 +0000
categories: Azure,Tags,PowerShell
---
Today a customer approached with a very interesting request, they had been enforcing the use of tags in their Azure environment, but they found that the tag names were not consistent.

For example, they wanted to use `ApplicationName` as the tag name, but they found tags such as `App`, `Application Name`, and other variations.

To avoid this in the future, we are planning to use [Azure Policies](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies) to enforce the use of specific tags and values.

But for existing resources, we wanted to find the different variations used and replace that with the approved tag, `ApplicationName`. We came up with the following PowerShell script to do this, it also generates a report of the changes made.

## Script Explanation
Required Modules: az.ResourceGraph, PSFramework. The PSFramework is used for converting from PSCustomObject to Hashtable and for logging.
PSResource is a module that can be used to install other modules from the PowerShell Gallery, its available on latest versions of PowerShell 7. If you are using an older or Windows PowerShell, use Install-module

The script uses Resource Graph to find all resource, across all subscriptions regardless of context. I find this to be faster when working with larger environments.

We start by defining what is the proper name for the tag, and a list of different variations that we want to fix. We then build a query to find all resources with any of the bad tags and no good tag.
```PowerShell
$GoodTagName = "ApplicationName" # This is the proper Tag Name that you want to use for all resources

$BadTagNames = @( # This is a list of the tag names that you want to fix.
    "AplicationName"
    "Apllication"
    "App"
    "Appication Name"
    "ApplcationName"
    "ApplicatinName"
    "Application Name "
    "Application Name"
    "Application"
    "ApplicationN"
    "ApplicationNam"
    "ApplicationName "
)

$BadTagNamesQuery = ($BadTagNames | ForEach-Object { "isnotnull(tags['$_'])" }) -join " or "

# Build a query to find all resources with any of the bad tags and no good tag
$query = @"
resources
| where ($BadTagNamesQuery) and isnull(tags['$GoodTagName'])
"@
$result = Search-AzGraph -Query $query
```

We then loop through the results, starting with a report entry as an ordered hashtable.
```
$reportEntry = [ordered]@{
    Name           = $resource.name
    ResourceGroup  = $resource.resourceGroup
    SubscriptionId = $resource.subscriptionId
    ResourceType   = $resource.type
    ResourceId     = $resource.id
}
```
For each resource we check if it has multiple bad tags. If it does, we skip it. This is because if there is more than one bad variation we don't know which value to use. This would require some manual intervention later.

```PowerShell
$badTag = ($resource.tags | Get-Member -MemberType NoteProperty | Where-Object { $_.Name -in $BadTagNames }).Name
if ($badTag.count -ne 1) {
    # Skip resources with multiple bad tags
    Write-PSFMessage -Level Warning -Message "Resource $($resource.id) has $tagCount bad tags"
    $reportEntry['Action'] = "Error"
    $reportEntry['ErrorMessage'] = "Resource has multiple bad tags: {0}" -f ($badTag -join ", ")
}
```
If the resource only has one bad tag, we use its value to create a new tag with the proper name and remove the bad tag. The `Try, Catch, Finally` is used to log the action and any errors that may occur

```PowerShell
    else {
        # Only for resources with 1 bad tag
        $tagValue = $resource.tags.$badTag
        $tags = $resource.tags | ConvertTo-PSFHashtable
        $tags[$GoodTagName] = $tagValue
        $tags.Remove($badTag)

        $reportEntry['BadTag'] = $badTag
        $reportEntry['BadTagValue'] = $tagValue

        try{
            $null = Get-AzResource -ResourceId $resource.id | Set-AzResource -Tag $tags -Force  -ErrorAction Stop
            $reportEntry['Action'] = "Tag updated"
        }
        catch{
            $reportEntry['Action'] = "Error"
            $reportEntry['ErrorMessage'] = $_.Exception.Message
        }

    }
```


## Full Script
{% highlight PowerShell %}
# Install-PSResource -Name az.ResourceGraph, PSFramework

$GoodTagName = "ApplicationName" # This is the proper Tag Name that you want to use for all resources

$BadTagNames = @( # This is a list of the tag names that you want to fix.
    "AplicationName"
    "Apllication"
    "App"
    "Appication Name"
    "ApplcationName"
    "ApplicatinName"
    "Application Name "
    "Application Name"
    "Application"
    "ApplicationN"
    "ApplicationNam"
    "ApplicationName "
)

$BadTagNamesQuery = ($BadTagNames | ForEach-Object { "isnotnull(tags['$_'])" }) -join " or "

# Build a query to find all resources with any of the bad tags and no good tag
$query = @"
resources
| where ($BadTagNamesQuery) and isnull(tags['$GoodTagName'])
"@
$result = Search-AzGraph -Query $query

$report = foreach ($resource in $result) {
    # This is used for logging.
    $reportEntry = [ordered]@{
        Name           = $resource.name
        ResourceGroup  = $resource.resourceGroup
        SubscriptionId = $resource.subscriptionId
        ResourceType   = $resource.type
        ResourceId     = $resource.id
    }

    $badTag = ($resource.tags | Get-Member -MemberType NoteProperty | Where-Object { $_.Name -in $BadTagNames }).Name
    if ($badTag.count -ne 1) {
        # Skip resources with multiple bad tags
        Write-PSFMessage -Level Warning -Message "Resource {0} has {1} bad tags" -StringValues $resource.id, $badTag.count
        $reportEntry['Action'] = "Error"
        $reportEntry['ErrorMessage'] = "Resource has multiple bad tags: {0}" -f ($badTag -join ", ")
    }
    else {
        # Only for resources with 1 bad tag
        $tagValue = $resource.tags.$badTag
        $tags = $resource.tags | ConvertTo-PSFHashtable
        $tags[$GoodTagName] = $tagValue
        $tags.Remove($badTag)

        $reportEntry['BadTag'] = $badTag
        $reportEntry['BadTagValue'] = $tagValue

        try{
            $null = Get-AzResource -ResourceId $resource.id | Set-AzResource -Tag $tags -Force  -ErrorAction Stop
            $reportEntry['Action'] = "Tag updated"
        }
        catch{
            $reportEntry['Action'] = "Error"
            $reportEntry['ErrorMessage'] = $_.Exception.Message
        }

    }
    # [PSCustomObject]$reportEntry | fl -Property SubscriptionId,ResourceGroup,Name, BadTag,Action  | out-host # Uncomment this line to see the report in the console
    [PSCustomObject]$reportEntry
}
$report | Export-Csv -Path "C:\temp\FixTag-$GoodTagName-$timestamp.csv"

{% endhighlight %}
