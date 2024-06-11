---
title: 'GitHub Action for adding Twistlock Scan Results to job summary'
author: Josh Johanning
date: 2024-03-23 17:30:00 -0500
description: GitHub Action to convert Twistlock's JSON scan results to markdown to add to the job summary
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Octokit]
media_subpath: /assets/screenshots/2024-03-23-github-action-for-twistlock-results
image:
  path: twistlock-job-summary-light.png
  width: 100%
  height: 100%
  alt: Twistlock Scan Results in GitHub Job Summary
---

## Overview

I was working with a customer recently who was using Twistlock / Prisma Cloud Scan to scan their Docker containers. Some CLI tools, like [Checkmarx's](https://checkmarx.com/resource/documents/en/34965-68643-scan.html#UUID-a0bb20d5-5182-3fb4-3da0-0e263344ffe7_section-idm4631465209593633552409907579) `cx scan create <params> --report-format markdown`, allow you to output the scan results in a markdown format natively. This is useful for adding the scan results to the job summary in GitHub Actions or even posting as a comment on a PR.

The [Twistlock CLI](https://pan.dev/prisma-cloud/docs/twistcli_gs/), however, does not have this feature. We can save the scan results as a JSON file, but we have to convert it to markdown ourselves. Luckily, there's a npm package [`json2md`](https://www.npmjs.com/package/json2md) that will do most of the heavy lifting! I created a custom type function using `json2md` to convert the specialized JSON that the `twistcli scan <params> --output-file scan-results.json` command creates to an easy-to-read markdown table.

## The Action

To make it easier to use this in GitHub Actions, I wrapped this in an [Action published to the marketplace](https://github.com/marketplace/actions/twistlock-prisma-scan-results-json-to-markdown). It takes the JSON scan result file as an input. Then, the action creates two markdown tables:

1. A table with high-level summary of the scan information, link to results, and sum of vulnerabilities by severity
2. A table with detailed information on each vulnerability found in the scan

The action has two outputs:

1. `summary-table`: File location to the summary table
2. `vulnerability-table`: File location to the vulnerability table

You can then use these outputs in a subsequent step to add the tables to the job summary or post as a comment on a PR.

## Usage

The sample below shows how to add the generated markdown tables to the job summary.

{% raw %}
```yml
steps:
  - run: twistcli scan <params> --output-file scan-results.json
  - name: convert-twistlock-json-results-to-markdown
    id: convert-twistlock-results
    uses: joshjohanning/twistlock-results-json-to-markdown-action@v1
    with:
      results-json-path: scanresults.json
  - name: write to job summary
    run: |
      cat ${{ steps.convert-twistlock-results.outputs.summary-table }} >> $GITHUB_STEP_SUMMARY
      cat ${{ steps.convert-twistlock-results.outputs.vulnerability-table }} >> $GITHUB_STEP_SUMMARY
```

This shows up in the job summary like this:
![Twistlock scan results in the Actions job summary](twistlock-job-summary-light.png){: .shadow }{: .light }
![Twistlock scan results in the Actions job summary](twistlock-job-summary-dark.png){: .shadow }{: .dark }
_Twistlock scan results in the Actions job summary_

## Adding as a PR Comment

I really like using the [marocchino/sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment) action to post comments on PRs:

> Create a comment on a pull request, if it exists update that comment.

To instead post the summary table as a comment in a pull request, you can use the following:

```yml
steps:
  - run: twistcli scan <params> --output-file scan-results.json
  - name: convert-twistlock-json-results-to-markdown
    id: convert-twistlock-results
    uses: joshjohanning/twistlock-results-json-to-markdown-action@v1
    with:
      results-json-path: scanresults.json
  - uses: marocchino/sticky-pull-request-comment@v2
    if: github.event_name == 'pull_request'
    with:
      path: ${{ steps.convert-twistlock-results.outputs.summary-table }}
```

You could alternatively use the [`gh cli`](https://cli.github.com/manual/gh_pr_comment) instead of a marketplace action to post the comment (though this won't update the existing comment, it will create a new comment every time the job is triggered in a PR).

```yml
steps:
  - run: twistcli scan <params> --output-file scan-results.json
  - name: convert-twistlock-json-results-to-markdown
    id: convert-twistlock-results
    uses: joshjohanning/twistlock-results-json-to-markdown-action@v1
    with:
      results-json-path: scanresults.json
  - name: create pr comment
    if: github.event_name == 'pull_request'
    run: |
      gh pr comment ${{ github.event.number }} \
        -R ${{ github.repository }}\
        -F ${{ steps.convert-twistlock-results.outputs.summary-table }}
```
{% endraw %}

## Summary

Drop a comment here or an issue or PR on my [repo](https://github.com/joshjohanning/twistlock-results-json-to-markdown-action) if you have any feedback or suggestions! Happy security scanning! 🛡️
