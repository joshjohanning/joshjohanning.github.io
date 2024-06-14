---
title: 'How to use gh auth login CLI Programmatically in GitHub Actions'
author: Josh Johanning
date: 2022-03-28 12:00:00 -0500
description: Using the gh cli to programmatically authenticate in GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, gh cli]
---

## Overview

Quick post here since I have to search for this every time I try to use the [`gh cli`](https://cli.github.com/) in GitHub Actions. In order to use the `gh cli`, you typically have to run `gh auth login` to authenticate. If you are running this from an interactive session (ie: on your local machine), you are provided with some prompts to easily authenticate to GitHub.com or GitHub Enterprise Server. If you try to do this from an command in a GitHub Actions, the action will just stall out and you will have to cancel since `gh auth login` is intended to be done in an interactive session.

{% raw %}

There is a [`gh auth login --with-token`](https://cli.github.com/manual/gh_auth_login) in the docs that provides an example for reading from a file, but if you're running in a GitHub Action workflow, your `${{ secrets.GITHUB_TOKEN }}` isn't going to be a file.

## Example 1 - gh auth login

Here's an example GitHub Action sample for logging into the `gh cli` and using [`gh api`](https://cli.github.com/manual/gh_api) to retrieve a repositories topics:

```yml
    steps:
    - run: |
        echo ${{ secrets.GITHUB_TOKEN }} | gh auth login --with-token
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/topics --jq='.names'
```

This works, but there's a better way that doesn't require running a `gh auth login` command at all.

## Example 2 - env variable

✨ If you try to run a `gh` command without authenticating, you will see the following error message:

> gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable. Example:
> ```yml
> env:
>   GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
> ```

With this, you will notice you don't have to run `gh auth login` at all. You can just set the `GH_TOKEN` environment variable to the value of `${{ secrets.GITHUB_TOKEN }}` and the `gh cli` will use that token to authenticate. You can set the environment variable at the [workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#env) level, [job](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idenv) level, or [step](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsenv) level.

This is an example of the least privilege approach, setting the `env` variable at the [step](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsenv) level, and allowing different steps to use different tokens if needed:

```yml
    steps:
      - run: gh issue create --title "My new issue" --body "Here are more details."
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

If you are using `gh cli` in multiple steps or jobs in a workflow, setting the `GH_TOKEN` as a [workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#env) (or https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#env) level, [job](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idenv) [`env`] variable might be more efficient:

```yml
env:
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} # setting GH_TOKEN for the entire workflow

jobs:
  prebuild:
    runs-on: ubuntu-latest
    # env: 
    #   GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} # setting GH_TOKEN for the entire job
    steps:
    - run: |
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/topics --jq='.names'
  build:
    runs-on: ubuntu-latest
    steps:
    - run: |
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/branches --jq='.[].name'
```

## Example 3 - Authenticate with GitHub App

This example combines concepts learned in this post with the [*Demystifying GitHub Apps: Using GitHub Apps to Replace Service Accounts*](/posts/github-apps/) post. 

You may want to use a GitHub app to authenticate and use the `gh cli` in a GitHub Action workflow to do something. You can manage permissions more with the GitHub App, and installing it on the org / granting access to multiple repositories whereas `${{ secrets.GITHUB_TOKEN }}` only has access to resources inside of the repository running the action. In addition, you can give the actor a more meaningful name (e.g.: `PR-Enforcer-Bot`) vs. the default `github-actions[bot]` user.

Here's an example that uses an app to create an issue in a *different* repository:

```yml
    steps:
      - uses: actions/create-github-app-token@v1
        id: app-token
        with: 
          app-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.PRIVATE_KEY }}
          # optional: owner not needed IF the app has access to the repo running the workflow
          #   if you get 'RequestError [HttpError]: Not Found 404', pass in owner
          owner: ${{ github.repository_owner }}

      - name: Create Issue
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: | 
          gh issue create --title "My new issue" --body "Here are more details." \
            -R my-org/my-repo
```

{% endraw %}
