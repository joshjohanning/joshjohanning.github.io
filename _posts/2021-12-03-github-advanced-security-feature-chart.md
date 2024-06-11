---
title: 'GitHub Advanced Security Feature Comparison'
author: Josh Johanning
date: 2021-12-03 16:30:00 -0600
description: A feature comparison between GitHub Enterprise, GitHub Enterprise with GitHub Advanced Security (GHAS), and Public Repos on github.com
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, Dependabot]
media_subpath: /assets/screenshots/2022-03-08-github-advanced-security-permissions-chart
image:
  path: ../2023-02-28-security-alerts/security-overview-light.png
  width: 100%
  height: 100%
  alt: Security Overview for an Organization
pin: false
---

## Overview

[GitHub Advanced Security (GHAS)](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security) is an addon for those on GitHub Enterprise. While it costs extra, the [code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning), [secret scanning](https://docs.github.com/en/github/administering-a-repository/about-secret-scanning), and the [dependency review](https://docs.github.com/en/code-security/supply-chain-security/about-dependency-review) feature set is quite impressive. Nearly all of these features are enabled by default for Public Repos hosted on github.com (with the exception of the [security overview](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview), [push protections for secrets](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning), and [custom patterns for secret scanning](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/defining-custom-patterns-for-secret-scanning)), so you can easily create a repo with some [sample code](https://github.com/joshjohanning/ghas-demo) from your personal GitHub account to play around with the features.

Follow updates in the [Changelog blog](https://github.blog/changelog/label/advanced-security/) for the latest updates on GitHub Advanced Security!

See also: [GitHub Advanced Security Permissions Chart](/posts/github-advanced-security-permissions-chart/)

## GitHub Advanced Security Feature Comparison

I made this chart a while back for a client when helping them determine if the GHAS addon was worth it to them:

| Feature | GHE | GHE + GHAS | Public Repos |
|---------|:----:|:-----------:|:------------:|
| [Dependency Graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph) | ✔️ | ✔️ | ✔️ |
| [Dependabot Alerts for Vulnerable Dependencies](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-alerts-for-vulnerable-dependencies) | ✔️ | ✔️ | ✔️ |
| [Dependabot Security Updates (PRs for vulnerabilities)](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-dependabot-security-updates) | ✔️ | ✔️ | ✔️ |
| [Dependabot Version Updates (PRs for package updates)](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates) | ✔️ | ✔️ | ✔️ |
| [Generate SBOM](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/exporting-a-software-bill-of-materials-for-your-repository) | ✔️ | ✔️ | ✔️ |
| [GitHub Security Advisories](https://docs.github.com/en/code-security/security-advisories/about-github-security-advisories) | ✔️ | ✔️ | ✔️ |
| [Security Policies](https://docs.github.com/en/code-security/getting-started/adding-a-security-policy-to-your-repository) | ✔️ | ✔️ | ✔️ |
| [Security Overview for the Org](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview) | ✔️ | ✔️ | ✔️ |
| [Security Overview for the Enterprise (Beta)](https://github.blog/changelog/2022-03-01-security-overview-for-enterprise-in-beta/) | ✔️ | ✔️ | ✔️ |
| [CodeQL Code Scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-code-scanning) | | ✔️ | ✔️ |
| [Dependency Review in Pull Request (rich diff)](https://github.blog/changelog/2021-10-05-dependency-review-is-generally-available/) | | ✔️ | ✔️ |
| [Dependency Review Action (Beta)](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement) | | ✔️ | ✔️ |
| [Secret Scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) | | ✔️ | ✔️ |
| [Secret Scanning - Custom Patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/defining-custom-patterns-for-secret-scanning) | | ✔️ | |
| [Secret Scanning - Push Protections (Beta)](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning) | | ✔️ | |

Notes:
- GHE = GitHub Enterprise
- GHAS = GitHub Advanced Security
- This chart primarily focuses on **GitHub Enterprise Cloud**, but note that Advanced Security is available for GitHub Enterprise Server [3.0 or higher](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security). There may be slight differences in the features available for GitHub Enterprise Server based on the version

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

**Dependabot version updates** requires creating a [`dependabot.yml`{: .filepath}](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates#configuration-options-for-private-registries) configuration file in your repository whereas **Dependabot security updates** automatically locates [supported package manifest files](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems) and alerts you when it contains vulnerable dependencies.

[Dependabot version updates](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/about-dependabot-version-updates#supported-repositories-and-ecosystems) supported package ecosystems differs from that of [Dependabot security updates](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems).

## Changelog

| Date        | Note |
|-------------|------|
| Apr 26 2023 | Removing subscript note on [secret scanning for public repos](https://github.blog/2023-02-28-secret-scanning-alerts-are-now-available-and-free-for-all-public-repositories/), added SBOM generation
| Oct 11 2022 | Removing Beta from [Security Overview for the Org](https://github.blog/changelog/2022-04-07-security-overview-for-organizations-is-generally-available/),<br>[Security Overview is available to all GitHub Enterprise customers](https://github.blog/changelog/2022-08-08-security-overview-is-now-available-to-all-github-enterprise-users/)<br> |
| Apr 06 2022 | Adding [Dependency Review Action (Beta)](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement) |
| Apr 04 2022 | Adding [Secret Scanning - Push Protections (Beta)](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning) |
| Mar 07 2022 | Adding new Security [Overview for the Enterprise (Beta)](https://github.blog/changelog/2022-03-01-security-overview-for-enterprise-in-beta/) and [secret scanning note for public repos](https://github.blog/changelog/2022-03-04-secret-scanning-advanced-security-customers-can-now-view-alerts-on-their-public-repositories/) |
| Jan 26 2022 | Adding [Dependabot section](#about-dependabot), reorganized chart |
| Dec 03 2021 | Initial post |
