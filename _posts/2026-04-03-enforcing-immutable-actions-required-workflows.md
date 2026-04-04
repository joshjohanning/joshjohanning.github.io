---
title: 'Enforcing Immutable Actions with Required Workflows'
author: Josh Johanning
date: 2026-04-03 20:00:00 -0500
description: Using a required workflow to enforce that all GitHub Actions in your pull requests reference immutable releases, adding a supply chain security gate at the organization or enterprise level
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Supply Chain Security]
media_subpath: /assets/screenshots/2026-04-03-enforcing-immutable-actions-required-workflows
image:
  path: enforcing-immutable-actions-light.png
  width: 100%
  height: 100%
  alt: An immutable actions check failing on a pull request because a workflow references a mutable action release
---

## Overview

[Immutable releases](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases) are one of the best supply chain security features GitHub offers for actions. When a repository enables immutable releases, published releases cannot be modified or deleted. This prevents an attacker (or compromised account) from force-pushing a tag after users have already pinned to it.

The problem is that there is no built-in policy today to **require** that actions referenced in your workflows use immutable releases. GitHub [recently added](https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/) a policy that supports blocking actions and requiring full 40-character SHA pinning, but that only covers commit SHAs. There is nothing for immutable releases specifically.

And there is another subtlety: just because a repository's *current* release is immutable doesn't guarantee that *future* releases will be. A repo owner can disable the immutable releases setting at any time, meaning past releases stay immutable but new ones won't be. So you need a way to continuously verify that the actions you depend on are referencing immutable releases or commit SHAs.

In my [previous post on maintaining open source GitHub Actions](/posts/maintaining-oss-github-actions/#immutable-releases), I mentioned the [`ensure-immutable-actions`](https://github.com/joshjohanning/ensure-immutable-actions) action I built for exactly this purpose. In this post, I will walk through how to set it up as a **required workflow** so that every pull request across your organization (or enterprise) is automatically checked.

## The ensure-immutable-actions Action

The [`joshjohanning/ensure-immutable-actions`](https://github.com/joshjohanning/ensure-immutable-actions) action scans the workflow files in a repository and reports on every action reference it finds. It categorizes each action as:

- ✅ **First-party**: Actions from `actions/*`, `github/*`, and `octokit/*` organizations are automatically allowed by default. These organizations are [excluded in GitHub's own CodeQL immutable actions query pack](https://github.com/github/codeql/blob/main/actions/ql/extensions/immutable-actions-list/ext/immutable_actions.yml). If you want to require that even first-party actions use immutable releases or full SHA references, you can opt in with the `include-first-party` input
- ✅ **Immutable**: Third-party actions referencing an immutable release tag or a full 40-character commit SHA
- ❌ **Mutable**: Third-party actions referencing a mutable release, a major version tag without a release (e.g., `v3`), or a branch name

When `fail-on-mutable` is `true` (the default), the action fails the workflow if any mutable actions are found, preventing the PR from being merged.

### Inputs

| Input | Description | Default |
| ----- | ----------- | ------- |
| `github-token` | GitHub token for API calls | {% raw %}`${{ github.token }}`{% endraw %} |
| `fail-on-mutable` | Fail the workflow if mutable actions are found | `true` |
| `workflows` | Specific workflow files to check (comma-separated). If not specified, checks **all** workflows | All workflows |
| `exclude-workflows` | Workflow files to exclude from checks. Only applies when `workflows` is not specified | - |
| `include-first-party` | Include first-party actions (`actions/*`, `github/*`, `octokit/*`) in immutability checks instead of automatically allowing them | `false` |

The `workflows` and `exclude-workflows` inputs are useful if you want to check only specific workflows or skip certain ones.

## Setting Up the Required Workflow

To enforce this check across all repositories in your organization (or enterprise), you can configure it as a [required workflow](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-workflows-to-pass-before-merging) using repository rulesets.

### Step 1: Create the Workflow

First, create the workflow in a shared repository. I use a dedicated [`required-workflows-public`](https://github.com/joshjohanning-org/required-workflows-public) repo for this.

> The [workflow file needs to be in a repository whose visibility](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#using-a-workflow-file) matches the repositories you want to run it in: a public workflow can run on any repository, an internal workflow on internal and private repositories, and a private workflow only on private repositories. If the workflow is in an internal or private repository, you will need to [allow access](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/managing-repository-settings#allowing-access-to-components-in-a-private-repository) to it from other repositories in the organization.
{: .prompt-info }

{% raw %}
```yml
name: Check Action Immutability

on:
  pull_request:

permissions:
  contents: read

jobs:
  check-immutable:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Ensure immutable actions
        uses: joshjohanning/ensure-immutable-actions@v2
```
{: file='.github/workflows/ensure-immutable-actions.yml'}
{% endraw %}

The workflow triggers on `pull_request`, which covers PR opens, reopens, and synchronization events (new pushes to the PR branch). The action checks out the repository and then scans the workflow files for mutable action references.

### Step 2: Create the Repository Ruleset

Next, create a repository ruleset at the organization level (or enterprise level) that requires this workflow to pass before merging:

1. Navigate to your organization's (or enterprise's) **Settings** → **Rules** → **Rulesets**
2. Create a new ruleset targeting the repositories you want to enforce
3. Set the target branches - I target the **default branch** so the check runs on any PR merging into `main` / `master` / etc.
4. Under **Require workflows to pass before merging**, add the workflow from your shared repository
5. Set the enforcement status to active (or evaluate if you want to see the workflow run but not be blocking)

![Repository ruleset configured to require the immutable actions check workflow](ruleset-config-light.png){: .shadow }{: .light }
![Repository ruleset configured to require the immutable actions check workflow](ruleset-config-dark.png){: .shadow }{: .dark }
_Configuring the required workflow in a repository ruleset_

The ruleset points to the `ensure-immutable-actions.yml`{: .filepath} workflow in the `required-workflows-public` repository on the `main` branch. Every PR in the targeted repositories will now run this check automatically.

### Step 3: Create a PR and Test

Open a pull request in one of the targeted repositories to verify the check runs. The required workflow will appear as a status check on the PR, and the job summary will show the results of the immutability scan.

## What It Looks Like

### All Actions Immutable (SHA Pinned)

When all third-party actions in the repository's workflows are using immutable releases or full SHA references, the check passes:

![Immutable actions check passing with a SHA-pinned third-party action](check-passed-sha-light.png){: .shadow }{: .light }{: width="550" }
![Immutable actions check passing with a SHA-pinned third-party action](check-passed-sha-dark.png){: .shadow }{: .dark }{: width="550" }
_All actions pass: first-party actions are automatically allowed, and the third-party action uses a full SHA reference_

In this example, the `hashicorp/setup-terraform` action is pinned to a full 40-character commit SHA, which is inherently immutable. The `actions/checkout` and `github/codeql-action` actions are first-party and automatically allowed.

### All Actions First-Party

If your workflows only use first-party actions, the check passes with no third-party actions to validate:

![Immutable actions check passing with only first-party actions](check-passed-first-party-light.png){: .shadow }{: .light }{: width="550" }
![Immutable actions check passing with only first-party actions](check-passed-first-party-dark.png){: .shadow }{: .dark }{: width="550" }
_Only first-party actions in the workflow, nothing else to check_

### Mutable Action Detected

When a workflow references an action with a mutable release, the check fails and blocks the PR from merging:

![Immutable actions check failing because a workflow uses a mutable action release](check-failed-mutable-light.png){: .shadow }{: .light }{: width="550" }
![Immutable actions check failing because a workflow uses a mutable action release](check-failed-mutable-dark.png){: .shadow }{: .dark }{: width="550" }
_The check fails: `azure/webapps-deploy@v2` is using a mutable release_

The job summary clearly shows which action is the problem and why. In this case, `azure/webapps-deploy@v2` is flagged as mutable because the release is not immutable. The developer can fix this by switching to a version that has an immutable release, pinning to a commit SHA, or asking the action maintainer to enable immutable releases.

> If you want help pinning actions to full commit SHAs, check out the [`gh-pin-actions`](https://github.com/amenocal/gh-pin-actions) `gh` CLI extension. It can automatically pin all actions in your workflow files to their corresponding SHAs.
{: .prompt-tip }

## Summary

While GitHub's built-in policies cover SHA pinning, there is currently no native policy to require immutable releases for actions. By combining the [`ensure-immutable-actions`](https://github.com/joshjohanning/ensure-immutable-actions) action with a required workflow in a repository ruleset, you can enforce immutable action references across your entire organization or enterprise.

This gives you a continuous enforcement mechanism, not just a point-in-time check. Even if an action maintainer disables immutable releases in the future, the next PR that references a new mutable version of that action will be caught.

> The action itself uses immutable releases! You can verify this on the [releases page](https://github.com/joshjohanning/ensure-immutable-actions/releases). Look for the immutable 🔒 badge on each release.
{: .prompt-info }

For more context on immutable releases and how I use this in my own action maintenance workflow, check out my post on [how I maintain my open source GitHub Actions](/posts/maintaining-oss-github-actions/#immutable-releases).

Constant vigilance! 👀
