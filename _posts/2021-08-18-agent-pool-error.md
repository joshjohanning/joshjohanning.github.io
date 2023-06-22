---
title: 'Azure DevOps: No agent pool found with identifier xx'
author: Josh Johanning
date: 2021-08-18 7:00:00 -0600
description: A solution to the vague error message that occurs sometimes when deleting/re-creating an agent pool
categories: [Azure DevOps, Pipelines]
tags: [Azure DevOps, Azure Pipelines]
---

## Overview

We are using the Virtual Machine Scale Set (VMSS) Azure DevOps agents pretty heavily. They are perfect for our situation, needing to be able to deploy to services locked down by private endpoints while not having to individually manage agent installation/configurations.

I re-created a VMSS to use a different .vhd image, and thought I had to delete/re-create the agent pool in Azure DevOps. I learned afterwards this isn't the best way, the best way is to just edit the existing agent pool and point to your Azure Subscription --> VMSS again to re-configure it.

I had deleted agent pools within a project no problem before, but this time, I actually wasn't the one that had originally created this pool. I went to go re-create the agent pool with the exact same name and received this error message: 

> No agent pool found with identifier 59.

I suspected I was able to delete the agent pool from the project because I was a Project Administrator, but I wasn't able to delete from the organization/collection since 1) I wasn't the creator of the agent pool, they are assigned special rights and 2) I wasn't a Project Collection Administrator. 

However, I was surprised to find that I couldn't even *see* the agent pool in the list.

> https://dev.azure.com/ORG/_settings/agentpools

I tried querying the REST API and it didn't appear there.

Since I didn't know any of the Project Collection Administrators in this organization, my solution was to ask the *original creator* to go to the Organization --> Agent Pools settings to delete the agent pool so I could re-create it.

Lesson learned - don't delete, just edit :). But not a super helpful error message from Azure DevOps's part.
