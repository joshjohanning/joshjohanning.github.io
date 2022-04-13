---
title: 'GitHub Advanced Security Feature Comparison'
author: Josh Johanning
date: 2021-12-03 16:30:00 -0600
description: A feature comparison between GitHub Enterprise, GitHub Enterprise with GitHub Advanced Security (GHAS), and Public Repos on github.com
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, Dependabot]
image:
  src: /assets/screenshots/2021-12-03-github-advanced-security-feature-chart/organization-security-overview.png
  width: 100%
  height: 100%
  alt: Security Overview for an Organization with GitHub Advanced Security
pin: true
---

## Overview

[GitHub Advanced Security (GHAS)](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security) is an addon for those on GitHub Enterprise. While it costs extra, the [code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning), [secret scanning](https://docs.github.com/en/github/administering-a-repository/about-secret-scanning), and the [dependency review](https://docs.github.com/en/code-security/supply-chain-security/about-dependency-review) feature set is quite impressive. Pretty much all of these features are enabled by default for Public Repos hosted on github.com (with the exception of the organization-level security overview and custom secret scanning patterns), so you can easily create a repo with some sample code from your personal GitHub account to test.

Follow updates in the [Changelog blog](https://github.blog/changelog/label/advanced-security/) for the latest updates on GitHub Advanced Security!

## GitHub Advanced Security Feature Comparison

I made this chart a while back for a client when helping them determine if the GHAS addon was worth it to them:

| Feature | GHE | GHE + GHAS | Public Repos |
|---------|:----:|:-----------:|:------------:|
| [Dependency Graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph) | X | X | X |
| [Dependabot Alerts for Vulnerable Dependencies](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-alerts-for-vulnerable-dependencies) | X | X | X |
| [Dependabot Security Updates (PRs for vulnerabilities)](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-dependabot-security-updates) | X | X | X |
| [Dependabot Version Updates (PRs for package updates)](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates) | X | X | X |
| [GitHub Security Advisories](https://docs.github.com/en/code-security/security-advisories/about-github-security-advisories) | X | X | X |
| [Security Policies](https://docs.github.com/en/code-security/getting-started/adding-a-security-policy-to-your-repository) | X | X | X |
| [Security Overview for the Org (Beta)](https://docs.github.com/en/code-security/security-overview/about-the-security-overview) | | X | n/a |
| [Security Overview for the Enterprise (Beta)](https://github.blog/changelog/2022-03-01-security-overview-for-enterprise-in-beta/) | | X | n/a |
| [CodeQL Code Scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-code-scanning) | | X | X |
| [Dependency Review in Pull Request (rich diff)](https://github.blog/changelog/2021-10-05-dependency-review-is-generally-available/) | | X | X |
| [Dependency Review Action (Beta)](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement) | | X | X |
| [Secret Scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) | | X | X `*` |
| [Secret Scanning - Custom Patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/defining-custom-patterns-for-secret-scanning) | | X | |
| [Secret Scanning - Push Protections (Beta)](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning) | | X | |

Notes:
- GHE = GitHub Enterprise
- GHAS = GitHub Advanced Security
- `*` - Note that you won't see a secret scanning menu for public repos, you will just get an email when a secret was committed to the repo and that the secret was (likely) automatically rolled or disabled
    + If you subscribe to GitHub Advanced Security and have a public repo, [you can still see the alerts](https://github.blog/changelog/2022-03-04-secret-scanning-advanced-security-customers-can-now-view-alerts-on-their-public-repositories/)
- This chart primarily focuses on GitHub Enterprise Cloud, but note that Advanced Security is available for GitHub Enterprise Server [3.0 or higher](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)

## About Dependabot

There are a few components of Dependabot, and while I tried to list each feature individually in the chart, I wanted to call out a [helpful quote of the documentation](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates) to help describe part of the differences between [version updates](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates) and [security updates](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-dependabot-security-updates): 

> About Dependabot version updates: 
> 
> When Dependabot identifies an outdated dependency, it raises a pull request to update the manifest to the latest version of the dependency. For vendored dependencies, Dependabot raises a pull request to replace the outdated dependency with the new version directly. You check that your tests pass, review the changelog and release notes included in the pull request summary, and then merge it. For more information, see "[Enabling and disabling Dependabot version updates](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/enabling-and-disabling-dependabot-version-updates)."
> 
> If you enable security updates, Dependabot also raises pull requests to update vulnerable dependencies. For more information, see "[About Dependabot security updates](https://docs.github.com/en/github/managing-security-vulnerabilities/about-dependabot-security-updates)."
> 
> When Dependabot raises pull requests, these pull requests could be for security or version updates:
> - _[Dependabot security updates](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-dependabot-security-updates)_ are automated pull requests that help you update dependencies with known vulnerabilities.
> - _[Dependabot version updates](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates)_ are automated pull requests that keep your dependencies updated, even when they don’t have any vulnerabilities. To check the status of version updates, navigate to the Insights tab of your repository, then Dependency Graph, and Dependabot.

**Dependabot version updates** requires creating a [dependabot.yml](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates#configuration-options-for-private-registries) configuration file in your repository whereas **Dependabot security updates** automatically locates [supported package manifest files](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems) and alerts you when it contains vulnerable dependencies.

[Dependabot version updates](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates#supported-repositories-and-ecosystems) supported package ecosystems differs from that of [Dependabot security updates](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems).

## Changelog

| Date        | Note |
|-------------|------|
| Apr 06 2022 | Adding [Dependency Review Action (Beta)](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement) |
| Apr 04 2022 | Adding [Secret Scanning - Push Protections (Beta)](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning) |
| Mar 07 2022 | Adding new Security [Overview for the Enterprise (Beta)](https://github.blog/changelog/2022-03-01-security-overview-for-enterprise-in-beta/) and [secret scanning note for public repos](https://github.blog/changelog/2022-03-04-secret-scanning-advanced-security-customers-can-now-view-alerts-on-their-public-repositories/) |
| Jan 26 2022 | Adding [Dependabot section](#about-dependabot); reorganized chart |
| Dec 03 2021 | Initial post |
