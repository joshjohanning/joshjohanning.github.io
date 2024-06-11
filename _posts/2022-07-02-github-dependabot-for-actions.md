---
title: 'Configure GitHub Dependabot to Keep Actions Up to Date'
author: Josh Johanning
date: 2022-07-02 08:00:00 -0500
description: Using Dependabot to keep Actions in GitHub Actions Workflows up to date, including how this works for custom private/internal actions within an organization
categories: [GitHub, Dependabot]
tags: [GitHub, Dependabot, Pull Requests, GitHub Actions]
media_subpath: /assets/screenshots/2022-07-02-github-dependabot-for-actions
image:
  path: dependabot-pr-post-image.png
  width: 100%
  height: 100%
  alt: Dependabot created pull requests for both marketplace and private / custom actions
---

## Overview

You probably know that Dependabot can be used to update your packages, such as NPM or NuGet, but did you also know you can use it to keep Actions up to date in your GitHub Actions Workflow?

What about for custom Actions that you have create in your organization, did you know you can use Dependabot to keep those up to date as well?

I will show you how to do this both for Actions in the public marketplace and custom actions you have created in your organization internally.

## Marketplace Actions

Configuring Dependabot with marketplace actions is pretty easy. We're using the [Dependabot Version Updates](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates) functionality, so we have to create our [`dependabot.yml`{: .filepath}](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file) file manually. There are 3 ways to do this:

1. Under the repository Settings page > Code security and analysis > Dependabot version updates, you can click the Configure button to prepopulate the `dependabot.yml`{: .filepath} file.
2. Under the repository Insights page > Dependency Graph > Dependabot > Create Config File
3. Create your own file in the `.github/dependabot.yml`{: .filepath} directory.

Whichever one you pick, you will still have to configure the `dependabot.yml`{: .filepath} file with which [package ecosystems](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#package-ecosystem) you want it to pick up.

For GitHub Actions in the marketplace, it would look like this:

```yml
version: 2
updates:

  # Maintain dependencies for GitHub Actions
  - package-ecosystem: "github-actions"
    # Workflow files stored in the default location of `.github/workflows`
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
```
{: file='.github/dependabot.yml'}

Note that even though your workflows are in the `.github/workflows`{: .filepath} directory, Dependabot still expects the `directory` on line 8 to be set to `"/"` ([documented here](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#directory)).

I also like to set `open-pull-requests-limit` explicitly, otherwise the [default maximum number of pull requests](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#open-pull-requests-limit) that will be created per package ecosystem defined is `5`.

## Custom Actions in Organization

So far, this is pretty well documented. But what is a little harder to figure out is how to use this for custom actions in private/internal repositories within an organization. There are two different ways to do this, and I will talk about both.

The first doesn't require any different configuration than the above. However, when you use a custom action, you will see an error in Dependabot:

![Error in Dependabot using custom action](dependabot-error.png){: .shadow }
_Dependabot throws an error and requests you to grant access_

You would have to grant access to _every_ custom action repository in your organization - which seems untenable. 

You can [proactively add the private action repositories](https://docs.github.com/en/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization#allowing-dependabot-to-access-private-dependencies) via Organization Settings > Code security and analysis > Grant Dependabot access to private repositories, but again, this seems less than ideal, especially since I don't think there's an API or GraphQL method of updating this. 

![Granting Dependabot access to private repos](dependabot-private-repos.png){: .shadow }
_Granting Dependabot access to private repos in organization settings_

The second way to do this is to use a Dependabot secret (GitHub PAT) and a `git` repository as a [private registry](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#git).

Here's the `dependabot.yml`{: .filepath} file:

{% raw %}
```yml
version: 2
updates:

  # Maintain dependencies for GitHub Actions
  - package-ecosystem: "github-actions"
    # Workflow files stored in the default location of `.github/workflows`
    directory: "/"
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

You'll notice the `password: ${{ secrets.GHEC_TOKEN }}` on line 17. We need to create a [Dependabot Secret](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-encrypted-secrets-for-dependabot) using a [GitHub Personal Access Token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token). I know that managing a PAT can be annoying and potentially insecure, but on the upside, Dependabot Secrets can _only_ be accessed via Dependabot, and the Dependabot implementation is essentially a black box to us. They can't be accessed maliciously / inappropriately through GitHub Actions workflow runs.

What I recommend is:

1. Creating a machine user or service account that only has read-access to the repositories in the organization - note that this will consume a license
2. Log into that account and [create a PAT](https://github.com/settings/personal-access-tokens/new) that doesn't expire with only the `repositories` scope selected
3. Create an [organization secret for Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-encrypted-secrets-for-dependabot#adding-an-organization-secret-for-dependabot) - this way all the repositories in the organization will be able to access the Dependabot secret

Notes: 
- In theory you could [create the PAT](https://github.com/settings/personal-access-tokens/new) under anyone's account since no one can dump the PAT from a GitHub Action workflow and it would be fine, but I think it's better to have it under a machine user or service account so that Dependabot will still work if / when that original person is no longer with the company
- Another reason to use a machine user or service account is that you can't currently create a PAT with a read only repo scope - it's all or nothing
- And before you ask: you unfortunately can't use a GitHub App here, at least not with the native Dependabot implementation

{% endraw %}

Once you configure the `dependabot.yml`{: .filepath} and Dependabot secret as discussed above, the next time Dependabot runs, it will create pull requests for you for both marketplace AND private / custom actions.

![Dependabot created pull requests for both marketplace and private / custom actions](dependabot-pr.png){: .shadow }
_Dependabot created pull requests for both marketplace and private / custom actions_

## What About Reusable Workflows?

You're in luck! Check out this [post](/posts/dependabot-reusable-workflows/) of mine for the details.

## Summary

Keeping marketplace actions up to date is one thing, but keeping your custom actions might be just as important! With the magic of Dependabot, you can keep your custom actions up to date without having to manually check for updates.
