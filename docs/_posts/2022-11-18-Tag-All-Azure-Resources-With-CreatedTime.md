---
layout: post
title:  "Tag all Azure Resources with Created Time"
date:   2022-11-18 00:00:00 +0000
categories: Azure PowerShell Tips
---
Having a tag attached to resources that can tell when it was created can be useful in many situations.
There are policies that can do this at creation time, but what about those pre-existing resources how do we tag them?

The answer is, as usual, PowerShell!, but this time with a little bit of REST API.

{% highlight PowerShell %}
# Get Id of current subscription
$subId = (Get-AzContext).Subscription.ID

# Get list of all resources in hte subscription and select resource Name, Created Time, and Id
$ResourceList = ((Invoke-AzRestMethod -Path "/subscriptions/$subId/resources?api-version=2020-06-01&`$expand=createdTime" -Method GET).Content|ConvertFrom-Json).value| Select-Object name, createdTime, id

# Update createdTime tag on all resources
$Tags = $ResourceList | ForEach-Object -ThrottleLimit 20 -Parallel { # Updating in parallel, keeping it to 20 at a time to not overload the API.
    Update-AzTag -ResourceId $_.id -Tag @{createdTime = $_.createdTime.toString('o')} -Operation Merge #toString('o') converts the date to ISO 8601 format (e.g. 2020-07-01T00:00:00.0000000Z)
}
{% endhighlight %}

P.S: This is a real discussion that I had with some colleagues, some parts of the cmdlets above was adapted from their helpful answers 🙂

Tag away!
