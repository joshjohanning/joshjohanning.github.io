---
title: 'What Extra Features come with GitHub Advanced Security?'
author: Josh Johanning
date: 2021-12-03 16:30:00 -0600
description: A feature comparison between GitHub Enterprise, GitHub Enterprise with GitHub Advanced Security (GHAS), and Public Repos on github.com
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security]
image:
  src: /assets/screenshots/2021-12-03-github-advanced-security-feature-chart/organization-security-overview.png
  width: 1279   # in pixels
  height: 613   # in pixels
  alt: Security Overview for an Organization with GitHub Advanced Security
---

## Overview

[GitHub Advanced Security (GHAS)](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security) is an addon for those on GitHub Enterprise Cloud. While it costs extra, the [code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning), [secret scanning](https://docs.github.com/en/github/administering-a-repository/about-secret-scanning), and the [dependency review](https://docs.github.com/en/code-security/supply-chain-security/about-dependency-review) feature set is quite rich. Pretty much all of these features are enabled by default for Public Repos hosted on github.com (with the exception of the organization-level security overview and custom secret scanning patterns), so you can easily create a repo with some sample code from your personal GitHub account to test.

## GitHub Advanced Security Feature Comparison

I made this chart a while back for a client when helping them determine if the GHAS addon was worth it to them:

| Feature | GHEC | GHEC + GHAS | Public Repos |
|---------|:----:|:-----------:|:------------:|
| [Security Overview for the Org (Beta)](https://docs.github.com/en/code-security/security-overview/about-the-security-overview) | | X | n/a |
| [Dependency Graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph) | X | X | X |
| [Dependency Review in Pull Request](https://github.blog/changelog/2021-10-05-dependency-review-is-generally-available/) | | X | X |
| [CodeQL Code Scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-code-scanning) | | X | X |
| [GitHub Security Advisories](https://docs.github.com/en/code-security/security-advisories/about-github-security-advisories) | X | X | X |
| [Dependabot Alerts / Security Updates](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/about-dependabot-security-updates) | X | X | X |
| [Security Policies](https://docs.github.com/en/code-security/getting-started/adding-a-security-policy-to-your-repository) | X | X | X |
| [Secret Scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) | | X | X `*` |
| [Secret Scanning - Custom Patterns](https://docs.github.com/en/enterprise-server@3.2/code-security/secret-scanning/defining-custom-patterns-for-secret-scanning) | | X | |

Notes:
- GHEC = GitHub Enterprise Cloud
- GHAS = GitHub Advanced Security
- `*` - Note that you won't see a secret scanning menu for public repos, you will just get an email when a secret was committed to the repo and that the secret was (likely) automatically rolled or disabled
