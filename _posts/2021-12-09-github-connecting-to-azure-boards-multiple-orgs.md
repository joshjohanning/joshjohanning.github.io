---
title: 'Connecting Azure Boards Github App to Multiple Azure DevOps Orgs'
author: Josh Johanning
date: 2021-12-09 22:00:00 -0600
description: Using the Azure Boards GitHub app to connect a single GitHub organization to multiple Azure DevOps projects or organizations for work item integration
categories: [GitHub, Integrations]
tags: [GitHub, Azure Boards, Azure DevOps, Pull Requests, Work Items]
image:
  src: /assets/screenshots/2021-12-09-github-connecting-to-azure-boards-multiple-orgs/azure-boards-github.png
  width: 1458   # in pixels
  height: 838   # in pixels
  alt: Connecting multiple Azure DevOps organizations to a GitHub organization with the Azure Boards GitHub app
---

## Overview

We all probably know by now that there is some pretty solid first-party support for linking GitHub to Azure DevOps, specifically, Azure Boards, with the [Azure Boards GitHub app](https://docs.microsoft.com/en-us/azure/devops/boards/github/?view=azure-devops). Assuming you have the right permissions, the [setup is straight forward](https://docs.microsoft.com/en-us/azure/devops/boards/github/install-github-app?view=azure-devops). When going through and setting up the GitHub app, you'll pick the [Azure DevOps organization and project](https://docs.microsoft.com/en-us/azure/devops/boards/github/media/github-app/choose-azure-boards-project.png?view=azure-devops) that you want to link to.

And this works great - however, what isn't as clear, what if you have a GitHub organization that you want to link to multiple Azure DevOps Projects or Organizations? Going through the Azure Boards installation process only allows you to select a _single_ Azure DevOps Organization and linking to a _single_ Project. Unless your team is following the [One Project to Rule Them All](https://colinsalmcorner.com/vsts-one-team-project-and-inverse-conway-maneuver/) strategy, you start to realize that this might not be a very tenable solution. The [documentation](https://docs.microsoft.com/en-us/azure/devops/boards/github/troubleshoot-github-connection?view=azure-devops#connecting-to-multiple-azure-devops-organizations) seems to indicate that connecting our GitHub organization to more than one Azure DevOps organization is not recommended (nor possible).

## Configuration

After a little playing around, here are the steps that I followed in order to satisfy our scenario of linking our GitHub organization to multiple Azure DevOps organizations with the Azure Boards GitHub app:

1. Install the [Azure Boards](https://github.com/marketplace/azure-boards) app to your GitHub org - select the Azure DevOps Org #1 organization that you want to link to as well as what GitHub repo(s) to link to
1. Navigate to the GitHub organization --> Settings --> [Installed GitHub Apps](https://docs.microsoft.com/en-us/azure/devops/boards/github/change-azure-boards-app-github-repository-access?view=azure-devops#change-repository-access) --> Azure Boards and make a change (ie select a new repo) and click 'Save'
   - If you had selected 'All repositories' in step #1, you will have to select 'Only select repositories' instead and select the repos by hand to be able to click 'Save' on this page
1. Link it to Azure DevOps Org #2
1. For each of the Azure DevOps organizations, navigate to the project --> [project settings --> GitHub Connections](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) to verify the repo mappings are correct - [add/remove](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#add-or-remove-repositories-or-remove-a-connection) GitHub repositories if necessary

Now you have a single GitHub organization linked to multiple Azure DevOps organizations!

## Example

Here it is in action - I created a commit in each repository in GitHub. They both happen to link to `AB#1` since these are both new Azure DevOps organizations and this was the first work item I created in each. I also wanted to prove that there wouldn't be any conflicts of the links that are created.

![Azure DevOps Org #1](/assets/screenshots/2021-12-09-github-connecting-to-azure-boards-multiple-orgs/example-org-1.png ){: .shadow }
_Azure DevOps Org #1 linked to GitHub repo A_

![Azure DevOps Org #2](/assets/screenshots/2021-12-09-github-connecting-to-azure-boards-multiple-orgs/example-org-2.png ){: .shadow }
_Azure DevOps Org #2 linked to GitHub repo B_

## Gotchas

1. Note that 1 GitHub repo would only be linked to 1 AzDO org/project at a time though - if you try to link a GitHub repo to more than 1 AzDO org, you get a fun `null` error message:
  ![null error message trying to add an already-linked GitHub repository](/assets/screenshots/2021-12-09-github-connecting-to-azure-boards-multiple-orgs/null-error.png ){: .shadow }
  _null error message trying to add an already-linked GitHub repository_

1. If you don't see your GitHub org in the Azure DevOps [project settings --> GitHub Connections](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) when [adding/removing](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#add-or-remove-repositories-or-remove-a-connection) GitHub repositories, try launching an incognito window to re-force the GitHub authentication flow to be able to authenticate to another GitHub repo
1. We once saw that the [GitHub connection](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) was disabled, but we think that was after trying to create a new GitHub connection from a new Azure DevOps org directly in Azure DevOps without first going through GitHub - if this happens, you should be able to manually re-enable the GitHub connection

## Summary

GitHub works best when using a [single org model](https://resources.github.com/downloads/github-guide-to-organizations.pdf). If you wanted to use the Azure Boards integration to link to multiple Azure DevOps organizations, you might be displeased at first after reading the documentation - but hopefully the steps in this article will help you configure the integration properly!
