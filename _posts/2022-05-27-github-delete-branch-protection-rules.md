---
title: 'Delete GitHub Branch Protection Rules Programmatically'
author: Josh Johanning
date: 2022-05-27 12:00:00 -0500
description: Delete GitHub Branch Protection Rules from a PowerShell script
categories: [GitHub, Scripts]
tags: [GitHub, Branch Protection Rules, Scripts]
---

## Overview

After a migration, or maybe when doing cleanup, you may want to delete branch protection rules in bulk. Instead of having to click through each branch protection rule individually, I wrote a PowerShell script that leverages the GraphQL endpoint. At the time I wrote this a few years ago, there wasn't an API for deleting branch protection rules, only GraphQL. However, there is now an [API endpoint for managing branch protection rules](https://docs.github.com/en/rest/branches/branch-protection), so if I were to re-write this, that's likely what I would use.

But this script works just fine as is to delete branch protection rules programmatically, and I thought it was time to share it!

## Script

The script is also located [here](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/delete-branch-protection-rules.ps1).

```powershell
##############################################################
# Delete branch protection rules
##############################################################

[CmdletBinding()]
param (
    [parameter (Mandatory = $true)][string]$PersonalAccessToken,
    [parameter (Mandatory = $true)][string]$GitHubOrg,
    [parameter (Mandatory = $true)][string]$GitHubRepo,
    [parameter (Mandatory = $true)][string]$PatternToDelete 
      # If you want to delete branch protection rules that start with "feature", pass in "feature*"
      # If you want to delete ALL branch protection rules, pass in "*"
)

# Example that deletes ALL branch protection rules:
# ./github-delete-branch-protection.ps1 -PersonalAccessToken "xxx" -GitHubOrg "myorg" -GitHubRepo "myrepo" -PatternToDelete "*"

# Example that deletes branch protection rules that start with feature:
# ./github-delete-branch-protection.ps1 -PersonalAccessToken "xxx" -GitHubOrg "myorg" -GitHubRepo "myrepo" -PatternToDelete "feature*"

# Set up API Header
$AuthenticationHeader = @{Authorization = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($PersonalAccessToken)")) }

####### Script #######

### GRAPHQL
# Get body
$body = @{
    query = "query BranchProtectionRule {
        repository(owner:`"$GitHubOrg`", name:`"$GitHubRepo`") { 
          branchProtectionRules(first: 100) { 
              nodes { 
                  pattern,
                  id
                      matchingRefs(first: 100) {
                          nodes {
                              name
                          }
                      }
                  }
              }
          }
      }"
}

write-host "Getting policies for repo: $GitHubRepo ..."
$graphql = Invoke-RestMethod -Uri "https://api.github.com/graphql" -Method POST -Headers $AuthenticationHeader -ContentType 'application/json' -Body ($body | ConvertTo-Json) # | ConvertTo-Json -Depth 10
write-host ""

foreach($policy in $graphql.data.repository.branchProtectionRules.nodes) {
    if($policy.pattern -like $PatternToDelete) {
        write-host "Deleting branch policy: $($policy.pattern) ..."
        $bodyDelete = @{
            query = "mutation {
                deleteBranchProtectionRule(input:{branchProtectionRuleId: `"$($policy.id)`"}) {
                clientMutationId
                }
            }"
        }
        $toDelete = Invoke-RestMethod -Uri "https://api.github.com/graphql" -Method POST -Headers $AuthenticationHeader -ContentType 'application/json' -Body ($bodyDelete | ConvertTo-Json)
        
        if($toDelete -like "*errors*") {
            write-host "Error deleting policy: $($policy.pattern)" -f red
        }
        else {
            write-host "Policy deleted: $($policy.pattern)"
        }
    }
}
```

## Usage

As an example, if I wanted to clean up all of the branch protection rules on my feature branches, the script can be called like:

```powershell
./github-delete-branch-protection.ps1 -PatternToDelete "feature*" -PersonalAccessToken "xxx" -GitHubOrg "myorg" -GitHubRepo "myrepo"
```

Alternatively, if I wanted to delete ALL branch protection rules, I can use a `"*"` wildcard to delete them all:
  
```powershell
./github-delete-branch-protection.ps1 -PatternToDelete "*" -PersonalAccessToken "xxx" -GitHubOrg "myorg" -GitHubRepo "myrepo"
```

## Output

Here's an example of an output / logs from the script: 

```
Getting policies for repo: gh-cli-get-branches-example ...

Deleting branch policy: test1 ...
Policy deleted: test1
Deleting branch policy: test2 ...
Policy deleted: test2
```

## Notes

If you have more than 100 branch protection rules that you are cleaning, update the `branchProtectionRules(first: 100)` and `matchingRefs(first: 100)`
