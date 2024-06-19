---
title: 'GitHub Actions: Working with the GitHub Context'
author: Josh Johanning
date: 2024-06-14 10:30:00 -0500
description: Take your Actions knowledge to the next level by mastering the GitHub context
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, IssueOps]
media_subpath: /assets/screenshots/2024-06-14-github-context
image:
  path: github-context-dark.png
  width: 100%
  height: 100%
  alt: Printing the GitHub context
---

## Overview

Understanding the GitHub context, and how to print out the entire context, can be super useful when working with GitHub Actions. It provides information about the workflow run, the repository, and the event that triggered the workflow. This context is available to every step in a workflow run and can be used in expressions, conditions, and even as out of the box variables.

## Contexts

{% raw %}

There are actually several [contexts](https://docs.github.com/en/actions/learn-github-actions/contexts) in addition to the GitHub context, such as the `env`, `job`, `jobs`, `steps`, `runner`, `secrets`, `strategy`, `matrix`, `needs`, and `inputs` contexts. You probably reference some of these without knowing, such as when you're referencing a secret you would use `${{ secrets.MY_SECRET }}`, or when you are using an input from the workflow with `${{ inputs.my_input }}`.

The GitHub context is the most probably the most commonly used and most helpful context that you don't already know about that provides information about the workflow run, the repository, and the event that triggered the workflow. You might be already using the `github` context without knowing it, such as when you're referencing the repository name with `${{ github.repository }}` or the branch/tag name with `${{ github.ref_name }}`. However, there are so many more additional properties that you can use! For example, in a workflow trigger via a pull request, you can access the pull request number with `${{ github.event.pull_request.number }}`.

## Working with the GitHub Context

Whenever I'm working on a complex workflow, I always start by printing out the entire GitHub context to see what's available. This is super easy to do with a simple step like this:

```yaml
      - name: Write GitHub context to log
        env:
          GITHUB_CONTEXT: ${{ toJSON(github) }}
        run: echo "$GITHUB_CONTEXT"
```

You want to do it this way (setting the `GITHUB_CONTEXT` environment variable) for string escaping purposes (if you try to `echo '${{ toJSON(github) }}'` directly, this will sometimes error out). Also, this is a good practice to [mitigate against script injection attacks](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#good-practices-for-mitigating-script-injection-attacks) since we're directly printing user input!

This will print out the entire GitHub context to the log, which you can then use to reference the properties you need. Traverse the JSON to see what's available and what you can use in your workflow. In this example, I can use `${{ github.event.enterprise.name }}` to get the enterprise name of the repository running the workflow.

![Printing the GitHub context](github-context-dark.png){: .shadow }{: .dark }
![Printing the GitHub context](github-context-light.png){: .shadow }{: .light }
_Printing the GitHub context_

You can even use the contexts in expressions. For example, we can conditionally run a job based on the labels of an issue that triggered the workflow:

```yml
on:
  issues:
    types: [opened]

jobs:
  new-repo-create:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.labels.*.name, 'new-repo')
    steps:
      - name: print issue title
        env:
          ISSUE_TITLE: ${{ github.event.issue.title }}
        run: echo "Issue title $ISSUE_TITLE"
      - name: print issue body
        env:
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: echo "Issue body $ISSUE_BODY"
      - name: print issue author
        env:
          ISSUE_AUTHOR: ${{ github.event.issue.user.login }}
        run: echo "Issue author $ISSUE_AUTHOR"
      - name: print issue number
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: echo "Issue number $ISSUE_NUMBER"
```
{: file='.github/workflows/new-repo-create.yml'}

This can be super helpful for IssueOps and LabelOps scenarios!

> Follow this pattern of setting any user-provided input to an environment variable before using it in a script to [mitigate against script injection attacks](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#good-practices-for-mitigating-script-injection-attacks).
{: .prompt-tip }

## Exporting all Contexts

You can do the same thing to print out the other contexts as well. Here's a full workflow example:

```yml
jobs:
  write_contexts_to_log:
    runs-on: ubuntu-latest
    steps:
      - name: Write GitHub context to log
        env:
          GITHUB_CONTEXT: ${{ toJSON(github) }}
        run: echo "$GITHUB_CONTEXT"
      - name: Write job context to log
        env:
          JOB_CONTEXT: ${{ toJSON(job) }}
        run: echo "$JOB_CONTEXT"
      # this errors out if you try to access it w/o using a reusable workflow
      # - name: Write jobs context to log (reusable workflows)
      #   env:
      #     JOB_CONTEXT: ${{ toJSON(jobs) }}
      #   run: echo "$JOBS_CONTEXT"
      - name: Write steps context to log
        env:
          STEPS_CONTEXT: ${{ toJSON(steps) }}
        run: echo "$STEPS_CONTEXT"
      - name: Write runner context to log
        env:
          RUNNER_CONTEXT: ${{ toJSON(runner) }}
        run: echo "$RUNNER_CONTEXT"
      - name: Write strategy context to log
        env:
          STRATEGY_CONTEXT: ${{ toJSON(strategy) }}
        run: echo "$STRATEGY_CONTEXT"
      - name: Write matrix context to log
        env:
          MATRIX_CONTEXT: ${{ toJSON(matrix) }}
        run: echo "$MATRIX_CONTEXT"
      - name: Write env context to log
        env:
          ENV_CONTEXT: ${{ toJSON(env) }}
        run: echo "$ENV_CONTEXT"
      - name: Write secrets context to log
        env:
          SECRETS_CONTEXT: ${{ toJSON(secrets) }}
        run: echo "$SECRETS_CONTEXT"
```
{: file='.github/workflows/write-contexts-to-log.yml'}

{% endraw %}

## Summary

Once you understand the GitHub context, so many more variable combinations and expressions become available to you. This can help you write more dynamic and flexible workflows that can adapt to different scenarios. I hope this helps you take your GitHub Actions knowledge to the next level! 🚀
