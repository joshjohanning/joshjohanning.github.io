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

## Example 2 - env variable

However, there is another way. If you try to run a `gh` command without authenticating, you will see the following error message:

```
gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable. Example:
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

If you are using the `gh cli` in multiple steps or jobs in a workflow, setting the `GH_TOKEN` as an [`env`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#env) might be better:

```yml
env:
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

jobs:
  prebuild:
    runs-on: ubuntu-latest
    steps:
    - run: |
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/topics --jq='.names'
  build:
    runs-on: ubuntu-latest
    steps:
    - run: |
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/branches --jq='.[].name'
```

With this, you will notice you don't have to run `gh auth login` at all. 

You can alternatively use [`jobs.<job_id>.steps[*].env`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsenv) or [`jobs.<job_id>.env`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idenv) to set an environment variable for a particular step or job instead of the whole workflow, but this would have to be added to each step/job that you were running `gh` commands in.

{% endraw %}
