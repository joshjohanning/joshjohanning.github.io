---
title: 'Use Dependabot in GitHub with Azure Artifacts'
author: Josh Johanning
date: 2022-03-10 8:00:00 -0600
description: Using Dependabot to update your dependencies that are hosted in Azure Artifacts
categories: [GitHub, Dependabot]
tags: [GitHub, Dependabot, Azure Artifacts, Azure DevOps, Pull Requests]
media_subpath: /assets/screenshots/2022-03-10-github-dependabot-with-azure-artifacts
image:
  path: dependabot-pr.png
  width: 100%
  height: 100%
  alt: Pull request from Dependabot with package residing in Azure Artifacts
---

## Overview

If you have heavy investment in Azure Artifacts, it can be hard to fully transition to [GitHub Packages](https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages). However, there is a bit of a transition. In GitHub, while you can see a list of packages the [organization level](https://docs.github.com/en/packages/learn-github-packages/viewing-packages#viewing-an-organizations-packages), the packages are installed _to a specific repository._ For further detail, here are the instructions for pushing various package ecosystems to GitHub:
- [npm](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages)
- [NuGet](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry)
- [Maven](https://docs.github.com/en/actions/publishing-packages/publishing-java-packages-with-maven)
- [Docker](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)

Alright but you might be thinking, if I'm not using GitHub Packages, won't Dependabot not work then? Well, no. [Dependabot](https://github.blog/2020-06-01-keep-all-your-packages-up-to-date-with-dependabot/) is not just for keeping your public packages up to date - Dependabot also supports private feeds, including Azure Artifacts!

## Configuration

For this to work, you just have to set up a [Dependabot secret](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/managing-encrypted-secrets-for-dependabot). I called my secret `AZURE_DEVOPS_PAT` below.

Here is the full `.github/dependabot.yml`{: .filepath} configuration:

{% raw %}

```yml
version: 2
registries:
  npm-azure-artifacts:
    type: npm-registry
    url: https://pkgs.dev.azure.com/jjohanning0798/PartsUnlimited/_packaging/npm-example/npm/registry/ 
    username: jjohanning0798
    password: ${{ secrets.AZURE_DEVOPS_PAT }}  # Must be an unencoded password
updates:
  - package-ecosystem: "npm"
    directory: "/"
    registries:
      - npm-azure-artifacts
    schedule:
      interval: "daily"
```
{: file='.github/dependabot.yml'}

{% endraw %}

## Confirming it works

Shortly after committing the `.dependabot.yml`{: .filepath} file, we can confirm it works as there's a new PR from Dependabot:
![Dependabot logs](dependabot-pr.png){: .shadow }
_Pull request created by Dependabot_

We can also look at our [Dependabot logs](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/troubleshooting-dependabot-errors#investigating-errors-with-dependabot-version-updates):

![Dependabot logs](dependabot-logs.png){: .shadow }
_Dependabot logs showing that there is a new package version from Azure Artifacts_


## Troubleshooting

### Don't use token with Azure DevOps

If you follow the [Dependabot documentation for NuGet](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates#nuget-feed) that's there today, for example, it has you use a `token` property instead of `username` and `password`:

{% raw %}

```yml
registries:
  nuget-azure-devops:
    type: nuget-feed
    url: https://pkgs.dev.azure.com/.../_packaging/My_Feed/nuget/v3/index.json
    token: ${{secrets.MY_AZURE_DEVOPS_TOKEN}} # this doesn't work
```
{: file='.github/dependabot.yml'}

{% endraw %}

If you check your [Dependabot logs](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/troubleshooting-dependabot-errors#investigating-errors-with-dependabot-version-updates), you will probably see `401` or `private_source_authentication_failure` errors. This is because Azure Artifacts needs to use basic authentication, which using the `username` and `password` fields provide. The username isn't used, but the password has to be an unencoded personal access token. 

{% raw %}

```yml
registries:
  nuget-azure-devops:
    type: nuget-feed
    url: https://pkgs.dev.azure.com/.../_packaging/My_Feed/nuget/v3/index.json
    username: octocat@example.com
    password: ${{secrets.MY_AZURE_DEVOPS_TOKEN}} # this works
```
{: file='.github/dependabot.yml'}

{% endraw %}

Alternatively, you could still use `token`, but just append a `:` at the end of the PAT as mentioned in this [issue here](https://github.com/dependabot/dependabot-core/issues/3555).

### Pull request limit

Another reason you might not be seeing your pull request from an outdated dependency in Azure Artifacts is if the pull request limit is not defined. By default, the limit is 5, so Dependabot will only create 5 pull requests for version updates as to not inundate you. If you check your pull requests, you might see you have more than 5, but some of those might be Dependabot Security Alerts, which don't count to that limit.

See the [docs](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates), but here's an example (see: `open-pull-requests-limit` on line 15):

{% raw %}

```yml
version: 2
registries:
  npm-azure-artifacts:
    type: npm-registry
    url: https://pkgs.dev.azure.com/jjohanning0798/PartsUnlimited/_packaging/npm-example/npm/registry/ 
    username: jjohanning0798
    password: ${{ secrets.AZURE_DEVOPS_PAT }}  # Must be an unencoded password
updates:
  - package-ecosystem: "npm"
    directory: "/"
    registries:
      - npm-azure-artifacts
    schedule:
      interval: "daily"
    open-pull-requests-limit: 15
```
{: file='.github/dependabot.yml'}

{% endraw %}

### Dependabot misconfiguration

If you have any other misconfiguration, such as the registry names not matching, you will be able to see from the [Dependabot logs](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/troubleshooting-dependabot-errors#investigating-errors-with-dependabot-version-updates) as well. Here's an example of such an error where the two registry names didn't match:

> The property '#/updates/0/registries' includes the "nuget-azure-artifacts" registry which is not defined in the top-level 'registries' definition

See the [docs](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates) for the configuration syntax and examples.

### Re-running Dependabot

Even though you might have the schedule set to "daily", Dependabot will run again if you push a change to the `.github/dependabot.yml`{: .filepath}. You can also run it manually at any time by navigating to:

1. Insights
2. Dependency Graph
3. Dependabot
4. Click into the last run, e.g.: "last checked 16 hours ago"
5. Check for updates

![Manually run Dependabot again](dependabot-update.png){: .shadow }
_Check for Dependabot updates again manually_

## Summary

Being able to use Dependabot with Azure Artifacts is a great way to keep your internally-created packages up to date. Teams can be notified automatically that there's a new version of the package available and after a successful build with passing unit tests, can accept and merge the PR. If a team doesn't want to use the updated version, they can simply close the PR and it won't be re-opened until a new version of the package is released. I always prefer to at least be notified of new versions, so I think this is awesome!

If the emails become too much, you can always modify your [notification settings](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications) 😀. 
