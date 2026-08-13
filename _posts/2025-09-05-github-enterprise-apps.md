---
title: 'Using GitHub Enterprise Apps for Programmatic App Management Across Orgs'
author: Josh Johanning
date: 2025-09-05 14:00:00 -0500
description: A comprehensive guide to using Enterprise GitHub Apps to programmatically install and manage applications across all organizations in your GitHub Enterprise
categories: [GitHub, Apps]
tags: [GitHub, GitHub Apps, GitHub Enterprise Management]
media_subpath: /assets/screenshots/2025-09-05-github-enterprise-apps
image:
  path: enterprise-github-apps-light.png
  width: 100%
  height: 100%
  alt: An Enterprise-owned GitHub App with the capability to install and manage apps across all organizations in the enterprise
---

## Overview

GitHub has made enterprise GitHub App management much easier with the [general availability of Enterprise GitHub Apps](https://github.blog/changelog/2025-03-10-enterprise-owned-github-apps-are-now-generally-available/) and [Enterprise-level access for GitHub Apps and installation automation APIs (public preview)](https://github.blog/changelog/2025-07-01-enterprise-level-access-for-github-apps-and-installation-automation-apis/). These capabilities allow enterprise administrators to programmatically install, manage, and audit GitHub Apps across hundreds of organizations without manually clicking through installation screens. This is particularly useful during migration scenarios where you need to programmatically install and configure apps across multiple organizations.

As of May 2026, there's also a new [enterprise installation API (public preview)](https://github.blog/changelog/2026-05-13-new-enterprise-installation-api-now-in-public-preview/) that lets an App look up its own enterprise installation ID via [`GET /enterprises/{enterprise}/installation`](https://docs.github.com/en/enterprise-cloud@latest/rest/apps/apps#get-an-enterprise-installation-for-the-authenticated-app) instead of grabbing it from the browser.

This post shows how to use these new APIs with practical bash examples.

> Check out my [Demystifying GitHub Apps: Using GitHub Apps to Replace Service Accounts](/posts/github-apps/) post if you're interested in learning more about what a GitHub App is! 🚀
{: .prompt-info }

## What Are Enterprise GitHub Apps?

Enterprise GitHub Apps are GitHub Apps owned by an enterprise account instead of an individual user or organization. Enterprise ownership gives enterprise owners a central place to create, manage, and transfer Apps used across their organizations.

The [March 2025 GA release](https://github.blog/changelog/2025-03-10-enterprise-owned-github-apps-are-now-generally-available/) made this ownership model generally available. Existing organization-owned Apps can be transferred to the enterprise, and permission updates are automatically accepted by organizations where the App is installed. This removes much of the organization-by-organization administration required for Apps used across a large enterprise.

Enterprise ownership alone does not grant an App access to enterprise resources. The App must also be installed on the enterprise and granted the appropriate enterprise permissions.

### Enterprise App Permission Scopes & Capabilities

The available permissions now extend well beyond App installation management. The following tree groups them by the enterprise resources they control:

```text
Enterprise permissions
├── Copilot and AI
│   ├── Copilot usage records (view API usage records)
│   ├── Enterprise Copilot metrics (view Copilot metrics)
│   └── Enterprise AI controls (manage enterprise-wide AI controls)
├── Roles and access
│   ├── Custom enterprise roles (manage roles and assignments)
│   ├── Enterprise custom organization roles (manage organization roles)
│   ├── Enterprise people (manage user access)
│   ├── Enterprise teams (manage enterprise teams)
│   └── Enterprise single sign-on (view and manage SSO configuration)
├── Properties and security
│   ├── Custom properties (manage repository property definitions)
│   ├── Enterprise custom properties for organizations
│   ├── Enterprise credentials (view and manage credentials)
│   └── Enterprise innersource vulnerabilities
├── Enterprise administration
│   └── Enterprise organizations (create and remove organizations)
└── GitHub App installation management
    ├── Enterprise organization installations (install or uninstall Apps)
    └── Enterprise organization installation repositories
        ├── Change access between all and selected repositories
        └── Add or remove repositories from an App installation
```
{: .nolineno}

GitHub maintains the endpoint-level details for each scope in the [enterprise permissions reference](https://docs.github.com/enterprise-cloud@latest/rest/overview/permissions-required-for-github-apps#enterprise-permissions).

> Requesting either **Enterprise organization installation repositories** or **Enterprise organization installations** prevents the App from being installed on other enterprises.
{: .prompt-warning }

## Installation Automation API Examples

The examples in this post use the **Enterprise organization installations** and **Enterprise organization installation repositories** permissions with the [enterprise-level access and installation automation APIs](https://github.blog/changelog/2025-07-01-enterprise-level-access-for-github-apps-and-installation-automation-apis/). Together, they allow an App to:

- Install or uninstall GitHub Apps across enterprise-owned organizations
- Audit which Apps are installed and which repositories they can access
- Add or remove repositories from an App installation
- Change an installation between all repositories and selected repositories

{: .prompt-tip }
> You can use an Enterprise-owned GitHub App to install another Enterprise-owned GitHub App into an organization, an organization-owned GitHub App, or even a third-party Marketplace app into an organization. The key difference is that the Enterprise-owned App has enterprise-level permissions and can be managed centrally by enterprise owners. See my post on [installing Marketplace apps with Enterprise Apps](/posts/github-enterprise-apps-install-marketplace-apps/) for a walkthrough.

Previously, installing an App still required someone to complete the web flow. Repository access could be changed through an organization-level [API](https://docs.github.com/en/enterprise-cloud@latest/rest/apps/installations?apiVersion=2022-11-28#add-a-repository-to-an-app-installation), but it required a classic PAT and could not change an installation between all and selected repositories. Auditing an App's repository access also required authenticating with the [App's JWT](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-bash-to-generate-a-jwt), [querying each organization installation](https://docs.github.com/en/enterprise-cloud@latest/rest/apps/apps?apiVersion=2022-11-28#get-an-organization-installation-for-the-authenticated-app), and inspecting its `repository_selection` value. The enterprise-level APIs make this workflow programmatic and centrally accessible.

Before diving into the examples, there are a few important things to know:

- **Prerequisites**: You'll need enterprise owner access (or delegated app manager permissions), and an [Enterprise GitHub App](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-your-enterprise-account/creating-github-apps-for-your-enterprise) with write access for both **Enterprise organization installations** and **Enterprise organization installation repositories**. You'll also need to generate and safeguard the App's private key.
- **Authentication**: In my examples, I'm using the [`gh token`](https://github.com/Link-/gh-token) CLI command to generate a token for App authentication. You can also generate your own [JWT](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app#example-using-bash-to-generate-a-jwt) and [App installation token](https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app) using your preferred method.
- **API Documentation**: There are two different sets of API endpoints for Apps, and navigating the documentation can be tricky. We'll be using the **[REST API for managing organization GitHub App installations for Enterprise Administration](https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/organization-installations?apiVersion=2022-11-28)**, not the regular [REST API endpoints for GitHub Apps](https://docs.github.com/en/enterprise-cloud@latest/rest/apps?apiVersion=2022-11-28) (which are app and org-based, not enterprise).

### Examples (Bash)

Need to look up your enterprise installation ID first? With the new [enterprise installation API (public preview)](https://github.blog/changelog/2026-05-13-new-enterprise-installation-api-now-in-public-preview/), you can grab it with a JWT instead of digging through the browser address bar:

```bash
# Mint a JWT for the app (no --installation-id needed)
jwt=$(gh token generate --app-id 1891481 --key /Users/joshjohanning/Downloads/josh-github-enterprise-app.2025-09-03.private-key.pem --token-only)

# Look up the enterprise installation ID
GH_TOKEN=$jwt gh api /enterprises/avocado-corp/installation --jq '.id'
```
{: .nolineno}

{: .prompt-tip }
> If the App's first installation was at the enterprise (no orgs), you can skip the `--installation-id` flag entirely on `gh token generate`. Otherwise, the new API above is the cleanest way to discover it programmatically.

Now the full set of examples for managing app installations across enterprise organizations:

```bash
# Generate token for the enterprise app
# The --installation-id option can be omitted if the App's first installation was at the enterprise (no orgs)
# Otherwise, look it up programmatically with the new enterprise installation API (see above), or grab it from the address bar
token=$(gh token generate --app-id 1891481 --installation-id 84179086 --key /Users/joshjohanning/Downloads/josh-github-enterprise-app.2025-09-03.private-key.pem --token-only)

# Get repositories accessible to an app installed in org
# This effectively shows which repos the app is installed on
GH_TOKEN=$token gh api /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations/45357471/repositories --paginate --jq '.[].full_name'

# Get repositories belonging to an enterprise-owned organization
# Useful to know WHICH repos are available to install the app on
# or compare with the previous API to see which repos the app is NOT installed on
GH_TOKEN=$token gh api /enterprises/avocado-corp/apps/installable_organizations/joshjohanning-org/accessible_repositories --jq '.[].full_name' --paginate

# Flip to "selected repositories" for app installed in org
# Note this fails with `422` if you select invalid repos
GH_TOKEN=$token gh api --method PATCH /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations/45357471/repositories --input - <<< '{
  "repository_selection": "selected",
  "repositories": [
    "issueops-samples",
    "reusable-workflows"
  ]
}'

# Flip back to "all repos" for app installed in org
GH_TOKEN=$token gh api --method PATCH /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations/45357471/repositories --input - <<< '{
  "repository_selection": "all"
}'

# Uninstall app in an org
GH_TOKEN=$token gh api --method DELETE /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations/45357471

# You need to retrieve the client_id of the app being installed in order to install it
# The easiest way is to grab the app's client_id from the app's settings page
# Programmatically:
# - If the app is public, you can query the client_id with any authentication (including Enterprise GitHub App)
# - If the app is private, you can query the client_id using the API with your user token with the scopes:
#     $ gh auth login -s read:enterprise or gh auth login -s read:org
#     $ gh api /apps/josh-github-enterprise-app-disc --jq .client_id

# Install app in an org
# Note it needs client_id and not the app_id
GH_TOKEN=$token gh api --method POST /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations -f 'client_id=Iv1.1051aca2d4910a24' -f 'repository_selection=all'

# List all apps and their permissions in an organization
GH_TOKEN=$token gh api /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations --paginate
```
{: .nolineno}

## Current Limitations

As of May 2026, there are some limitations to be aware of:

- **Limited permission scope**: Not every permission is available at the Enterprise level yet (like managing Enterprise settings)
- **Enterprise webhooks**: Enterprise installations cannot subscribe to webhooks yet
- ~~**Third-party apps**: Enterprises can only install apps owned by the enterprise or organizations within the enterprise~~ -- This works! You just need the app's `client_id`. See my post on [installing Marketplace apps programmatically with Enterprise Apps](/posts/github-enterprise-apps-install-marketplace-apps/) for details.
- ~~**Enterprise installation discovery**: Previously, there was no direct API to look up an enterprise installation ID - you had to paginate through all installations or grab it from the browser~~ -- [Resolved in May 2026](https://github.blog/changelog/2026-05-13-new-enterprise-installation-api-now-in-public-preview/) with `GET /enterprises/{enterprise}/installation` (public preview)
- **Rate limits**: Enterprise installations have their own 15,000 requests/hour budget - but note each installation still has its own rate limit
- **Creating apps**: You cannot currently create Apps through the API; I recommend using the [manifest flow](https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest) for codifying the app permissions and creation process through the UI

{: .prompt-info }
> Keep an eye on the [GitHub roadmap](https://github.com/orgs/github/projects/4247) and [changelog](https://github.blog/changelog/2025/?label=enterprise-management-tools) for updates on enterprise GitHub App capabilities.

## Summary

[Enterprise-owned GitHub Apps](https://github.blog/changelog/2025-03-10-enterprise-owned-github-apps-are-now-generally-available/) (GA March 2025), the [Enterprise-level installation automation APIs](https://github.blog/changelog/2025-07-01-enterprise-level-access-for-github-apps-and-installation-automation-apis) (public preview July 2025), and the new [enterprise installation API](https://github.blog/changelog/2026-05-13-new-enterprise-installation-api-now-in-public-preview/) (public preview May 2026) solve the manual pain of managing apps across many organizations. The key benefit is eliminating the need to click through installation screens for every org, plus automatic permission propagation when you update app settings.

The bash examples above demonstrate the core operations: install, uninstall, change repository access, and audit existing installations. These [Enterprise-level GitHub App management APIs](https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/organization-installations?apiVersion=2022-11-28) are particularly valuable during migrations or when you need to deploy security/compliance apps enterprise-wide.

Happy automating! 🔑 🚀
