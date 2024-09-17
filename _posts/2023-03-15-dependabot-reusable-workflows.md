---
title: 'Configuring Dependabot for Reusable Workflows in GitHub'
author: Josh Johanning
date: 2023-03-15 6:30:00 -0500
description: Configuring Dependabot to keep Reusable Workflows up to date in GitHub
categories: [GitHub, Dependabot]
tags: [GitHub, Dependabot, Pull Requests, GitHub Actions, Reusable Workflows]
media_subpath: /assets/screenshots/2023-03-15-dependabot-reusable-workflows
image:
  path: dependabot-pr.png
  width: 100%
  height: 100%
  alt: A Dependabot-created pull request for a reusable workflow version update
---

## Overview

We already can use [Dependabot Version Updates](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates) for keeping marketplace actions (in addition to custom internal/private actions) up to date (see my [post](/posts/github-dependabot-for-actions/)) for more details). However, as of [March 2023](https://github.blog/changelog/2023-03-13-dependabot-updates-support-reusable-workflows-for-github-actions/), we can use Dependabot for keeping Reusable Workflows up to date as well.

## Configuration

### Authorization

My previous [post](https://github.blog/changelog/2023-03-13-dependabot-updates-support-reusable-workflows-for-github-actions/) discusses how there are two ways that you can configure Dependabot when working with resources in internal or private repositories. To summarize, you can either:

1. When Dependabot can't access a private repository, the logs allow you to [grant authorization to Dependabot to access your repository](https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization#allowing-dependabot-to-access-private-dependencies)
  - Alternatively, add it in the organization settings --> Code security and analysis --> [Grant Dependabot access to private repositories](https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization#allowing-dependabot-to-access-private-dependencies)
  - This setting requires organization admin permissions to access
  - Unfortunately, there isn't an API to automate this process, it has to  be done within the UI
2. If you plan to create a large number repositories that you want to be a source for Dependabot, creating a [Dependabot secret](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/configuring-access-to-private-registries-for-dependabot#storing-credentials-for-dependabot-to-use) (preferably as an org-level Dependabot secret) with the value of a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) (preferably a [fine-grained token](https://github.blog/2022-10-18-introducing-fine-grained-personal-access-tokens-for-github/)) that has read-access to the required repositories would be preferred
  - I don't find it as detrimental to use a personal access token as a Dependabot secret since Dependabot secrets can *only be access by Dependabot*; you can't use a GitHub Actions workflow to expose the secret accidentally/intentionally
  - The only concern would be updating the token if it expires, is revoked, or the original author doesn't have access to the repo(s) anymore

If you only have a few, and rarely increasing set of repositories for custom actions / reusable workflows, I recommend the first approach. If you have a large number of repositories and/or are creating many new repositories for actions / reusable workflows, the **second option scales better**.

### YML

For the **first approach** (authorizing Dependabot access manually), the YML configuration is no different than if you were using Dependabot to keep [marketplace actions up to date](/posts/github-dependabot-for-actions/).

```yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/" # Workflow files stored in the default location of `.github/workflows`
    schedule:
      interval: "daily"
    open-pull-requests-limit: 5
```
{: file='.github/dependabot.yml'}

You will then have to check your [Dependabot run logs](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/troubleshooting-dependabot-errors#investigating-errors-with-dependabot-version-updates) to authorize Dependabot for that repository (or [add it via the organization settings](https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization#allowing-dependabot-to-access-private-dependencies)):
![Noticing a Dependabot run failure](dependabot-error.png){: .shadow }
_A Dependabot failure since it is unable to access private/internal repositories by default_

![Granting Dependabot access via the Dependabot logs](dependabot-grant-access.png){: .shadow }
_Granting access to Dependabot for this repository_

Once you grant Dependabot access to the repository, you will see it show up in organization settings –> Code security and analysis –> [Grant Dependabot access to private repositories](https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization#allowing-dependabot-to-access-private-dependencies). Additional repositories can be added here:
![The repository now shows up under the organization --> Code security and analysis settings --> Grant Dependabot access to private repositories](dependabot-private-repositories.png){: .shadow }
_This repository now shows up under organization settings –> Code security and analysis –> Grant Dependabot access to private repositories_

For the **second option** (using a Dependabot secret), you will need to add the `registries` property to the YML configuration. The `registries` will reference `type: git` and use a [Dependabot Secret](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/configuring-access-to-private-registries-for-dependabot#storing-credentials-for-dependabot-to-use) (preferably an org-level Dependabot secret):

{% raw %}
```yml
version: 2
updates:

  - package-ecosystem: "github-actions"
    directory: "/" # Workflow files stored in the default location of `.github/workflows`
    schedule:
      interval: "daily"
    registries:
      - github
registries:
  github:
    type: git
    url: https://github.com
    username: x-access-token # username doesn't matter
    password: ${{ secrets.GHEC_TOKEN }} # dependabot secret
```
{: file='.github/dependabot.yml'}
{% endraw %}

### Results

If things are working properly, you should see a successful run in your [Dependabot run logs](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/troubleshooting-dependabot-errors#investigating-errors-with-dependabot-version-updates):

![Example using Checks to create a line annotation](dependabot-success.png){: .shadow }
_Dependabot is able to access our repositories_

And if there is a new semver version of a reusable workflow, you should see a Dependabot-created pull request:

![Example using Checks to create a line annotation](dependabot-full.png){: .shadow }
_Example of a pull request for reusable workflow created by Dependabot_

> Pro-tip: You can reply `@dependabot merge` or `@dependabot squash and merge` (among other commands) to tell Dependabot to merge the pull request.
{: .prompt-tip }

## Summary

Now we can create and properly version reusable workflows AND have our downstream users automatically be notified of version updates. This helps a ton in making it front and center for developers that there's an update they need to look at! 🎉
