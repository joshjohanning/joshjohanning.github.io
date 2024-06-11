---
title: 'GitHub Actions: Building a Dynamic Matrix'
author: Josh Johanning
date: 2024-03-26 16:00:00 -0500
description: Building matrices dynamically in GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Matrix]
media_subpath: /assets/screenshots/2024-03-26-github-actions-dynamic-matrix
image:
  path: dynamic-matrix-light.png
  width: 100%
  height: 100%
  alt: Building a dynamic matrix in GitHub Actions
---

## Overview

{% raw %}
Sometimes when you're creating GitHub Actions, you have to get creative to get the job done. One such example is when you need to run several jobs at the same time, but the number of jobs and inputs are variable. If I were using Azure DevOps, I would use the [`${{ each }}` syntax in the YML](https://github.com/joshjohanning/pipeline-templates/blob/main/dotnet-core-web/dotnet-core-deploy.yml#L7) to run the job multiple times with different inputs. GitHub Actions wasn't implemented with this type YAML syntax (loops or anchors), but we can use a dynamic matrix to accomplish something similar. Using [`if:` conditionals on jobs](https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution) could work too, but then they show up as skipped in the Actions UI.

Instead, we can build a matrix dynamically. Let me show you how!

> Full disclosure, I'm piggy-backing on this post from my co-worker [@kenmuse](https://github.com/kenmuse), "[Dynamic Build Matrices in GitHub Actions](https://www.kenmuse.com/blog/dynamic-build-matrices-in-github-actions/). If you're interested in this topic, check out his post too!
{: .prompt-info }

## Example

In my example, I was tying this to an IssueOps migration approach. I had an [issue template](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository) that asked for a [list of repositories](https://github.com/joshjohanning-org/dynamic-matrix-example/issues/new?template=repos.yml). I wanted to dynamically run a job for each repository in the list.

To prepare the workflow, I parse the issue body with the [stefanbuck/github-issue-parser](https://github.com/stefanbuck/github-issue-parser) action. This action will convert my input into a predefined variable as opposed to having to write something to parse the issue template myself.

Then, I write some JavaScript to convert the list of repositories into a valid JSON object. Finally, in another job, I can then use the JSON job output of the previous job to create the dynamic matrix.

## Workflow

```yml
name: dynamic-matrix
on:
  issue_comment:
    types: [created]

jobs:
  prepare:
    runs-on: ubuntu-latest
    if: contains(github.event.comment.body, '/run-automation')
    outputs:
      repositories: ${{ steps.json.outputs.repositories }}
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
      
      - name: Parse issue body
        id: parse-issue-body
        uses: stefanbuck/github-issue-parser@v3

      - run: echo $JSON_STRING
        env:
          JSON_STRING: ${{ steps.parse-issue-body.outputs.jsonString }}

      - name: Build matrix
        uses: actions/github-script@v7
        id: json
        with:
          script: |
            let repositories = process.env.REPOSITORIES.replace(/\r/g, '').split('\n');
            let json = JSON.stringify(repositories);
            console.log(json);
            core.setOutput('repositories', json);
        env:
          REPOSITORIES: ${{ steps.parse-issue-body.outputs.issueparser_repositories }}

  run-matrix:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      matrix: 
        repository: ${{ fromJson(needs.prepare.outputs.repositories) }}
      fail-fast: false
      max-parallel: 15
    steps:
      - run: echo "${{ matrix.repository }}" # this will print the repo name
```

This will render the following workflow:
![Building a Dynamic Matrix in GitHub Actions](dynamic-matrix-light.png){: .shadow }{: .light }
![Building a Dynamic Matrix in GitHub Actions](dynamic-matrix-dark.png){: .shadow }{: .dark }
_Building a Dynamic Matrix in GitHub Actions_

## Another Workflow Example

Here's another example that I'm sharing here for reference. I thought this one was interesting because:

1. I was building my own JSON object manually (see lines 14-15 below)
2. Note the difference with the `matrix:` line (line 22) compared to lines 40-41 in the [example above](#example). Since I am already defining `repository` in the JSON array I'm building manually, I don't need to define my own object for the matrix.

```yml
name: workflow-dispatch-dynamic-matrix
on:
  workflow_dispatch:

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      repository: ${{ steps.json.outputs.repository }}
    steps:
      - name: build matrix
        id: json
        run: |
          repository='{ "repository": ["repo1","repo2","repo3","repo4"] }'
          echo "repository=$repository" >> "$GITHUB_OUTPUT"

  run-matrix:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix: ${{ fromJson(needs.prepare.outputs.repository) }}
    steps:
      - run: echo "${{ matrix.repository }}"
```
{% endraw %}

## Summary

I have built this out for a few customers now, so I wanted to finally capture this in a blog post. This can be really useful for dynamically running jobs in GitHub Actions as opposed to using [`if:` conditions to not run jobs](https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution). The problem with the `if:` condition is that it will still show the job as skipped in the Actions visualization.

Another example I have built recently was using a dynamic matrix to run a trigger N number of workflows in other repositories managed via a JSON input file stored centrally (this I will have to write as a separate blog post in itself!).

There are times where this is a no-brainer solution, but I have also seen others try to shoe-horn this into a solution where they are trying to run non-similar jobs in a matrix. This is not the intended use of a matrix I would say. Building out a dynamic matrix is a great solution for running similar jobs and the only thing that changes between the jobs is some input parameters.

I hope this helps you in your GitHub Actions journey! 🚀
