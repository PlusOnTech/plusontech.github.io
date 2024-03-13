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

{% highlight PowerShell %}
# Required Modules: az.ResoureGraph, PSFramework
# Install-PSResource -Name az.ResoureGraph, PSFramework

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
        Write-PSFMessage -Level Warning -Message "Resource $($resource.id) has $tagCount bad tags"
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
