---
title: 'Tips for Handling Dependabot, CodeQL, and Secret Scanning Alerts'
author: Josh Johanning
date: 2023-02-28 15:30:00 -0600
description: My musings on handling security alerts in GitHub
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Advanced Security, Dependabot, CodeQL, Secret Scanning]
media_subpath: /assets/screenshots/2023-02-28-security-alerts
image:
  path: security-overview-light.png
  width: 100%
  height: 100%
  alt: Security Overview for an Organization
---

## Overview

I recently had the opportunity to work with a large organization to help them manage their security alerts. Often, enabling security tooling is easy, it's what comes next (how do we handle alerts, who fixes them, when are they fixed, etc.). For posterity, this post is a summary of my thoughts on how to handle security alerts in GitHub.

## My Notes

- It might seem obvious, but organizations should **focus on the critical and high alerts first**
  - Teams can get overwhelmed by the sheer number of alerts that Dependabot finds, but should instead be focusing on the **critical/high alert count**
- Leverage **Pull requests gates**
  - For **Dependencies**: Introduce **gates during pull requests** to at least not allow people to introduce *new* vulnerable dependencies (ie: use the [Dependency Review Action](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement))
  - I would **advise against blocking all PRs from being merged if ANY** high/critical Dependabot alerts are present. This would affect PRs that didn't touch dependencies as well
  - Certainly, in some high-regulated environments, this could have the desired effect of **forcing** teams to fix their security issues, but in nearly all cases, **I would advise against this**
  - While you can't set up a **[Required Workflow](https://docs.github.com/en/enterprise-cloud@latest/actions/using-workflows/required-workflows) for CodeQL**, you can use this for the **[Dependency Review action](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement)/status check for Dependabot** ([sample here](https://gist.github.com/joshjohanning/0d3c49431ee8e7e3a30a306f6017604a)) for enforcing this across an organization
  - For **Code Scanning**: Same idea, introduce gates to **prevent people from merging new code that contains a potential vulnerability**. 
  - This can be done by adding the **CodeQL status check to the [branch protection rule](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/triaging-code-scanning-alerts-in-pull-requests#code-scanning-results-check)**. For best coverage, add the `CodeQL` status check as well as the `Analyze` status check(s) (ie: `Analyze (java)`)
- Some organizations introduce an **SLA** for Dependabot Alerts that are found
  - Example: If a **critical Dependabot alert is found, you have 10 days** to fix it. High = 30, Medium = 60, Low = 90 (or similar)
  - Typically, a **grace period** is given when turning on security settings to allow teams to burn down the backlog of alerts
  - One could potentially use **[webhook event](https://docs.github.com/webhooks-and-events/webhooks/webhook-events-and-payloads#dependabot_alert) to trigger when alerts** are found and post them to a Slack or Teams channel or even as Issues on their repository!
  - Ideally, **you wouldn't need to rely on the SLA for Code Scanning results** after initial onboarding since vulnerabilities should be caught and fixed during pull requests with **gates** 😄. There are **more likely to be Dependabot Alerts that are found out of band than CodeQL alerts**
- Some organizations introduce an **“alert cap”** similar to a **“bug cap”** - if we get more than 10 Dependabot alerts for example, we have to burn them down
- Ultimately, organizations **need to have procedural practices in place** (culture) to make security a concern so that people don’t “ignore” alerts and instead work to fix them
  - If you're asking "**How can I snooze a Dependabot alert**", your team's approach is **wrong**. It's okay to get alerts, new vulnerabilities are discovered in packages every day, but you should be working to fix them
- Great **testing** is effective at helping you resolve Dependabot alerts
  - If a Dependabot Security Alert PR is created, if your **build job and unit tests pass**, then you can be reasonably confident that the **PR is safe to merge**
  - There isn't a great way to **distinguish between a Dependabot Security Update PR vs. a [Dependabot Version Update](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates) PR** (non-security related), but you can use something like **labels** defined in your `dependabot.yml`{: .filepath} file to help you distinguish between the two
- Some teams ask if they can ignore **development-scoped dependencies (such as devDependencies)**
  - This was added as a feature to [**Dependabot in June 2022**](https://github.blog/changelog/2022-06-23-dependabot-alerts-filter-alerts-by-the-scope-of-the-dependency-runtime-and-development/)
  - While it is true your end users won't be affected by a vulnerability in a development dependency, your developers very well may be, and your developers certainly have access to privileged information that make them a **prime attack vector**
- Other **Dependabot notes**
  - Upon merging the PR with the package version fix, the **Dependabot Alert will automatically be closed** (**same with Code Scanning alerts**, once the code is fixed, the alert is automatically closed)
  - Not all Dependabot Alerts get created as Security Update pull requests **([there’s a limit of 10](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/troubleshooting-dependabot-errors#dependabot-cannot-open-any-more-pull-requests))**, but can be slightly less than that due to various factors
  - Sometimes dependencies have a **vulnerability disclosed with no new version to update to** - in those situations you have to evaluate the risk and decide if you want to continue using that dependency or not
  - Some ecosystems, like Python's `pip`, can show you if you are **[referencing a vulnerable code path in a dependency](https://github.blog/2022-04-14-dependabot-alerts-now-surface-if-code-is-calling-vulnerability/)**
- Using **[tspascoal/dependabot-alerts-helper](https://github.com/tspascoal/dependabot-alerts-helper) scripts** to export Dependabot alerts to a CSV file
  - This can be used for further **analysis / grouping / management** of alerts, especially if you are responsible for a lot of repositories
  - The **[Security Overview](https://docs.github.com/en/enterprise-cloud@latest/code-security/security-overview/about-the-security-overview)** is great, but if you only care about specific repositories but have access to a lot of repositories, it can be a little noisy 
  - This tool also supports **merging Dependabot Security Update PRs in bulk**. The idea, in theory, is that if you used the same old version of a package throughout your organization, and are able to verify it’s a non-breaking change, then you could mass merge those open Dependabot pull requests to resolve those alerts
  - This analysis and testing could be useful **if a single package was causing a high percentage of alerts** across a team/organization
  - Maybe you can’t do this throughout the organization, but a single team could use some of the tools to at least make changes in **their x number of repos that they own**
  - Also, who doesn't like pulling up Excel for some pretty tables/charts? 📊
- Additional **useful apps**
  - **[advanced-security/probot-security-alerts](https://github.com/advanced-security/probot-security-alerts)** - A Probot app to ensure that people are not closing security alerts without actually fixing them / require them to have the proper permissions to do so
  - **[advanced-security/ghas-reviewer-app](https://github.com/advanced-security/ghas-reviewer-app)** - Similar to the above
  - **[github/safe-settings](https://github.com/github/safe-settings)** - This can be useful to ensure repositories have a pull request review requirement as well as requiring the [Dependency Review](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review#dependency-review-enforcement) status check
  - **[NickLiffen/ghas-enablement](https://github.com/NickLiffen/ghas-enablement)** - Useful to push CodeQL workflows as well as enabling Security features to a set of repositories
  - **[advanced-security/generate-sbom-action](https://github.com/advanced-security/generate-sbom-action)** - A GitHub Action to generate a Software Bill of Materials (SBOM) for your repository
  - **[KittyChiu/probot-secret-remediation](https://github.com/KittyChiu/probot-secret-remediation/)** - A Probot app to automatically create issues when a secret scanning push protection is bypassed
  - **[github/ghas-jira-integration](https://github.com/github/ghas-jira-integration)** - A GitHub Action to create Jira issues from GitHub Advanced Security alerts
  - **[advanced-security/policy-as-code](https://github.com/advanced-security/policy-as-code)** - A GitHub Action to enforce policies on your repository based on risk threshold
- Integrate GHAS with other **[Security Information and Events Management (SIEM) tools](https://github.blog/2022-10-13-introducing-github-advanced-security-siem-integrations-for-security-professionals)**
  - Such as **[Splunk's dashboard](https://github.com/splunk/github_app_for_splunk#integration-overview-dashboard)**
  - Or **[Microsoft Sentinel](https://github.blog/2022-10-13-introducing-github-advanced-security-siem-integrations-for-security-professionals/#microsoft-sentinel)**
- What to do when you find **Secret Scanning** results
  - Turn on **[push protections](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/protecting-pushes-with-secret-scanning)**! This won't block all secrets, but it will block secret types with a high confidence score to minimize disruptive false positives
  - It is easier/more secure to **rotate secrets** than to **clean the repo history** with something like [BFG to remove the commit](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
  - You still might want to clean the repo history for reasons, but regardless, **you should still rotate the secret**! And note in the README when the history was re-written for audit purposes.
  - For additional coverage, create **[custom secret scanning patterns](https://docs.github.com/en/enterprise-cloud@latest/code-security/secret-scanning/defining-custom-patterns-for-secret-scanning)** based on your use cases. These can also be created with push protections enabled. Here's a [repo](https://github.com/advanced-security/secret-scanning-custom-patterns) for some predefined patterns!

## Further Reading

- Check out some of [@colindembovsky](https://github.com/colindembovsky/)'s posts, such as [Shift Left - How far is too far?](https://colinsalmcorner.com/shift-left-how-far-is-too-far/), [Fine Tuning CodeQL Scans using Query Filters](https://colinsalmcorner.com/fine-tuning-codeql-scans/), [GHAS Will Win the AppSec Wars](https://colinsalmcorner.com/ghas-will-win-the-appsec-wars/), and [Mission Control - and what it means for DevSecOps](https://colinsalmcorner.com/mission-control/)
- Check out [@kenmuse](https://github.com/kenmuse)'s [Security Theater](https://www.kenmuse.com/blog/security-theater/) post - just because a different vendor says they are cover every compliance rule, doesn't mean they really do
- Check out [@nickliffen](https://github.com/nickliffen)'s [Why Advanced Security?](https://nickliffen.dev/articles/why-advanced-security.html) post - "there is more to a security tool than the number of results found!"
- Check out this post from the GitHub Blog, [5 tips for prioritizing Dependabot alerts](https://github.blog/2022-09-19-5-tips-for-prioritizing-dependabot-alerts/), for additional ideas with Dependabot alerts
- Check out this page from the GitHub docs, [Adopting GitHub Advanced Security at scale](https://docs.github.com/en/enterprise-cloud@latest/code-security/adopting-github-advanced-security-at-scale), for ideas on using a phased approach to rollout GitHub Advanced Security across your organization
