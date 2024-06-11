---
title: 'GitHub: Block Pull Requests if a Vulnerable Dependency is Added'
author: Josh Johanning
date: 2022-04-06 15:00:00 -0500
description: Block Pull Requests in GitHub if you add a vulnerable dependency/package version
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, GitHub Actions, Pull Requests, Policy Enforcement, Branch Protection Rules]
media_subpath: /assets/screenshots/2022-04-06-dependency-review-action
image:
  path: dependency-review-action.png
  width: 100%
  height: 100%
  alt: Dependency Review action
---

## Overview

GitHub has added a [new Dependency Review action](https://github.com/actions/dependency-review-action) to help keep vulnerable dependencies out of your repository! One of the complaints with the way Dependabot Security Alerts works in GitHub is that it only works from the default branch. As a result, you aren't alerted that you are adding a vulnerable package until after you have already merged to the default branch. The previous solution to this was the [Dependency Review (rich diff)](https://github.blog/changelog/2021-10-05-dependency-review-is-generally-available/) in a pull request, but this was slightly hidden and there was no enforcement capabilities.

Note that the [new Dependency Review action](https://github.com/actions/dependency-review-action) still requires a GitHub Advanced Security license, as mentioned in the [GitHub Changelog blog post](https://github.blog/changelog/2022-04-06-github-action-for-dependency-review-enforcement/):
> The dependency review action is available for use in public repositories. The action is also available in private repositories owned by organizations that use GitHub Enterprise Cloud and have a license for GitHub Advanced Security.

## Dependency Review Action

The action is relatively [simple](https://github.com/actions/dependency-review-action) (no inputs as of yet) - and here's some [additional documentation](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement).

```yml
name: 'Dependency Review'
on: [pull_request]

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - name: 'Checkout Repository'
        uses: actions/checkout@v3
      - name: 'Dependency Review'
        uses: actions/dependency-review-action@v1
```
{: file='.github/workflows/dependency-review.yml'}

## Results

To try this at home, you can attempt to `"tar": "2.2.2"` to the `dependencies` section of your `package.json`{: .filepath} file. This will cause the action to fail since there are several vulnerabilities in this version of `tar`:

![Attempt at adding a vulnerable dependency](dependency-review-action.png){: .shadow }
_Dependency Review Action preventing a pull request with a vulnerable dependency added_

I think this is _much_ better than the prior option for finding/preventing vulnerable dependencies in a pull request:

![Rich diff of dependencies in a pull request](dependency-review-rich-diff.png){: .shadow }
_The previous option for dependency review in a pull request (rich diff)_

## Making this a required status check

## How To

To make this a required status check on the pull request, follow these instructions: 

1. The first thing you need is a public repository (GHAS is free for public repos) or a private repository with the GitHub Advanced Security license enabled
1. Under the Settings tab in the repository, navigate to Branches
1. Create a [branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule#creating-a-branch-protection-rule) for your default branch - check the 'Require status checks to pass before merging' box
1. Add the dependency review job as a status check - in the example above, it's `dependency-review`
   - If you don't see the `dependency-review` to add as a status check to the branch protection, **it won't appear as an option until you initiate at least one PR on the repository that triggers this job**.

![Status checks required for a pull request](pull-request-status-checks.png){: .shadow }
_Branch Protection Policy with the dependency-review status check configured_

## Summary

This [new Dependency Review action](https://github.com/actions/dependency-review-action) uses the [dependency review API endpoint](https://docs.github.com/en/rest/reference/dependency-graph#dependency-review) to determine if you are adding a new vulnerable package version to your codebase. It doesn't catch/block if there are _any_ vulnerable dependencies, only dependencies added/modified in the pull request.
