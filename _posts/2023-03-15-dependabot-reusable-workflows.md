---
title: 'Configuring Dependabot for Reusable Workflows in GitHub'
author: Josh Johanning
date: 2023-03-15 6:30:00 -0500
description: Configuring Dependabot to keep reusable workflows up to date in GitHub, including access to workflows in private and internal repositories
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

We already can use [Dependabot version updates](https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-version-updates) to keep marketplace actions and custom private or internal actions up to date. See my [post on configuring Dependabot for actions](/posts/github-dependabot-for-actions/) for details. As of [March 2023](https://github.blog/changelog/2023-03-13-dependabot-updates-support-reusable-workflows-for-github-actions/), we can use Dependabot to keep reusable workflows up to date as well.

## Configuration

### Authorization

Reusable workflows in private or internal repositories use the same Dependabot repository access model as private actions:

1. Prefer an organization or enterprise repository access policy. Organization owners can grant Dependabot access to internal repositories centrally and explicitly add private repositories. Enterprise owners can also allow access to internal repositories across organizations in the same enterprise.
2. Use a centralized organization private registry configuration when the dependency source requires separate credentials. Prefer OIDC or a dedicated, scoped, expiring read-only token.
3. Keep a `git` registry with a Dependabot secret only as a legacy fallback when the repository policy cannot cover the source.

My [Dependabot for actions post](/posts/github-dependabot-for-actions/#custom-actions-in-an-organization) is the source of truth for the current UI paths, REST APIs, plan constraints, registry authentication options, and legacy fallback.

> This authorization guidance has been updated since the post was originally published. The screenshots below preserve the March 2023 Dependabot interface for historical context.
{: .prompt-info }

### YML

When repository access is covered by an organization or enterprise policy, the YML configuration is no different than the configuration for keeping [marketplace actions up to date](/posts/github-dependabot-for-actions/).

{% raw %}
```yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/" # Workflow files stored in the default location of `.github/workflows`
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
```
{: file='.github/dependabot.yml'}
{% endraw %}

If Dependabot does not have access, the [Dependabot job logs](https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-job-logs) report the failure. When this post was originally published, the logs offered a direct authorization flow:

![Noticing a Dependabot run failure](dependabot-error.png){: .shadow }
_(March 2023 UI) A Dependabot failure because it cannot access a private or internal repository_

![Granting Dependabot access via the Dependabot logs](dependabot-grant-access.png){: .shadow }
_(March 2023 UI) Granting Dependabot access from the job logs_

![Repository listed in the March 2023 organization Dependabot access settings](dependabot-private-repositories.png){: .shadow }
_(March 2023 UI) The repository listed in the organization Dependabot access settings_

The current organization and enterprise policy paths replace this repository-by-repository workflow for internal repositories. If you still need the `git` registry fallback, use the scoped and expiring example in the [legacy fallback section of the actions post](/posts/github-dependabot-for-actions/#legacy-fallback-for-git-sources).

### Results

If things are working properly, you should see a successful run in your [Dependabot job logs](https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-job-logs):

![Successful Dependabot run in the job logs](dependabot-success.png){: .shadow }
_Dependabot is able to access our repositories_

And if there is a new semver version of a reusable workflow, you should see a Dependabot-created pull request:

![Dependabot pull request for a reusable workflow update](dependabot-full.png){: .shadow }
_Example of a pull request for reusable workflow created by Dependabot_

> Pro-tip: You can reply `@dependabot merge` or `@dependabot squash and merge` (among other commands) to tell Dependabot to merge the pull request.
{: .prompt-tip }

## Summary

We can create and properly version reusable workflows and have downstream users automatically notified of version updates. For private or internal workflows, use the current organization or enterprise repository access model rather than maintaining a broad PAT in every consuming repository.
