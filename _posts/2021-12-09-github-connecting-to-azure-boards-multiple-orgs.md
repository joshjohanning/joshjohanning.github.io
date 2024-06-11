---
title: 'Connecting Azure Boards GitHub App to Multiple Azure DevOps Orgs/Projects'
author: Josh Johanning
date: 2021-12-09 22:00:00 -0600
description: Using the Azure Boards GitHub app to connect a single GitHub organization to multiple Azure DevOps organizations or projects for work item integration
categories: [GitHub, Integrations]
tags: [GitHub, Azure Boards, Azure DevOps, Work Items]
media_subpath: /assets/screenshots/2021-12-09-github-connecting-to-azure-boards-multiple-orgs
image:
  path: azure-boards-github.png
  width: 1458   # in pixels
  height: 838   # in pixels
  alt: Connecting multiple Azure DevOps organizations or projects to a GitHub org with the Azure Boards GitHub app
---

## Overview

We all probably know by now that there is some pretty solid first-party support for linking GitHub to Azure DevOps, specifically, Azure Boards, with the [Azure Boards GitHub app](https://docs.microsoft.com/en-us/azure/devops/boards/github/?view=azure-devops). Assuming you have the right permissions, the [setup is straight forward](https://docs.microsoft.com/en-us/azure/devops/boards/github/install-github-app?view=azure-devops). When going through and setting up the GitHub app, you'll pick the [Azure DevOps organization and project](https://docs.microsoft.com/en-us/azure/devops/boards/github/media/github-app/choose-azure-boards-project.png?view=azure-devops) that you want to link to.

And this works great - however, what isn't as clear, what if you have a GitHub organization that you want to link to multiple Azure DevOps organizations or projects? Going through the Azure Boards installation process only allows you to select a *single* Azure DevOps organization and linking to a *single* Azure DevOps project. Unless your team is following the [One Project to Rule Them All](https://colinsalmcorner.com/vsts-one-team-project-and-inverse-conway-maneuver/) strategy, you start to realize that this might not be a very tenable solution. The [documentation](https://docs.microsoft.com/en-us/azure/devops/boards/github/troubleshoot-github-connection?view=azure-devops#connecting-to-multiple-azure-devops-organizations) seems to indicate that connecting our GitHub organization to more than one Azure DevOps organization is not recommended (nor possible).

## Configuration

After a little playing around, here are the steps that I followed in order to satisfy our scenario of linking our GitHub organization to multiple Azure DevOps organizations with the Azure Boards GitHub app:

1. Install the [Azure Boards](https://github.com/marketplace/azure-boards) app to your GitHub org - select the Azure DevOps Org #1 organization that you want to link to as well as what GitHub repo(s) to link to
1. Navigate to the GitHub organization --> Settings --> [Installed GitHub Apps](https://docs.microsoft.com/en-us/azure/devops/boards/github/change-azure-boards-app-github-repository-access?view=azure-devops#change-repository-access) --> Azure Boards and make a change (i.e.: select a new repo) and click 'Save'
   - If you had selected 'All repositories' in step #1, you will have to select 'Only select repositories' instead and select the repos by hand to be able to click 'Save' on this page
1. Link it to Azure DevOps Org #2
1. For each of the Azure DevOps organizations, navigate to the project --> [project settings --> GitHub Connections](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) to verify the repo mappings are correct - [add/remove](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#add-or-remove-repositories-or-remove-a-connection) GitHub repositories if necessary

Now you have a single GitHub organization linked to multiple Azure DevOps organizations!

## Example

Here it is in action - I created a commit in each repository in GitHub. They both happen to link to `AB#1` since these are both new Azure DevOps organizations and this was the first work item I created in each. I also wanted to prove that there wouldn't be any conflicts of the links that are created.

![Azure DevOps Org #1](example-org-1.png ){: .shadow }
_Azure DevOps Org #1 linked to GitHub repo A_

![Azure DevOps Org #2](example-org-2.png ){: .shadow }
_Azure DevOps Org #2 linked to GitHub repo B_

Additionally, you can use the '[Add Repositories](https://learn.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#add-or-remove-repositories-or-remove-a-connection)' option (available after highlighting the row and using the vertical ellipses) to add additional repositories to the existing GitHub connection. Note that it's a 1-at-a-time process, so you'll have to repeat this step for each additional repository you want to add.
![Example showing multiple repositories linked](multiple-repos-linked.png ){: .shadow }
_Example showing multiple repositories linked to a GitHub connection in Azure DevOps_

## Gotchas

1. Note that *1 GitHub repo* can only be linked to *1 AzDO organization* at a time. However, the GitHub repo can be linked to multiple projects within the same AzDO org. If you try to link a GitHub repo to more than 1 AzDO org, you will see a `null` error message:
   
   ![null error message trying to add an already-linked GitHub repository](null-error.png ){: .shadow }
  _null error message trying to add a GitHub repository already linked to another Azure DevOps org_
1. You need to install/authorize both the **Azure Boards GitHub App** as well as the **Azure Boards OAuth App** in GitHub. If you don't see your GitHub org in the Azure DevOps [project settings --> GitHub Connections](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) when [adding/removing](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#add-or-remove-repositories-or-remove-a-connection) GitHub repositories, try **launching an incognito window** to force the GitHub authentication flow to be able to **grant** authorization to your GitHub organization. This is likely because your GitHub organization is using the [default OAuth policy settings](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions#about-oauth-app-access-restrictions) where an org owner has to approve OAuth app access. You will only need to do this the one time for your GitHub organization. If you aren't a GitHub org owner, you can pass along the information to an org owner to approve your [OAuth app request](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/approving-oauth-apps-for-your-organization). Note that this is different than authorizing for [SSO/SAML](https://docs.github.com/en/enterprise-cloud@latest/apps/oauth-apps/using-oauth-apps/authorizing-oauth-apps#oauth-apps-and-organizations), but you might need to do that too!
   
   ![Authorizing the Azure Boards OAuth app](authorize-github.png ){: .shadow }
   _Don't forget to authorize (grant) the **Azure Boards OAuth app** with your GitHub org as well as installing the **marketplace GitHub app**_
2. When adding new GitHub repositories to Azure DevOps, the form uses radio buttons so you can only select *1 repo at a time*. If you want to add multiple repos, you will need to use the add repositories button and select each repo individually
3. If you do too much linking back and forth to GitHub, you may run into a *secondary rate limit* issue. Either wait it out or use a different GitHub account to do the repository linking
4. We once saw that the [GitHub connection](https://docs.microsoft.com/en-us/azure/devops/boards/github/add-remove-repositories?view=azure-devops#open-project-settingsgithub-connections) was disabled, but we think that was after trying to create a new GitHub connection from a new Azure DevOps org directly in Azure DevOps without first going through GitHub - if this happens, you should be able to manually re-enable the GitHub connection

## Summary

GitHub works best when using a [single org model](https://gist.github.com/rwnfoo/3e19747f6dc2c5b9cfb0ff9c89d834b4). If you wanted to use the Azure Boards integration to link to multiple Azure DevOps organizations or projects, you might be displeased at first after reading the documentation and going through the initial setup - but hopefully the steps in this article will help you configure the integration properly!

Update: I just tested this in March 2023 and can verify that this still works as expected. I also added/clarified a few gotchas around the GitHub OAuth app authorization flow.
