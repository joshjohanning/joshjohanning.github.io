---
title: 'Configure GitHub Dependabot to Keep Actions Up to Date'
author: Josh Johanning
date: 2022-07-02 08:00:00 -0500
description: Using Dependabot to keep GitHub Actions workflows up to date, including current access options for private and internal actions
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

You probably know that Dependabot can be used to update your packages, such as npm or NuGet, but did you also know you can use it to keep actions up to date in your GitHub Actions workflows?

What about custom actions that you have created in your organization? Dependabot can keep those up to date as well.

I will show you how to do this for actions in the public marketplace and custom actions in private or internal repositories.

## Marketplace Actions

Configuring Dependabot with marketplace actions is pretty easy. We're using [Dependabot version updates](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configure-version-updates), so we have to create a [`dependabot.yml`{: .filepath}](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference) file. There are three ways to do this:

1. Under the repository **Settings** page > **Advanced Security** > **Dependabot version updates**, click **Enable** to prepopulate the `dependabot.yml`{: .filepath} file.
2. Under the repository **Insights** page > **Dependency graph** > **Dependabot**, click **Create config file**.
3. Create `.github/dependabot.yml`{: .filepath} yourself.

Whichever one you pick, you will still have to configure the `dependabot.yml`{: .filepath} file with the [package ecosystems](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#package-ecosystem-) you want it to pick up.

For GitHub Actions in the marketplace, it would look like this:

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
    open-pull-requests-limit: 10
```
{: file='.github/dependabot.yml'}
{% endraw %}

Note that even though your workflows are in the `.github/workflows`{: .filepath} directory, Dependabot still expects `directory` to be set to `"/"` ([documented here](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#directory--)).

I also like to set `open-pull-requests-limit` explicitly. Otherwise, the [default maximum number of pull requests](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#open-pull-requests-limit--) created per package ecosystem is `5`.

## Custom Actions in an Organization

Custom actions stored in private or internal repositories need one additional piece: Dependabot must be allowed to read the repository that contains the action. The preferred solution is now a repository access policy, not a personal access token (PAT) in every consuming repository.

If access is missing, the Dependabot job reports an error like this:

![Error in Dependabot using custom action](dependabot-error.png){: .shadow }
_Dependabot throws an error and requests you to grant access_

### Organization and Enterprise Repository Access

For dependencies hosted in the same organization, an organization owner can grant Dependabot access centrally. In the organization, go to **Settings** > **Security** > **Advanced Security** > **Global settings**, then find **Grant Dependabot access to repositories**. Set the default access level to **Internal** to cover all current and future internal repositories, or explicitly add private repositories that host actions.

On GitHub Enterprise Cloud, an enterprise owner can also enable internal access across organizations in the same enterprise. Go to the enterprise **Policies** page > **Advanced Security** > **Grant Dependabot access to repositories**, and select internal access. This is useful when the workflow repository and the internal action repository are in different organizations.

The [Dependabot repository access REST API](https://docs.github.com/en/rest/dependabot/repository-access) supports both organization and enterprise administration. For example, `PUT /orgs/{org}/dependabot/repository-access/default-level` can set the organization default to `internal`, while `PUT /enterprises/{enterprise}/dependabot/repository-access/default-level` applies across organizations.

> Repository visibility and licensing still matter. **Internal** repositories require an enterprise account. Private repositories are available on other plans, but each private action repository must be explicitly granted unless it is covered by an applicable policy. The cross-organization internal repository policy is available on GitHub.com and GitHub Enterprise Cloud; for GitHub Enterprise Server, verify that your installed version includes it.
{: .prompt-info }

![Granting Dependabot access to private repos](dependabot-private-repos.png){: .shadow }
_The older organization UI for granting Dependabot access to repositories_

No registry entry or PAT is needed in `dependabot.yml`{: .filepath} when the repository access policy covers the action repository. The original `github-actions` update block is enough.

### Centralized Private Registry Configuration

Repository access and private registry access solve different problems. Use the repository policy above for actions stored in GitHub repositories. Use a [private registry configuration](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/manage-your-dependency-security/configure-access-to-private-registries) for package feeds or Git sources that require separate credentials.

Organization owners can define these centrally under **Organization Settings** > **Security** > **Secrets and variables** > **Private registries** > **New private registry**. Select which repositories can use the configuration: all repositories, private and internal repositories, or selected repositories. Organization configurations can also be managed with the [private registries REST API](https://docs.github.com/en/rest/private-registries/organization-configurations), starting with `POST /orgs/{org}/private-registries`.

Prefer the safest authentication method supported by the registry:

1. Use **OIDC** when available. Dependabot receives short-lived credentials for each update job instead of storing a long-lived secret. Current organization-level OIDC providers include Azure, AWS CodeArtifact, Cloudsmith, Google Artifact Registry, and JFrog Artifactory.
2. Otherwise, use a dedicated, read-only registry token with the smallest repository or package scope and a short expiration. Store it in the centralized organization registry configuration.
3. Use username and password only when the registry does not support a safer token or OIDC option.

> Organization-level private registries are available on GitHub.com and GitHub Enterprise Cloud. GitHub Enterprise Server support depends on the installed release, and OIDC/provider support can differ by version. Check the documentation for your GHES version before designing around it.
{: .prompt-tip }

### Legacy Fallback for Git Sources

Keep the `git` registry pattern only when a repository access policy cannot cover the source, such as a private Git dependency outside the enterprise or an older GitHub Enterprise Server release. It should not be the default for same-organization or same-enterprise internal actions.

In that case, define the credential once as an organization Dependabot secret under **Organization Settings** > **Security** > **Secrets and variables** > **Dependabot**, restrict its repository access, and reference it from `dependabot.yml`{: .filepath}:

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
    username: dependabot-read
    # Organization Dependabot secret containing the scoped, expiring PAT
    password: ${{ secrets.ORG_GITHUB_READ_TOKEN }}
```
{: file='.github/dependabot.yml'}
{% endraw %}

Use a fine-grained PAT when the target supports it, grant read-only **Contents** access only to the required repositories, set an expiration, and rotate it. A classic PAT should be the last resort because its `repo` scope is broad. A machine user can avoid tying the fallback to an employee, but it consumes a paid seat for private or internal repository access and does not make a non-expiring token safe. GitHub App installation tokens are short-lived, but native Dependabot cannot mint a fresh installation token inside a static `dependabot.yml`{: .filepath} registry definition, so they are not a direct substitute in this fallback.

> Dependabot secrets are encrypted and are not exposed directly to workflow steps, but that is not a reason to use a broad or non-expiring credential. Treat the token as a production secret and scope, expire, audit, and rotate it accordingly.
{: .prompt-warning }

Once repository or registry access is configured, the next Dependabot run can create pull requests for both marketplace and private or internal actions.

![Dependabot created pull requests for both marketplace and private / custom actions](dependabot-pr.png){: .shadow }
_Dependabot created pull requests for both marketplace and private / custom actions_

## What About Reusable Workflows?

You're in luck! Check out this [post](/posts/dependabot-reusable-workflows/) of mine for the details.

## Summary

Keeping marketplace actions up to date is one thing, but keeping custom actions might be just as important. Prefer organization or enterprise repository access for GitHub-hosted actions, centralize private registry configuration when credentials are required, and reserve a tightly scoped expiring PAT for legacy cases that the native access model cannot cover.
