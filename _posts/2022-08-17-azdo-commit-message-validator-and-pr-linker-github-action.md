---
title: 'Azure DevOps Commit Message Validator and PR Linker GitHub Action'
author: Josh Johanning
date: 2022-08-17 13:00:00 -0500
description: Enforce that each commit in a pull request has AB# in the commit message and link all of the work items to the pull request
categories: [GitHub, Actions]
tags: [Azure DevOps, Work Items, GitHub, GitHub Actions, Pull Requests, Branch Protection Rules]
media_subpath: /assets/screenshots/2022-08-17-azdo-commit-message-validator-and-pr-linker-github-action
image:
  path: blocking-pr-post-image.png
  width: 100%
  height: 100%
  alt: Blocking a PR from merging because it's missing an AB# in the commit message
---

## Overview

I was with a client recently that was using GitHub for source control and GitHub Advanced Security, and Azure DevOps for Boards and Pipelines. [Integrating GitHub with Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/boards/github/link-to-from-github?view=azure-devops) is relatively simple for linking commits and pull requests, but there were a few pieces that we wanted to improve on. One was making sure / enforcing in the pull request that each commit contains an Azure Boards work item link with `AB#123` in the commit message. We also found that commits that contained work item links weren't automatically linked to the pull request. The pull request needs to contain `AB#123` in the pull request title or body in order for the link to be automatically created. 

Because of these limitations, I built an [action](https://github.com/joshjohanning/azdo_commit_message_validator) to be ran in a pull request to make sure that all commits have a `AB#123` link in the commit message, as well as link all corresponding work items to the pull request.


## Using the Action

The [action](https://github.com/joshjohanning/azdo_commit_message_validator) loops through each commit and:

1. makes sure it has `AB#123` in the commit message
2. if yes, add a GitHub Pull Request link to the work item in Azure DevOps

### Prerequisites

1. Create a repository secret titled `AZURE_DEVOPS_PAT` - it needs to be a full PAT
2. Pass the Azure DevOps organization to the azure-devops-organization input parameter (line no. 14 below)

### YML

This should only be triggered via pull requests.

{% raw %}
```yml
name: pr-commit-message-enforcer-and-linker

on:
  pull_request:
    branches: [ "main" ]

jobs:
  pr-commit-message-enforcer-and-linker:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Azure DevOps Commit Validator and Pull Request Linker
      uses: joshjohanning/azdo_commit_message_validator@v1
      with:
        azure-devops-organization: myorg # The name of the Azure DevOps organization
        azure-devops-token: ${{ secrets.AZURE_DEVOPS_PAT }} # "Azure DevOps Personal Access Token (needs to be a full PAT)
        fail-if-missing-workitem-commit-link: true # Fail the action if a commit in the pull request is missing AB# in the commit message
        link-commits-to-pull-request: true # Link the work items found in commits to the pull request
```
{: file='.github/workflows/pr-commit-message-enforcer-and-linker.yml'}

{% endraw %}

### Branch Protection Policy

After you create the workflow, you can add this as a status check to the branch protection policy on your default branch. If you aren't seeing the `pr-commit-message-enforcer-and-linker` job name, you might have to create a pull request that triggers the job first and then add the branch protection policy.
![Branch protection policy](branch-protection-policy.png){: .shadow }
_Configuring the status check in the branch protection policy_

Once added, if commit message(s) don't contain an `AB#123` link, the pull request will be blocked from merging.
![Status checks failing on pull request](checks-failing-on-pr.png){: .shadow }
_The status checks on the pull request are failing because of missing work item links in the commit message(s)_

## Screenshots

If a commit in the pull request is missing AB# in the commit message, the action will fail:
![Blocking the pull request because it's missing work item links](blocking-pr.png){: .shadow }
_Blocking the pull request because it's missing work item links_

The action will link all work items found in commits to the pull request:
![Linking the work items to the pull request](linking-workitem-to-pr.png){: .shadow }
_Linking the work items to the pull request_

The pull request showing along with the commit on the work item in Azure DevOps:
![Pull request](pr-link.png){: .shadow }
_Pull request link on a work item in Azure DevOps_

## Summary

The gist is that it makes sure that all commits in the pull request have an AB# link in the commit message, and that all work items found in the commits are linked to the pull request. I'm working with an undocumented API that I describe a bit more in the [README of the repository](https://github.com/joshjohanning/azdo_commit_message_validator/#how-this-works) if you're interested. Test it out - feedback's always welcome!
