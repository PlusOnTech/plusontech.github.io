---
layout: post
title:  "Find Empty Extension Attributes in AD"
date:   2022-10-19 00:00:00 +0
categories: PowerShell, ActiveDirectory
---
Extension Attributes in Active Directory are an easy way to add information about your users without extending the schema.
They are added automatically when you prepare the AD forest for Exchange and most administrators use them to store information that are not standard.

However, when it is time to add some new information you need to make sure you are not using an attribute you used somewhere else for a different reason.
The script below searches through all extensionAttributes (or any other attribute if you wish) and counts how many users have the defined.

```PowerShell
$adAttribute = @"
extensionAttribute1
extensionAttribute2
extensionAttribute3
extensionAttribute4
extensionAttribute5
extensionAttribute6
extensionAttribute7
extensionAttribute8
extensionAttribute9
extensionAttribute10
extensionAttribute11
extensionAttribute12
extensionAttribute13
extensionAttribute14
extensionAttribute15
"@ -split "`r`n" # This splits the lines by line breaks making it a string array.

$adUser = Get-ADUser -Filter * -Properties $adAttribute # Gets all users in AD, you can filter by OU if needed.
$report = foreach($item in $adAttribute){
    [PSCustomObject]@{
        Attribute = $item
        Count     = ($aduser.$item | where {-not([string]::IsNullOrEmpty($_) -or [string]::IsNullOrWhiteSpace($_))}).Count
    }
}

$report | Format-Table
```

The result is a list of attributes with number of users where they are defined!
```
Attribute            Count
---------            -----
extensionAttribute1   5091
extensionAttribute2   4567
extensionAttribute3   2606
extensionAttribute4      0
extensionAttribute5      0
extensionAttribute6      0
extensionAttribute7      0
extensionAttribute8   4874
extensionAttribute9   4892
extensionAttribute10     0
extensionAttribute11     0
extensionAttribute12     0
extensionAttribute13     0
extensionAttribute14     0
extensionAttribute15     0
```