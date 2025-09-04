---
title: 'Exporting GitHub Actions Dependency Data for Your Organization'
author: Josh Johanning
date: 2025-08-24 09:30:00 -0500
description: 'Compare three methods for getting GitHub Actions usage data for organization governance: The Dependency Insights view in GitHub, @stoe/action-reporting-cli, and my custom SBOM script'
categories: [GitHub, Actions]
tags: [GitHub Actions, SBOM, gh cli]
media_subpath: /assets/screenshots/2025-08-24-github-actions-export-actions-usage
image:
  path: actions-usage-sbom-by-version-light.png
  width: 100%
  height: 100%
  alt: GitHub Actions usage reporting comparison
---

## Overview

In my [previous post on GitHub Actions Allow Lists](/posts/github-actions-allow-list-as-code/), I discussed how to manage which Actions your organization can use through configuration as code. But before you implement an allow list, you might ask: **What Actions are my organization actually using?**

Whether you're building an allow list, conducting a security audit, hunting down deprecated versions, or just want to know what's actually running in your CI/CD pipelines, you need visibility into which GitHub Actions are being used across your repositories.

This post covers three different approaches for getting that data:

1. **[GitHub's Dependency Insights](#method-1-githubs-dependency-insights-view-only)** - Native viewing capabilities (but no export)
2. **[@stoe/action-reporting-cli](#method-2-stoeaction-reporting-cli-full-featured-solution)** - Full-featured CLI tool with multiple export formats (csv, json, or markdown)
3. **[My Custom Software Bill of Materials (SBOM) Script](#method-3-custom-sbom-script-my-lightweight-solution)** - Lightweight shell script for automated reporting on Actions usage with capabilities to resolve SHAs to tag versions (exports to csv or markdown)

> If you want to skip ahead, use the links above to jump to any of these methods.
{: .prompt-tip }

## Why This Matters

Let's quickly recap why Actions usage reporting is important:

1. **Security & Compliance**: Know your third-party dependencies and assess potential security risks
2. **Governance Planning**: Make informed decisions about which Actions to allow or restrict
3. **Dependency Management**: Track Action versions and identify outdated or deprecated Actions (like `actions/upload-artifact` and `actions/download-artifact` versions earlier than `v4`)
4. **Supply Chain Visibility**: Build a Software Bill of Materials (SBOM) for your CI/CD pipeline

## Three Methods for Getting Actions Usage Data

### Method 1: GitHub's Dependency Insights (View-Only)

GitHub's [Dependency Insights](https://docs.github.com/en/enterprise-cloud@latest/organizations/collaborating-with-groups-in-organizations/viewing-insights-for-dependencies-in-your-organization) feature provides visibility into dependencies used across your organization. You'll need to filter specifically for GitHub Actions to see only Actions usage:

![Dependency Insights screenshot showing Actions usage](dependency-insights-light.png){: .light }
![Dependency Insights screenshot showing Actions usage](dependency-insights-dark.png){: .dark }
*GitHub's native Dependency Insights showing Actions usage across an organization*

While this view provides excellent visibility, it has limitations:

- **No export functionality**: You can view the data but not export it for further analysis
- **Limited filtering**: Basic filtering options compared to programmatic approaches
- **No historical data**: Shows current state but lacks trend analysis

That's where the automated tools come in handy - they give you more control and can export data for deeper analysis.

### Method 2: @stoe/action-reporting-cli (Full-Featured Solution)

The [`@stoe/action-reporting-cli`](https://github.com/stoe/action-reporting-cli) tool by Stefan Stölzle provides comprehensive GitHub Actions reporting capabilities.

> See my [automated workflow implementation](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/.github/workflows/action-usage.yml) that runs this tool on a schedule and save the outputs back to the repository.
{: .prompt-tip }

**Key features:**

- **Multiple export formats**: CSV, JSON, and Markdown outputs
- **Comprehensive data collection**: in addition to what actions are used, [can also report](https://github.com/stoe/action-reporting-cli?tab=readme-ov-file#report-content-options) on secrets, variables, permissions, listeners (workflow triggers), and/or runners
- **Flexible scope options**: run for an entire enterprise (can't use GitHub App though), organization, or a single repository
- **Advanced filtering**: exclude GitHub-created actions, unique actions reporting, and ability to exclude archived and forked repositories

**Sample output**:

> owner | repo | name | workflow | state | created_at | updated_at | last_run_at | uses
> --- | --- | --- | --- | --- | --- | --- | --- | ---
> joshjohanning-org | .github |  | [.github/workflows/update-organization-readme-badges.yml](https://github.com/joshjohanning-org/.github/blob/HEAD/.github/workflows/update-organization-readme-badges.yml) | active | 2024-05-23T16:58:49.000Z | 2024-05-23T16:58:49.000Z | 2025-08-17T07:07:40.000Z | <ul><li>[actions/checkout](https://github.com/actions/checkout) <code>v4</code></li><li>[actions/create-github-app-token](https://github.com/actions/create-github-app-token) <code>v2</code></li><li>[joshjohanning/organization-readme-badge-generator](https://github.com/joshjohanning/organization-readme-badge-generator) <code>v1</code></li></ul>
> joshjohanning-org | issueops-samples |  | [.github/workflows/delete-repos-delete.yml](https://github.com/joshjohanning-org/issueops-samples/blob/HEAD/.github/workflows/delete-repos-delete.yml) | active | 2023-11-08T16:05:40.000Z | 2025-04-02T15:48:27.000Z | 2025-08-13T14:57:36.000Z | <ul><li>[actions/checkout](https://github.com/actions/checkout) <code>v5</code></li><li>[issue-ops/parser](https://github.com/issue-ops/parser) <code>76d5aa095754de1493cbe41934484c4287e16350</code></li><li>[actions/create-github-app-token](https://github.com/actions/create-github-app-token) <code>v2</code></li><li>[actions/github-script](https://github.com/actions/github-script) <code>v7</code></li><li>[joshjohanning/approveops](https://github.com/joshjohanning/approveops) <code>caad905b2ba78301a0db7f484ef6fe3c770e6985</code></li></ul>
>
> ```md
> owner | repo | name | workflow | state | created_at | updated_at | last_run_at | uses
> --- | --- | --- | --- | --- | --- | --- | --- | ---
> joshjohanning-org | .github |  | [.github/workflows/update-organization-readme-badges.yml](https://github.com/joshjohanning-org/.github/blob/HEAD/.github/workflows/update-organization-readme-badges.yml) | active | 2024-05-23T16:58:49.000Z | 2024-05-23T16:58:49.000Z | 2025-08-17T07:07:40.000Z | <ul><li>[actions/checkout](https://github.com/actions/checkout) <code>v4</code></li><li>[actions/create-github-app-token](https://github.com/actions/create-github-app-token) <code>v2</code></li><li>[joshjohanning/organization-readme-badge-generator](https://github.com/joshjohanning/organization-readme-badge-generator) <code>v1</code></li></ul>
> joshjohanning-org | issueops-samples |  | [.github/workflows/delete-repos-delete.yml](https://github.com/joshjohanning-org/issueops-samples/blob/HEAD/.github/workflows/delete-repos-delete.yml) | active | 2023-11-08T16:05:40.000Z | 2025-04-02T15:48:27.000Z | 2025-08-13T14:57:36.000Z | <ul><li>[actions/checkout](https://github.com/actions/checkout) <code>v5</code></li><li>[issue-ops/parser](https://github.com/issue-ops/parser) <code>76d5aa095754de1493cbe41934484c4287e16350</code></li><li>[actions/create-github-app-token](https://github.com/actions/create-github-app-token) <code>v2</code></li><li>[actions/github-script](https://github.com/actions/github-script) <code>v7</code></li><li>[joshjohanning/approveops](https://github.com/joshjohanning/approveops) <code>caad905b2ba78301a0db7f484ef6fe3c770e6985</code></li></ul>
> ```
> {: file='actions-output.md'}

> *Full example output - `@stoe/action-reporting-cli`: [`json`](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/actions-output.json), [`md`](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/actions-output.md), [`csv`](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/actions-output.csv)*
{: .prompt-info }

**Sample Usage:**

```bash
# Organization-wide analysis with all data types
npx @stoe/action-reporting-cli 
  --owner my-org 
  --all 
  --exclude 
  --unique both 
  --csv ./reports/actions.csv 
  --json ./reports/actions.json 
  --md ./reports/actions.md
```

### Method 3: Custom SBOM Script (My Lightweight Solution)

The approach I've developed focuses on SBOM-style reporting with automated GitHub workflows. The [script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-actions-usage-in-organization.sh) is located in my [`github-misc-scripts` repository](https://github.com/joshjohanning/github-misc-scripts).

> See my [automated workflow implementation](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/.github/workflows/action-usage-sbom.yml) that runs this tool on a schedule and save the outputs back to the repository.
{: .prompt-tip }

**Key features:**

What makes this script useful:

- **Usage frequency counts**: Shows how many times each Action is used across the organization in an SBOM-like report
- **Version distribution**: Identifies which versions of Actions are most commonly used
- **SHA resolution**: Automatically resolves commit SHAs to readable tag versions when possible

**Sample Output - Count by Action:**

> | Count | Action |
> | --- | --- |
> | 121 | actions/checkout |
> | 28 | actions/upload-artifact |
> | 10 | github/codeql-action/upload-sarif |
> | 4 | joshjohanning/approveops |
> 
> ```markdown
> | Count | Action |
> | --- | --- |
> | 121 | actions/checkout |
> | 28 | actions/upload-artifact |
> | 10 | github/codeql-action/upload-sarif |
> | 4 | joshjohanning/approveops |
> ```
> {: file='count-by-action-sbom.md'}

> [*Full example output - SBOM Count by Action*](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/count-by-action-sbom.md)
{: .prompt-info }

**Sample Output - Count by Version:**

> | Count | Action |
> | --- | --- |
> | 57 | actions/checkout@v3 |
> | 54 | actions/checkout@v4 |
> | 11 | actions/upload-artifact@v4 |
> | 3 | github/codeql-action/upload-sarif@17573ee1cc1b9d061760f3a006fc4aac4f944fd5 # sha not associated to tag |
> | 2 | joshjohanning/approveops@caad905b2ba78301a0db7f484ef6fe3c770e6985 # v2.0.3 |
> 
> ```markdown
> | Count | Action |
> | --- | --- |
> | 57 | actions/checkout@v3 |
> | 54 | actions/checkout@v4 |
> | 11 | actions/upload-artifact@v4 |
> | 3 | github/codeql-action/upload-sarif@17573ee1cc1b9d061760f3a006fc4aac4f944fd5 # sha not associated to tag |
> | 2 | joshjohanning/approveops@caad905b2ba78301a0db7f484ef6fe3c770e6985 # v2.0.3 |
> ```
> {: file='count-by-version-sbom.md'}

> [*Full example output - SBOM Count by Version*](https://github.com/joshjohanning-org/export-actions-usage-report/blob/main/count-by-version-sbom.md)
{: .prompt-info }

**Sample Usage:**

```sh
# different options
./get-actions-usage-in-organization.sh joshjohanning-org count-by-version csv > output.csv
./get-actions-usage-in-organization.sh joshjohanning-org count-by-action md > output.md
./get-actions-usage-in-organization.sh joshjohanning-org count-by-version md --resolve-shas > output.md
./get-actions-usage-in-organization.sh joshjohanning-org count-by-action md --dedupe-by-repo > output.md
```

> **Need single repository analysis?** I also have a [repository-level version of this script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-actions-usage-in-repository.sh) that works the same way but analyzes just one repository instead of an entire organization.
{: .prompt-tip }

## Choosing the Right Method

- [**Use GitHub Dependency Insights**](#method-1-githubs-dependency-insights-view-only) first to get familiar with your organization's usage patterns
- [**Use @stoe/action-reporting-cli**](#method-2-stoeaction-reporting-cli-full-featured-solution) for comprehensive analysis with flexible export options, and especially if you want to [report on other things](https://github.com/stoe/action-reporting-cli?tab=readme-ov-file#report-content-options) like secrets, variables, permissions, listeners (workflow triggers), and/or runners (For implementing, see: [Using the Pre-Built Workflows](#using-the-pre-built-workflows) section)
- [**Use my custom SBOM script**](#method-3-custom-sbom-script-my-lightweight-solution) if you want usage statistics and the ability to resolve SHAs to tag versions (For implementing, see: [Using the Pre-Built Workflows](#using-the-pre-built-workflows) section)

## Using the Pre-Built Workflows

To implement these solutions in your organization:

1. **Fork or copy** the [export-actions-usage-report](https://github.com/joshjohanning-org/export-actions-usage-report) repository
   - If you fork it, make sure to enable Actions for the forked repository to allow the scheduled job to run
2. **Set up GitHub App authentication**:
   - Create a GitHub App with the following permissions:
     - **Repository permissions:** "Actions" (Read) - to read workflows and their usage (for [`@stoe/action-reporting-cli`](https://github.com/stoe/action-reporting-cli))
     - **Repository permissions:** "Contents" (Read) - to access SBOM data via dependency graph (for my [custom SBOM script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-actions-usage-in-organization.sh))
   - Install the app on your organization granting it access to all repositories
   - Add the App ID as a repository variable (`APP_ID`)  
   - Add the private key as a repository secret (`PRIVATE_KEY`)
   - You can use a personal access token, but a GitHub app has a higher rate limit
     - See [my post on GitHub Apps](/posts/github-apps/) for detailed instructions on creating and configuring a GitHub App
3. **Customize the workflows** if needed (different schedule, additional output formats, etc.)

The workflows will automatically:

- Run on a weekly schedule (or manually triggered)
- Generate usage reports using both [`@stoe/action-reporting-cli`](https://github.com/stoe/action-reporting-cli) and [my custom SBOM script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-actions-usage-in-organization.sh)
- Commits results back to the repository to be able to track changes over time
  - Additionally, pushes results as [workflow job summaries](/posts/github-code-coverage/#adding-code-coverage-to-job-summary)

## Summary

Having visibility into your organization's GitHub Actions usage is essential for security, managing dependencies, and making informed decisions about your CI/CD pipelines. While GitHub's native Dependency Insights provide a good starting point, automated export solutions offer the flexibility and depth needed for comprehensive and historical analysis.

Whether you're implementing an Actions allow list, conducting security audits, or just wanting better visibility into your CI/CD dependencies, these tools provide the foundation for data-driven decision making.

🚀 Ready to get started? Check out the [export-actions-usage-report repository](https://github.com/joshjohanning-org/export-actions-usage-report) and start building your Actions usage reporting today!
