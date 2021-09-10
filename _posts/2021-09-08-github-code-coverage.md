---
title: 'GitHub Actions: Publish Code Coverage Summary to Pull Requests'
author: Josh Johanning
date: 2021-09-08 22:00:00 -0500
description: Using GitHub Actions to add a code coverage summary report comment to a pull request
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Pull Requests]
image:
  src: /assets/screenshots/2021-09-08-github-code-coverage/github-action-pr.png
  width: 933   # in pixels
  height: 280   # in pixels
  alt: GitHub Action Code Coverage Pull Request Comment
---

## Overview

This is a follow-up to my previous post: [The Easiest Way to Generate and Publish .NET Code Coverage in Azure DevOps](/posts/azure-devops-code-coverage/)

I was familiar with adding Code Coverage to my pipelines in Azure DevOps and having a Code Coverage tab appear on the pipeline summary page, but I wasn't sure what was available for GitHub Actions. With GitHub Actions really starting to pick up steam, especially with recent additions such as [Composite Actions](https://www.colinsalmcorner.com/github-composite-actions/), I thought now would be a great time to explore.

![github action pull request](/assets/screenshots/2021-09-08-github-code-coverage/github-action-pr.png)

## Adding Code Coverage Report to GitHub Actions

I found this GitHub Action in the marketplace - [Code Coverage Summary](https://github.com/marketplace/actions/code-coverage-summary). There might be others, but this one seemed simple and had the functionality I was looking for.

This post assumes you are using the `coverlet.collector` [NuGet package](https://www.nuget.org/packages/coverlet.collector/). For a refresher, see the "[the better way](https://josh-ops.com/posts/azure-devops-code-coverage/#the-better-way)" section of my previous post.

Here's the relevant part of my `.github/workflows` [Action file](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/blob/main/.github/workflows/dotnet.yml):

```yml
    # Add coverlet.collector nuget package to test project - 'dotnet add <TestProject.cspoj> package coverlet
    - name: Test
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage" --logger trx --results-directory coverage
      
    - name: Copy Coverage To Predictable Location
      run: cp coverage/*/coverage.cobertura.xml coverage/coverage.cobertura.xml

    - name: Code Coverage Summary Report
      uses: irongut/CodeCoverageSummary@v1.0.2
      # uses: joshjohanning/CodeCoverageSummary@v1.0.2
      with:
        filename: coverage/coverage.cobertura.xml
        badge: true
        format: 'markdown'
        output: 'both'

    - name: Add Coverage PR Comment
      uses: marocchino/sticky-pull-request-comment@v2
      if: github.event_name == 'pull_request'
      with:
        recreate: true
        path: code-coverage-results.md
```

Note the test command here that we are using to generate the Cobertura code coverage summary file:

```bash
dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage" --logger trx
```

We have to use a copy command to copy the `coverage.cobertura.xml` to a known location - the marketplace action we are using doesn't seem to support wildcards and Coverlet uses a random guid folder path.

```bash
cp coverage/*/coverage.cobertura.xml coverage/coverage.cobertura.xml
```

The next action is the [Code Coverage Summary Report](https://github.com/irongut/CodeCoverageSummary) action: 

> CodeCoverageSummary Inputs:
> * filename: **coverage/coverage.cobertura.xml**
> * badge: **true** &#124; false
> * format: **markdown** &#124; text
> * output: console &#124; file &#124; **both**

```yml
    - name: Code Coverage Summary Report
      uses: irongut/CodeCoverageSummary@v1.0.2
      with:
        filename: coverage/coverage.cobertura.xml
        badge: true
        format: 'markdown'
        output: 'both'
```

This would be enough to show the code coverage in the action run: 
![github action code coverage report](/assets/screenshots/2021-09-08-github-code-coverage/github-action-code-coverage.png)

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

![github action pull request](/assets/screenshots/2021-09-08-github-code-coverage/github-action-pr.png)

This is also demonstrated on my [pull request here](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/pull/2). 

You'll notice the badge along with the markdown table summarizing the code coverage report.

Also, if a new commit is pushed to the PR, triggering a new action run, the comment will be deleted/re-added with the updated code coverage summary.

## ReportGenerator?

The above works well if you have a single test project, but if you have more than one, see the "[Why not ReportGenerator?](https://josh-ops.com/posts/azure-devops-code-coverage/#why-not-reportgenerator) section in my previous post for the commands and rationale.

The equivalent in GitHub Actions would be:

```yml
    - name: Create code coverage report
      run: |
        dotnet tool install -g dotnet-reportgenerator-globaltool
        reportgenerator -reports:$(Agent.WorkFolder)/**/coverage.cobertura.xml -targetdir:$(Build.SourcesDirectory)/CodeCoverage -reporttypes:'Cobertura'
```

## Conclusion

Maybe not as pretty as the Cobertura report shown in Azure DevOps, but just as effective! 

And hey, now on the GitHub Pull Request, you get to actually see the code coverage report before the *[end of the entire pipeline run](https://josh-ops.com/posts/azure-devops-code-coverage/#code-coverage-tab-not-showing-up)* like in Azure DevOps :). 
