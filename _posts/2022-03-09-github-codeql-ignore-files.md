---
title: 'Ignore Files in GitHub CodeQL Analysis'
author: Josh Johanning
date: 2022-03-08 12:00:00 -0600
description: Exclude file(s) from the results of GitHub's CodeQL Analysis tool (part of GitHub Advanced Security)
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, CodeQL]
media_subpath: /assets/screenshots/2022-03-09-github-codeql-ignore-files
---

## Overview

I was recently working with a customer and we flipped on the `security-and-quality` [query suite](https://docs.github.com/en/enterprise-cloud@latest/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-code-scanning#running-additional-queries) and received a _a lot_ of results, mostly in our tests. We wanted a way to ignore these files for the purposes of the CodeQL analysis. One could argue that these should be scanned and acted on too, but you have to start somewhere, right? And there are plenty of use cases where might want to ignore a file for the purposes of the CodeQL analysis.

There are a few ways to do this, one through a filter in the UI, and another with actions. I will show you both below!

## Filter out tests in the UI

You might notice that some of the queries have the phrase _(Test)_ before the file name. CodeQL tries to determine which files are tests and which are application code automatically:

![CodeQL - Tests](codeql-test.png){: .shadow }
_CodeQL result found in a test file_

You can filter out *Test* results in the UI by adding the `autofilter:true` filter in the search bar:

![Filtering out CodeQL vulnerabilities in test files with autofilter](codeql-test-autofilter.png){: .shadow }
_Filtering out CodeQL vulnerabilities in test files with autofilter_

You cannot use `autofilter:false` to filter *only* test results, however.

## Exclude files using an action

I found the [filter-sarif](https://github.com/zbazztian/filter-sarif) action that did just what the team wanted to do - filter out all `**/*test*.js`{: .filepath} files! The action isn't published on the marketplace (yet!), but it is a public GitHub repository, therefore we can use it just like we can any other action that is published to the marketplace. 

The [docs](https://github.com/zbazztian/filter-sarif#patterns) reference some cool patterns you can use - such as ignoring all tests, but even inside of those tests, still report `sql-injection` vulnerabilities. You can find the ID for this by referencing the [CodeQL Query Help](https://codeql.github.com/codeql-query-help/) page, selecting your [language](https://codeql.github.com/codeql-query-help/javascript/), selecting the [query](https://codeql.github.com/codeql-query-help/javascript/js-sql-injection/), and find the `ID` reference. Alternatively, you can dig into the [CodeQL GitHub repoistory](https://github.com/github/codeql) to find the [query](https://github.com/github/codeql/blob/main/javascript/ql/src/Security/CWE-089/SqlInjection.ql) and reference the `@ID` value.

Here's an example:

{% raw %}

```yml
name: "CodeQL"

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '43 22 * * 3'

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        language: [ 'javascript' ]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v2
      with:
        languages: ${{ matrix.language }}

    - name: Autobuild
      uses: github/codeql-action/autobuild@v2

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2
      with:
        upload: false # disable the upload here - we will upload in a different action
        output: sarif-results

    - name: filter-sarif
      uses: advanced-security/filter-sarif@v1
      with:
        # filter out all test files unless they contain a sql-injection vulnerability
        patterns: |
          -**/*test*.js
          +**/*test*.js:js/sql-injection
        input: sarif-results/${{ matrix.language }}.sarif
        output: sarif-results/${{ matrix.language }}.sarif

    - name: Upload SARIF
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: sarif-results/${{ matrix.language }}.sarif

    # optional: for debugging the uploaded sarif
    - name: Upload loc as a Build Artifact
      uses: actions/upload-artifact@v3
      with:
        name: sarif-results
        path: sarif-results
        retention-days: 1
```
{: file='.github/workflows/codeql-analysis.yml'}

{% endraw %}

If you couldn't tell by the design of the workflow, this isn't necessarily _skipping_ these files as part of the scan - it simply strips them out from the `.sarif`{: .filepath} that's being uploaded to GitHub.

After running, you will see that the vulnerability in the excluded test file is now marked automatically as _closed:_

![Closed CodeQL vulnerabilities](codeql-closed.png){: .shadow }
_CodeQL result marked as closed after we are excluding test files_

If we click into the vulnerability, we also see the history. For example, I was testing this workflow so it opened and closed a few times:

![CodeQL vulnerability history](codeql-history.png){: .shadow }
_CodeQL result history - when it was opened, closed, and reappeared_

## Summary

You can either filter out non-application code in the UI or using an action to exclude certain files based on a pattern. Happy scanning!
