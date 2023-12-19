---
title: 'Authorize and Restore Azure Artifacts NuGet Packages in GitHub Actions'
author: Josh Johanning
date: 2021-01-04 14:15:00 -0600
description: Authenticate to Azure Artifacts from GitHub Actions for builds and code scanning workflows
categories: [GitHub, Actions]
tags: [Azure DevOps, NuGet, GitHub, GitHub Actions, Azure Artifacts, Artifactory, CodeQL]
---

## Summary

While this post is geared towards Azure DevOps and Azure Artifacts, this approach will work for any third-party feed that requires authentication (like [Artifactory](#artifactory)!).

I needed to be able to restore my NuGet packages hosted in an Azure Artifacts instance in a GitHub Action workflow. In Azure Pipelines, it's relatively simple with the [Restore NuGet Packages](https://docs.microsoft.com/en-us/azure/devops/pipelines/packages/nuget-restore?view=azure-devops) task. This task dynamically creates a NuGet config with the proper authentication details to Azure Artifacts. In GitHub Actions, there isn't a native action readily available for us to accomplish this. 

I tried the [shubham90-nugetauth](https://github.com/marketplace/actions/shubham90-nugetauth) marketplace action, but I couldn't get it to work. The inputs only called for the `azure-devops-org-url`, not a particular Artifact feed, so I wasn't sure how it sets up the configuration for the feeds since an organization could have multiple NuGet feeds. There is another action that looks promising, [GitHub NuGet Private Source Authorisation](https://github.com/marketplace/actions/github-nuget-private-source-authorisation), but I decided to use the command line for increased flexibility.

What I did instead was borrow some of my scripting knowledge from my [NuGet Pusher](/posts/nuget-pusher-script/) post to programmatically add my Azure Artifacts feed as a source (with credentials) and restore the solution. This is summarized with a few simple commands:

{% raw %}

```yaml
- name: Auth NuGet
  run: |
    dotnet nuget add source ${{ env.nuget_feed_source }} \
      --name ${{ env.nuget_feed_name }} \
      --username az \
      --password ${{ secrets.AZURE_DEVOPS_TOKEN }} \
      --configfile ${{ env.nuget_config }} \ # required if have config file
      --store-password-in-clear-text # required if using Linux
  
- name: Restore NuGet Packages
  run: dotnet restore ${{ env.solution_file_path }}
```

Notes:
- This should work with either .NET Core as well as full .NET Framework on both Linux and Windows
- On Linux runners, you need to use `--store-password-in-clear-text`{: .code} - not required on Windows
- The `--configfile` argument is optional - if not specified, there is a [hierarchy involved](https://docs.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior):
    1. It will first try to use the `NuGet.config`{: .filepath} in the current working directory first
    2. Next, it will use the local user `NuGet.config`{: .filepath} in `%appdata%\NuGet\NuGet.Config`{: .filepath} (Windows) or `~/.nuget/NuGet.Config`{: .filepath} (Linux/Mac, depending on distro)
    3. Note: You cannot add a source to the `NuGet.config`{: .filepath} with a name that already exists - either add the source with a new source name or run [`dotnet nuget remove source`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-remove-source) to remove the source first
- If the `.sln`{: .filepath} is in the root (or current working directory), you can simply run `dotnet restore` without the solution path as well
- Reference the [`dotnet nuget add source`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-add-source) and [`dotnet restore`](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-restore) docs for more information

## Setup

Let's take a step back and add some things that are necessary to make this work.

1. First, we have to [generate an Azure DevOp Personal Access Token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows#create-a-pat) to 
1. Next, we have to [create a secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) either at the repository or GitHub organization with an Azure DevOps PAT that has access to the Artifact feed. I called my secret: `AZURE_DEVOPS_TOKEN`
2. For reusability and ease, let's add in a few environment variables to the GitHub Action workflow:

```yaml
env:
  solution_file: 'My.Core.App.sln'
  nuget_feed_name: 'My-Azure-Artifacts-Feed'
  nuget_feed_source: 'https://pkgs.dev.azure.com/<AZURE-DEVOPS-ORGANIZATION>/_packaging/<MY-AZURE-ARTIFACTS-FEED>/nuget/v3/index.json'
  nuget_config: '.nuget/NuGet.Config'
```

Note that my Azure Artifacts feed was scoped to the Organization level, the NuGet Feed Source url will be slightly different depending on if you used a Project feed. The source URL can be found by navigating to the Azure Artifacts feed and clicking the **Connect to Feed** button.

## Complete Workflow

Including the complete code scanning workflow for reference - the only bit custom here is the environment variables and the Auth NuGet and Restore NuGet Packages run commands:

```yaml
name: "CodeQL"

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '19 0 * * 2'
  workflow_dispatch:  # manual trigger

env:
  solution_file: 'My.Core.App.sln'
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

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    # Initializes the CodeQL tools for scanning.
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v2
      with:
        languages: ${{ matrix.language }}

    # # If exists, remove existing AzDO NuGet source that doesn't have authentication
    # - name: Remove existing entry from NuGet config
    #   run: | 
    #     dotnet nuget remove source ${{ env.nuget_feed_name }} \
    #       --configfile ${{ env.nuget_config }}

    - name: Auth NuGet
      run: | 
        dotnet nuget add source ${{ env.nuget_feed_source }} \
          --name ${{ env.nuget_feed_name }} \
          --username az \
          --password ${{ secrets.AZURE_DEVOPS_TOKEN }} \
          --configfile ${{ env.nuget_config }}
     
    - name: Restore NuGet Packages
      run: dotnet restore ${{ env.solution_file_path }}
    
    # Autobuild attempts to build any compiled languages  (C/C++, C#, or Java).
    # If this step fails, then you should remove it and run the build manually (see below)
    - name: Autobuild
      uses: github/codeql-action/autobuild@v2

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2
```
{: file='.github/workflows/codeql-analysis.yml'}

If I was running on `ubuntu-latest`, the only change would be to add the `--store-password-in-clear-text`argument:

```yml
    - name: Auth NuGet
      run: | 
        dotnet nuget add source ${{ env.nuget_feed_source }} \
          --name ${{ env.nuget_feed_name }} \
          --username az --password ${{ secrets.AZURE_DEVOPS_TOKEN }} \
          --configfile ${{ env.nuget_config }} \
          --store-password-in-clear-text
```
{: file='.github/workflows/codeql-analysis.yml'}

## When would you have to use the dotnet sources remove command?

You may have noticed a commented-out run command in the above workflow:

```yaml
# If exists, remove existing AzDO NuGet source that doesn't have authentication
- name: Remove existing entry from NuGet config
  run: | 
    dotnet nuget remove source ${{ env.nuget_feed_name }} \
      --configfile ${{ env.nuget_config }}
```
{: file='.github/workflows/codeql-analysis.yml'}

If your `NuGet.config`{: .filepath} already has an Azure DevOps entry, you will need to remove it with [dotnet nuget remove source](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-remove-source) otherwise you will likely see `401 Unauthorized` errors during the `dotnet restore`. This is because that entry doesn't (or at least shouldn't!) have any credentials associated with it committed into source control, so it essentially tries to access it anonymously and will fail.

Also, we have to remove it because we cannot add a sources entry to the `NuGet.config`{: .filepath} with the same name.

## Improvement Ideas / Notes

1. If your solution does not contain a `NuGet.config`{: .filepath} file, you may have to create a temporary config file similar to how the [NuGet Command task](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/NuGetCommandV2/nugetrestore.ts#L136) works in Azure DevOps
  - Alternatively, simply omitting the `--configfile` argument will use the [local user](https://docs.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior) `NuGet.config`{: .filepath}
  - Using the local `NuGet.config`{: .filepath} will certainly work with GitHub-hosted runners since it's a fresh instance each time, but you may run into conflicts if you're on a shared self-hosted runner
  - This [marketplace action](https://github.com/marketplace/actions/github-nuget-private-source-authorisation) uses a [local `NuGet.config`{: .filepath}](https://github.com/StirlingLabs/GithubNugetAuthAction/blob/main/action.sh#L25:L34) by default
1. The `Restore NuGet Packages` command might not be needed since the `Autobuild` action performs a restore as well - therefore one may also be able to remove the `solution_file` variable - but I always like to have an explicit task for restoring packages so I know exactly if that failed
2. If you the `Autobuild` Action does not successfully build your project for code scanning, you will have to build it manually. Using full .NET Framework, there is an additional action that you need to add to add MSBuild to the path ([`microsoft/setup-msbuild@v1`](https://github.com/marketplace/actions/setup-msbuild)). Here's an example:

```yaml
- name: Add msbuild to PATH
  uses: microsoft/setup-msbuild@v1

- name: MSBuild Solution
  run: msbuild ${{ env.solution_file }} /p:Configuration=release
```

## Artifactory

I've seen a few instances where a team is using an [API key to access Artifactory](https://www.jfrog.com/confluence/display/JFROG/NuGet+Repositories#NuGetRepositories-NuGetAPIKeyAuthentication), so the command is slightly different:

```yaml
- name: Auth NuGet
  run: nuget setapikey admin:${{ secrets.API_KEY }} -Source Artifactory
  
- name: Restore NuGet Packages
  run: dotnet restore ${{ env.solution_file_path }}
```

Notes:
- The `-ConfigFile` argument can optionally be used to specify a `NuGet.config`{: .filepath} file
- Reference the [`nuget setapikey`](https://docs.microsoft.com/en-us/nuget/reference/cli-reference/cli-ref-setapikey) docs for more information

{% endraw %}
