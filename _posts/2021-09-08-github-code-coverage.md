---
title: 'GitHub Actions: Publish Code Coverage Summary to Pull Request and Job Summary'
author: Josh Johanning
date: 2021-09-08 22:00:00 -0500
description: Using GitHub Actions to add a code coverage summary report comment to a pull request and job summary
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Pull Requests, Code Coverage]
media_subpath: /assets/screenshots/2021-09-08-github-code-coverage
image:
  path: github-action-pr-post-image.png
  width: 100%
  height: 100%
  alt: Code Coverage summary posted to a pull request comment using an Action from the GitHub Actions Marketplace
---

## Overview

This is a follow-up to my previous post: [The Easiest Way to Generate and Publish .NET Code Coverage in Azure DevOps](/posts/azure-devops-code-coverage/)

I was familiar with adding Code Coverage to my pipelines in Azure DevOps and having a Code Coverage tab appear on the pipeline summary page, but I wasn't sure what was available for GitHub Actions. With GitHub Actions really starting to pick up steam, especially with recent additions such as [Composite Actions](https://www.colinsalmcorner.com/github-composite-actions/), I thought now would be a great time to explore.

## Adding Code Coverage to Pull Request

I found this GitHub Action in the marketplace - [Code Coverage Summary](https://github.com/marketplace/actions/code-coverage-summary). There might be others, but this one seemed simple and had the functionality I was looking for.

This post assumes you are using the `coverlet.collector` [NuGet package](https://www.nuget.org/packages/coverlet.collector/). For a refresher, see the "[the better way](/posts/azure-devops-code-coverage/#the-better-way)" section of my previous post.

Here's the relevant part of my GitHub Actions [workflow file](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/blob/main/.github/workflows/dotnet.yml):

```yml
    # Add coverlet.collector nuget package to test project - 'dotnet add <TestProject.cspoj> package coverlet
    - name: Test
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage" --logger trx --results-directory coverage

    - name: Code Coverage Summary Report
      uses: irongut/CodeCoverageSummary@v1.3.0
      with:
        filename: 'coverage/*/coverage.cobertura.xml'
        badge: true
        format: 'markdown'
        output: 'both'

    - name: Add Coverage PR Comment
      uses: marocchino/sticky-pull-request-comment@v2
      if: github.event_name == 'pull_request'
      with:
        recreate: true
        path: code-coverage-results.md

    - name: Write to Job Summary
      run: cat code-coverage-results.md >> $GITHUB_STEP_SUMMARY
```
{: file='.github/workflows/dotnet.yml'}

Note the test command here that we are using to generate the Cobertura code coverage summary file:

```bash
dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage" --logger trx --results-directory coverage
```
{: .nolineno}

The next action is the [Code Coverage Summary Report](https://github.com/irongut/CodeCoverageSummary) action: 

> CodeCoverageSummary Inputs:
> * filename: `coverage/*/coverage.cobertura.xml`{: .filepath}
> * badge: **true** &#124; false
> * format: **markdown** &#124; text
> * output: console &#124; file &#124; **both**

```yml
    - name: Code Coverage Summary Report
      uses: irongut/CodeCoverageSummary@v1.3.0
      with:
        filename: 'coverage/*/coverage.cobertura.xml'
        badge: true
        format: 'markdown'
        output: 'both'
```

> The [CodeCoverageSummary v1.3.0](https://github.com/irongut/CodeCoverageSummary/releases/tag/v1.3.0) action now supports glob pattern matching for multiple coverage files. Therefore, you don't have to use the example below with [reportgenerator](#reportgenerator) to combine the code coverage report before processing!
{: .prompt-tip }

This would be enough to show the code coverage in the action run: 
![github action code coverage report](github-action-code-coverage.png){: .shadow }
_Code Coverage Summary Report in the Action run logs_

However, the fun doesn't stop there. How useful would it be to post this to the PR so it's nice and easy for reviewers? Well, the next [action](https://github.com/marketplace/actions/sticky-pull-request-comment) shows a simple way we can add (and sticky) a PR comment with our code coverage report:

```yml
    - name: Add Coverage PR Comment
      uses: marocchino/sticky-pull-request-comment@v2
      if: github.event_name == 'pull_request'
      with:
        recreate: true
        path: code-coverage-results.md
```

Perfect - nothing for us to configure here, either. On the pull request, this comment is added: 

![github action pull request](github-action-pr.png){: .shadow }
_Code Coverage Summary Report added as a pinned comment to the Pull Request_

This is also demonstrated on my [pull request here](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/pull/2). 

You'll notice the badge along with the markdown table summarizing the code coverage report.

The nice thing with this action is that if a new commit is pushed to the PR triggering a new action run, the comment will be deleted/re-added with the updated code coverage summary.

## Adding Code Coverage to Job Summary

I think this is looking great, but what if we don't happen to create a pull request, how can we neatly see our code coverage report? Well, since May 9, 2022 (and GitHub Enterprise Server >= 3.6.0) we can use the [Job Summary](https://github.blog/changelog/2022-05-09-github-actions-enhance-your-actions-with-job-summaries/)! 

Since the CodeCoverageSummary action is already generating the markdown for us, all we have to do is append it to the [`$GITHUB_STEP_SUMMARY`](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary) environment variable. Add in the following run command to the end of the job:

```yml
    - name: Write to Job Summary
      run: cat code-coverage-results.md >> $GITHUB_STEP_SUMMARY
```

Now, it publishes to the job summary page right next to the workflow run logs! See the screenshot below:

![Job Summary in GitHub Actions](github-action-code-coverage-job-summary.png){: .shadow }
_Code Coverage Summary Report added to the job summary_

## ReportGenerator?

If you are trying to run this on a Windows runner, you will quickly notice the [Code Coverage Summary Report](https://github.com/irongut/CodeCoverageSummary) action is a Docker-based container action, [meaning it _only_ runs on Linux runners](/posts/github-container-jobs/#caveats). Or perhaps you work in a GitHub instance that uses an [Actions allow list](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization) to only allow approved actions to run. Don't worry! You can still very easily generate a nice-looking code coverage report and upload to the job summary.

 I have [documented this for Azure DevOps](/posts/azure-devops-code-coverage/#why-not-reportgenerator) in the past, but this would be the GitHub equivalent:

```yml
    - name: Create code coverage report
      run: |
        dotnet tool install -g dotnet-reportgenerator-globaltool
        reportgenerator -reports:coverage/*/coverage.cobertura.xml -targetdir:CodeCoverage -reporttypes:'MarkdownSummaryGithub,Cobertura'

    - name: Write to Job Summary
      run: cat CodeCoverage/SummaryGithub.md >> $GITHUB_STEP_SUMMARY
```
{: file='.github/workflows/dotnet.yml'}

This is what it looks like in the job summary:
![github action code coverage report](github-action-reportgenerator-job-summary.png){: .shadow }
_Code Coverage Summary Report generated by `reportgenerator` added to the job summary_

## Conclusion

Maybe not _as pretty_ as the Cobertura report shown in Azure DevOps, but just as effective! Certainly the addition of [job summaries](https://github.blog/changelog/2022-05-09-github-actions-enhance-your-actions-with-job-summaries/) makes this a better experience.

And hey, now on the GitHub Pull Request, you get to actually see the code coverage report before the *[end of the entire pipeline run](/posts/azure-devops-code-coverage/#code-coverage-tab-not-showing-up)* like in Azure DevOps 😀. 
