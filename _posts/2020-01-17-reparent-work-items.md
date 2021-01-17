---
title: 'Azure DevOps: Bulk Reparent Work Items'
author: Joshua Johanning
date: 2021-01-17 16:30:00 -0600
categories: [Azure DevOps]
tags: [Azure DevOps, Work Items]
---

## Summary

Reparenting work items in the Azure DevOps UI is clunky - it can be done in mass using the parent mapping feature, but what if you have hundreds or thousands of work items split across multiple parent/child relationships? Then it becomes harder since you can't use this functionality in a query window, only from the backlog.

This is a *very* simple bash script utilizing the [Azure DevOps CLI extension](https://docs.microsoft.com/en-us/azure/devops/cli/?view=azure-devops) that can very quickly update the parent on a query of work items. I used a combination of the CLI commands with PowerShell, since PowerShell makes it super simple to use loops and JSON parsing. Before the Azure DevOps CLI, this would have to have been done with using the [APIs](https://docs.microsoft.com/en-us/rest/api/azure/devops/wit/?view=azure-devops-rest-6.0), which isn't hard, but this solution uses way fewer lines of code!

In this example, I am using a Tag on the work items I want to update.

{% raw %}

```powershell
###############################
# Reparent work item
###############################

# Prerequisites:
# az devops login (then paste in PAT when prompted)

[CmdletBinding()]
param (
    [string]$org, # Azure devops org without the URL, eg: "MyAzureDevOpsOrg"
    [string]$project, # Team project name that contains the work items, eg: "TailWindTraders"
    [string]$tag, # only one tag is supported, would have to add another clause in the $wiql, eg: "Reparent"
    [string]$newParentId # the ID of the new work item, eg: "223"
)

az devops configure --defaults organization="https://dev.azure.com/$org" project="$project"

$wiql="select [ID], [Title] from workitems where [Tags] CONTAINS '$tag' order by [ID]"

$query=az boards query --wiql $wiql | ConvertFrom-Json

ForEach($workitem in $query) {
    $links=az boards work-item relation show --id $workitem.id | ConvertFrom-Json
    ForEach($link in $links.relations) {
        if($link.rel -eq "Parent") {
            $parentId=$link.url.Split("/")[-1]
            if($parentId -ne $newParentId) {
                write-host "Unparenting" $links.id "from $parentId"
                az boards work-item relation remove --id $links.id --relation-type "parent" --target-id $parentId --yes

                write-host "Parenting" $links.id "to $newParentId"
                az boards work-item relation add --id $links.id --relation-type "parent" --target-id $newParentId
            }
            else {
                write-host "Work item" $links.id "is already parented to $parentId"
            }
        }
    }
}
```

{% endraw %}

## Improvement Ideas

* Utilize the [environment variable or from file method](https://docs.microsoft.com/en-us/azure/devops/cli/log-in-via-pat?view=azure-devops&tabs=windows) to be able to run `az devops login` in an unattended fashion
* Use the APIs if you are so inclined, but I still like to use the CLI when possible
