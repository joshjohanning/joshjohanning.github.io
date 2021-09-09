---
title: 'Authorize and Restore Azure Artifacts NuGet Packages in GitHub Actions'
author: Josh Johanning
date: 2021-01-04 14:15:00 -0600
categories: [GitHub, Actions]
tags: [Azure DevOps, NuGet, GitHub, GitHub Actions, Azure Artifacts]
---

## Summary

I needed to be able to restore my NuGet packages hosted in an Azure Artifacts instance in a GitHub Action workflow. In Azure Pipelines, it's relatively simple with the [Restore NuGet Packages](https://docs.microsoft.com/en-us/azure/devops/pipelines/packages/nuget-restore?view=azure-devops) task. However, there is not really an equivalent native Action to be able to add to the workflow to accomplish this.

I tried this 3rd party Action titled [shubham90-nugetauth](https://github.com/marketplace/actions/shubham90-nugetauth), but couldn't really get it to work. The inputs only called for the `azure-devops-org-url`, not a particular Artifact feed, so I wasn't sure how it sets up the configuration for the feeds since an organization could have multiple NuGet feeds. I will say I did not spend a lot of time on this Action, though.

What I did instead was borrow some of my scripting knowledge from my [NuGet Pusher](/posts/nuget-pusher-script/) post to programmatically add my Azure Artifacts feed as a source (with credentials) and restore the solution. This is summarized with two simple commands:

{% raw %}

```yaml
    - name: Auth NuGet
      run: nuget sources add -Name ${{ env.nuget_feed_name }} -Source ${{ env.nuget_feed_source }} -username "az" -password ${{ secrets.AZDO_PAT }} -ConfigFile ${{ env.nuget_config }}
     
    - name: Restore NuGet Packages
      run: nuget restore ${{ env.solution_file }}
```

The solution I was working with was a full .NET Framework 4.6.1, so I'm also using the `windows-latest` runner, but this should work with .NET Core with the simple tweak of using `dotnet restore` instead of `nuget restore`.

## Setup

Let's take a step back and add some things that are necessary to make this work.

1. First, we have to create a secret either at the repository or GitHub organization with an Azure DevOps PAT that has access to the Artifact feed. I called my secret: `AZDO_PAT`
1. If the secret name is named something different, make sure to modify the `nuget sources add ... -password ${{ secrets.AZDO_PAT }}` command to use the appropriate secret.
1. Next, we have to add in a few other environment variables to the GitHub Action workflow:

```yaml
env:
  solution_file: 'My.NetFramework47.App.sln'
  nuget_feed_name: 'My-Azure-Artifacts-Feed'
  nuget_feed_source: 'https://pkgs.dev.azure.com/<AZURE-DEVOPS-ORGANIZATION>/_packaging/<MY-AZURE-ARTIFACTS-FEED>/nuget/v3/index.json'
  nuget_config: '.nuget/NuGet.Config'
```

Note that my Azure Artifacts feed was scoped to the Organization level, the NuGet Feed Source url will be slightly different depending on if you used a Project feed. The source URL can be found by navigating to the Azure Artifacts feed and clicking the `Connect to Feed` button.

## Complete Workflow

Including the complete workflow for reference - the only bit custom here is the environment variables and the Auth NuGet and Restore NuGet Packages command Actions.

```yaml
# For most projects, this workflow file will not need changing; you simply need
# to commit it to your repository.
#
# You may wish to alter this file to override the set of languages analyzed,
# or to provide custom queries or build logic.
#
# ******** NOTE ********
# We have attempted to detect the languages in your repository. Please check
# the `language` matrix defined below to confirm you have the correct set of
# supported CodeQL languages.
#
name: "CodeQL"

on:
  push:
    branches: [ main ]
  pull_request:
    # The branches below must be a subset of the branches above
    branches: [ main ]
  schedule:
    - cron: '19 0 * * 2'
  workflow_dispatch:  # manual trigger

env:
  solution_file: 'My.NetFramework47.App.sln'
  nuget_feed_name: 'My-Azure-Artifacts-Feed'
  nuget_feed_source: 'https://pkgs.dev.azure.com/<AZURE-DEVOPS-ORGANIZATION>/_packaging/<MY-AZURE-ARTIFACTS-FEED>/nuget/v3/index.json'
  nuget_config: '.nuget/NuGet.Config'
  
jobs:
  analyze:
    name: Analyze
    runs-on: windows-latest

    strategy:
      fail-fast: false
      matrix:
        language: [ 'csharp' ]
        # CodeQL supports [ 'cpp', 'csharp', 'go', 'java', 'javascript', 'python' ]
        # Learn more:
        # https://docs.github.com/en/free-pro-team@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-code-scanning#changing-the-languages-that-are-analyzed

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    # Initializes the CodeQL tools for scanning.
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v1
      with:
        languages: ${{ matrix.language }}
        # If you wish to specify custom queries, you can do so here or in a config file.
        # By default, queries listed here will override any specified in a config file.
        # Prefix the list here with "+" to use these queries and those in the config file.
        # queries: ./path/to/local/query, your-org/your-repo/queries@main

    - name: Auth NuGet
      run: nuget sources add -Name ${{ env.nuget_feed_name }} -Source ${{ env.nuget_feed_source }} -username "az" -password ${{ secrets.AZDO_PAT }} -ConfigFile ${{ env.nuget_config }}
     
    - name: Restore NuGet Packages
      run: nuget restore ${{ env.solution_file }}
    
    # Autobuild attempts to build any compiled languages  (C/C++, C#, or Java).
    # If this step fails, then you should remove it and run the build manually (see below)
    - name: Autobuild
      uses: github/codeql-action/autobuild@v1

    # ℹ️ Command-line programs to run using the OS shell.
    # 📚 https://git.io/JvXDl

    # ✏️ If the Autobuild fails above, remove it and uncomment the following three lines
    #    and modify them (or add more) to build your code if your project
    #    uses a compiled language

    #- run: |
    #   make bootstrap
    #   make release

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v1
```

## Improvement Ideas

1. The `Restore NuGet Packages` command might not be needed since the `Autobuild` action performs a restore as well - therefore one may also be able to remove the `solution_file` variable
1. If your solution does not contain a NuGet.config file (`nuget_config: '.nuget/NuGet.Config'`), you may have to create a temporary config file similar to how the [NuGet Command](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/NuGetCommandV2/nugetrestore.ts#L136) task works in Azure DevOps
1. If you the `Autobuild` Action does not successfully build your project, you will have to build it manually. Using full .NET Framework, there is an additional Action that you need to add to add MSBuild to the path (`warrenbuckley/Setup-MSBuild@v1`). This Action requires `ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'` to be set because [GitHub Actions: Deprecating set-env and add-path commands](https://github.blog/changelog/2020-10-01-github-actions-deprecating-set-env-and-add-path-commands/). Also make sure to add the `/p:UseSharedCompilation=false` argument mentioned from [Troubleshooting the CodeQL workflow: No code found during the build](https://docs.github.com/en/free-pro-team@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/troubleshooting-the-codeql-workflow#no-code-found-during-the-build). Here's an example:

```yaml
    - name: Setup MSBuild Path
      uses: warrenbuckley/Setup-MSBuild@v1
      env:
        ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'

    ...
 
    - name: MSBuild Solution
      run: msbuild ${{ env.solution_file }} /p:Configuration=debug /p:UseSharedCompilation=false
```

{% endraw %}
