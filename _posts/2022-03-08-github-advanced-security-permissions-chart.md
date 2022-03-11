---
title: 'GitHub Advanced Security Permissions Chart'
author: Josh Johanning
date: 2022-03-08 12:00:00 -0600
description: An access requirements/permissions comparison between various roles in GitHub Enterprise and GitHub Advanced Security, such as what users with Write access to the repository get vs. what requires elevated privileges
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security]
img_path: /assets/screenshots/2022-03-08-github-advanced-security-permissions-chart
image:
  src: ../2021-12-03-github-advanced-security-feature-chart/organization-security-overview.png
  width: 100%
  height: 100%
  alt: Security Overview for an Organization with GitHub Advanced Security
---

## Overview

I have several [posts discussing GitHub Advanced Security](/tags/github-advanced-security/), but practically a question that I get often is: _"Who can access the alerts on each repository?"_

I hope to solve that with this permissions / access requirements chart!

See also: [GitHub Advanced Security Feature Comparison](/posts/github-advanced-security-feature-chart/)

## Access requirements for security features

This chart is loosely based on [the one from GitHub](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-access-to-your-organizations-repositories/repository-roles-for-an-organization), with a few additions, modifications, and clarifications. 

| Feature                                                                                                                                                                                                                                                         | Read, Triage | Write           | Maintain        | Admin | Security Mgr    | Org Owner        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------:|:---------------:|:---------------:|:-----:|:---------------:|:----------------:|
| [Receive Dependabot alerts](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/about-alerts-for-vulnerable-dependencies)                                                                                                    |              |                 |                 |   X   |       X         |     X            |
| [Dismiss Dependabot alerts](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/viewing-and-updating-vulnerable-dependencies-in-your-repository)                                                                             |              |                 |                 |   X   |       X         |     X            |
| [Designate others to receive security alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts)                              |              |                 |                 |   X   |       X         |     X            |
| [Create security advisories](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-advisories/about-github-security-advisories)                                                                                                             |              |                 |                 |   X   |       X         |     X            |
| [Manage access to GHAS features in the repo](https://docs.github.com/en/enterprise-cloud@latest/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-security-and-analysis-settings-for-your-repository) |              |                 |                 |   X   |       X         |     X            |
| [Enable the dependency graph](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/exploring-the-dependencies-of-a-repository)                                                       |              |                 |                 |   X   |       X         |     X            |
| [View dependency reviews](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/about-dependency-review)                                                                                                                       |      X       |   X             |     X           |   X   |       X         |     X            |
| [View code scanning alerts on pull requests](https://docs.github.com/en/enterprise-cloud@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/triaging-code-scanning-alerts-in-pull-requests)                                                 |      X       |   X             |     X           |   X   |       X         |     X            |
| [Manage code scanning alerts](https://docs.github.com/en/enterprise-cloud@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/managing-code-scanning-alerts-for-your-repository)                                                             |              |   X             |     X           |   X   |       X         |     X            |
| [View secret scanning alerts in a repository](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-alerts-from-secret-scanning)                                                                                        |              | X<sup>[1]</sup> | X<sup>[1]</sup> |   X   |       X         |     X            |
| [Manage secret scanning alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-alerts-from-secret-scanning)                                                                                                      |              | X<sup>[1]</sup> | X<sup>[1]</sup> |   X   |       X         |     X            |
| [Access to the org's security overview](https://docs.github.com/en/code-security/security-overview/about-the-security-overview)                                                                                                                                 |              |                 |                 |       |       X         |     X            |
| [Access to the enterprise's security overview](https://docs.github.com/en/code-security/security-overview/about-the-security-overview)                                                                                                                          |              |                 |                 |       | X<sup>[2]</sup> | X<sup>[2]</sup>  |
| [Manage GHAS features at org level](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization)           |              |                 |                 |       |       X         |     X            |
| [Designate Security Managers](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/managing-security-managers-in-your-organization)                                                         |              |                 |                 |       |                 |     X            |
| Read access to repo(s)                                                                                                                                                                                                                                          |      X       |   X             |     X           |   X   | X<sup>[3]</sup> |     X            |
| Write access to repo(s)                                                                                                                                                                                                                                         |              |   X             |     X           |   X   |                 |     X            |

Notes:

- [1] Repository writers and maintainers can only see alert information for their own commits
- [2] At the enterprise security overview level, you would only see organizations that you are added as an org owner or security manager
- [3] Security managers get read-only access to every repository in the organization
- This chart primarily focuses on GitHub Advanced Security in GitHub Enterprise Cloud

## Granting access to security alerts

Security alerts for a repository are visible to people with admin access to the repository and, when the repository is owned by an organization, organization owners. You can also [give additional teams and people access to the alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts).

When [adding users to be able to view security alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts), there is a bit of text that explains (emphasis mine): 

> Admins, users, and teams in the list below have permission to **view and manage code scanning, Dependabot, or secret scanning alerts**. These users may be notified when a new vulnerability is found in one of this repository's dependencies and when a secret or key is checked in. They will also see additional details when viewing Dependabot security updates. Individuals can manage how they receive these alerts in their [notification settings](https://github.com/settings/notifications).

Note: Organization owners and repository administrators can only grant access to view security alerts, such as secret scanning alerts, to people or teams who have write access to the repo.

## Custom Repository Roles

Organization administrators can create [Custom Repository Roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/managing-custom-repository-roles-for-an-organization) to customize and fine-tune different permission sets that repository administrators can grant. For example, I want to create a role that allows users to have Write access AND be able to view/dismiss Dependabot Alerts:

![Custom Repository Roles](custom-roles.png){: .shadow }

## Changelog

| Date        | Note                                 |
|-------------|--------------------------------------|
| Mar 11 2021 | Adding section about security alerts |
| Mar 08 2021 | Initial post                         |

