---
title: 'How to use gh auth login CLI Programmatically in GitHub Actions'
author: Josh Johanning
date: 2022-03-28 12:00:00 -0500
description: Using the gh cli to programmatically authenticate in GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, gh cli]
---

## Overview

Quick post here since I have to search for this every time I try to use the [`gh cli`](https://cli.github.com/) in GitHub Actions. In order to use the `gh cli`, you have to run `gh auth login` to login. If you are running this from an interactive session (ie: on your local machine), you are provided with some prompts to easily authenticate to GitHub.com or GitHub Enterprise Server. If you try to do this from an command in a GitHub Actions, the action will just stall out and you will have to cancel since `gh auth login` is intended to be done in an interactive session. 

{% raw %}

There is a [`gh auth login --with-token`](https://cli.github.com/manual/gh_auth_login) in the docs that provides an example for reading from a file, but if you're running in a GitHub Action workflow, your `${{ secrets.GITHUB_TOKEN }}` isn't going to be a file.

## Example

Here's an example GitHub Action sample for logging into the `gh cli` and using [`gh api`](https://cli.github.com/manual/gh_api) to retrieve a repositories topics: 

```yml
    steps:
    - run: |
        echo ${{ secrets.GITHUB_TOKEN }} | gh auth login --with-token
        gh api -X GET /repos/${{ GITHUB.REPOSITORY }}/topics --jq='.names'
```

{% endraw %}
