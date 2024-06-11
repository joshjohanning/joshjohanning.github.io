---
title: 'GitHub Advanced Security Permissions Chart'
author: Josh Johanning
date: 2022-03-08 12:00:00 -0600
description: An access requirements/permissions comparison between various roles in GitHub Enterprise and GitHub Advanced Security, such as what users with Write access to the repository get vs. what requires elevated privileges
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, Dependabot]
media_subpath: /assets/screenshots/2022-03-08-github-advanced-security-permissions-chart
image:
  path: ../2023-02-28-security-alerts/security-overview-dark.png
  width: 100%
  height: 100%
  alt: Security Overview for an Organization
pin: false
---

## Overview

I have several [posts discussing GitHub Advanced Security](/tags/github-advanced-security/), but practically a question that I get often is: _"Who can access the alerts on each repository?"_

I hope to solve that with this permissions / access requirements chart!

See also: [GitHub Advanced Security Feature Comparison](/posts/github-advanced-security-feature-chart/)

## Access requirements for security features

This chart is loosely based on [the one from GitHub](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-access-to-your-organizations-repositories/repository-roles-for-an-organization), with a few additions, modifications, and clarifications. 

| Feature                                                                                                                                                                                                                                                  | Read<sup>[1]</sup> | Write,Mntn <sup>[2]</sup> | Admin           | Sec. Mgr    | Org Owner        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------:|:---------------:|:---------------:|:---------------:|:----------------:|
| [Receive Dependabot alerts](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/about-alerts-for-vulnerable-dependencies)                                                                                                    |              | ✔️               | ✔️               |       ✔️         |     ✔️            |
| [Dismiss Dependabot alerts](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/viewing-and-updating-vulnerable-dependencies-in-your-repository)                                                                             |              | ✔️               | ✔️               |       ✔️         |     ✔️            |
| [Designate others to receive security alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts)                              |              |                 | ✔️               |       ✔️         |     ✔️            |
| [Create security advisories](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-advisories/about-github-security-advisories)                                                                                                             |              |                 | ✔️               |       ✔️         |     ✔️            |
| [Manage access to GHAS features in the repo](https://docs.github.com/en/enterprise-cloud@latest/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-security-and-analysis-settings-for-your-repository) |              |                 | ✔️               |       ✔️         |     ✔️            |
| [Enable the dependency graph](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/understanding-your-software-supply-chain/exploring-the-dependencies-of-a-repository)                                                       |              |                 | ✔️               |       ✔️         |     ✔️            |
| [View dependency reviews](https://docs.github.com/en/enterprise-cloud@latest/code-security/supply-chain-security/about-dependency-review)                                                                                                                       |      ✔️       |   ✔️             | ✔️               |       ✔️         |     ✔️            |
| [View code scanning alerts on pull requests](https://docs.github.com/en/enterprise-cloud@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/triaging-code-scanning-alerts-in-pull-requests)                                                 |      ✔️       |   ✔️             | ✔️               |       ✔️         |     ✔️            |
| [Manage code scanning alerts](https://docs.github.com/en/enterprise-cloud@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/managing-code-scanning-alerts-for-your-repository)                                                             |              |   ✔️             | ✔️               |       ✔️         |     ✔️            |
| [View secret scanning alerts in a repository](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-alerts-from-secret-scanning)                                                                                        |              | ⚠️<sup>[3]</sup> | ✔️               |       ✔️         |     ✔️            |
| [Manage secret scanning alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-alerts-from-secret-scanning)                                                                                                      |              | ⚠️<sup>[3]</sup> | ✔️               |       ✔️         |     ✔️            |
| [Access to the org's security overview](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview)                                                                                                         |              | ✔️<sup>[4]</sup> | ✔️<sup>[4]</sup> |       ✔️         |     ✔️            |
| [Access to the enterprise's security overview](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview)                                                                                                  |              |                 |                 | ✔️<sup>[5]</sup> | ✔️<sup>[5]</sup>  |
| [Manage GHAS features at org level](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-security-and-analysis-settings-for-your-organization)           |              |                 |                 |       ✔️         |     ✔️            |
| [Designate Security Managers](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/managing-security-managers-in-your-organization)                                                         |              |                 |                 |                 |     ✔️            |
| Read access to repo(s)                                                                                                                                                                                                                                          |      ✔️       |   ✔️             | ✔️               | ✔️<sup>[6]</sup> |     ✔️            |
| Write access to repo(s)                                                                                                                                                                                                                                         |              |   ✔️             | ✔️               |                 |     ✔️            |

Notes:

- [1] **Read** and **Triage** have the same rights for security features
- [2] **Write** and **Maintain** have the same rights for security features
- [3] Repository **writers** and **maintainers** can only see secret alert information for their own commits, but **only as a direct link to the secret scanning alert sent via email** ⚠️
- [4] Now that the [org-level security overview is available to all Enterprise users](https://github.blog/changelog/2022-08-08-security-overview-is-now-available-to-all-github-enterprise-users/), org members can see consolidated results of repositories that they can see alerts for (e.g., **write** for CodeQL and Dependabot, **admin** for secrets)
- [5] In the [enterprise-level security overview](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview) level, one would see organizations where they are added as an **org owner** or **security manager** - **enterprise owners** must [join an organization as an owner](https://docs.github.com/en/enterprise-cloud@latest/admin/user-management/managing-organizations-in-your-enterprise/managing-your-role-in-an-organization-owned-by-your-enterprise) to see alerts
- [6] **Security managers** get read-only access to every repository in the organization
- This chart primarily focuses on **GitHub Enterprise Cloud**, but note that Advanced Security is available for GitHub Enterprise Server [3.0 or higher](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security). There may be slight differences in the features available for GitHub Enterprise Server based on the version

## Granting access to security alerts

Security alerts for a repository are visible to people with admin access to the repository and, when the repository is owned by an organization, organization owners. You can also [give additional teams and people access to the alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts).

When [adding users to be able to view security alerts](https://docs.github.com/en/enterprise-cloud@latest/github/administering-a-repository/managing-security-and-analysis-settings-for-your-repository#granting-access-to-security-alerts), there is a bit of text that explains (emphasis mine): 

> Admins, users, and teams in the list below have permission to **view and manage code scanning, Dependabot, or secret scanning alerts**. These users may be notified when a new vulnerability is found in one of this repository's dependencies and when a secret or key is checked in. They will also see additional details when viewing Dependabot security updates. Individuals can manage how they receive these alerts in their [notification settings](https://github.com/settings/notifications).

Note: Organization owners and repository administrators can only grant access to view security alerts, such as secret scanning alerts, to people or teams who have **write** access to the repo.

## Custom Repository Roles

Organization administrators can create [Custom Repository Roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/managing-custom-repository-roles-for-an-organization) to customize and fine-tune different permission sets that repository administrators can grant. For example, I want to create a role that allows users to have Write access AND be able to view/dismiss Dependabot Alerts:

![Custom Repository Roles](custom-roles.png){: .shadow }
_Custom repository roles - creating a custom role to allow viewing/managing of Dependabot alerts_

Note that there is currently a maximum of 3 custom roles that can be created in the organization.

## Changelog

| Date        | Note                                 |
|-------------|--------------------------------------|
| Apr 26 2023 | Write and Maintain can [view/manage Dependabot alerts now](https://github.blog/changelog/2023-02-07-dependabot-alerts-default-permissions-change/) (GHES 3.9+)                         |
| Oct 11 2022 | Removing Beta from [Security Overview for the Org](https://github.blog/changelog/2022-04-07-security-overview-for-organizations-is-generally-available/),<br>[Security Overview is available to all GitHub Enterprise customers](https://github.blog/changelog/2022-08-08-security-overview-is-now-available-to-all-github-enterprise-users/),<br>Consolidated Read/Triage and Write/Maintain since they have the same security permissions |
| Mar 11 2021 | Adding section about security alerts |
| Mar 08 2021 | Initial post                         |
