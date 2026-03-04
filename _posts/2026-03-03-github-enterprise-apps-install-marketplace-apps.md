---
title: 'Installing GitHub Marketplace Apps Programmatically with Enterprise Apps'
author: Josh Johanning
date: 2026-03-03 20:00:00 -0600
description: How to use Enterprise GitHub Apps to programmatically install third-party GitHub Marketplace apps across your enterprise organizations
categories: [GitHub, Apps]
tags: [GitHub, GitHub Apps, GitHub Enterprise Management, GitHub Marketplace]
media_subpath: /assets/screenshots/2026-03-03-github-enterprise-apps-install-marketplace-apps
image:
  path: marketplace-apps-light.png
  width: 100%
  height: 100%
  alt: The GitHub Marketplace showing available apps for installation
---

## Overview

In my previous post on [using Enterprise GitHub Apps for programmatic app management](/posts/github-enterprise-apps/), I noted that enterprises could only install apps owned by the enterprise or its organizations. My colleague [@tspascoal](https://github.com/tspascoal) discovered that you can also use Enterprise GitHub Apps to **install third-party Marketplace apps** into your organizations programmatically.

This is a big deal for enterprises that need to roll out tools like Azure Boards/Pipelines, Jira Cloud, or any other Marketplace app across dozens (or hundreds) of organizations without clicking through installation screens one by one.

> If you're new to Enterprise GitHub Apps, start with my [Enterprise GitHub Apps](/posts/github-enterprise-apps/) post for the full background on enterprise-level app management and the installation automation APIs.
{: .prompt-info }

## How It Works

The installation API is the same one covered in my [previous post](/posts/github-enterprise-apps/#installation-automation-api-examples). Instead of using the `client_id` of an enterprise or org-owned app, you use the **`client_id`** of the Marketplace app you want to install. There are two ways to get it.

### Option 1: Look Up the App Slug

Every GitHub App has a slug (the URL-friendly name you see in `github.com/apps/<slug>`). You can query the app's `client_id` directly:

```bash
gh api apps/azure-boards --jq .client_id
```
{: .nolineno}

This returns:

```text
Iv1.27a3a93db4db56bb
```
{: .nolineno}

> The app slug is **not** the same as the Marketplace listing URL. For example, the Shopify Marketplace listing is at `github.com/marketplace/shopify-online-store`, but the actual app slug is `shopify` (`github.com/apps/shopify`). The easiest way to find the real slug is to Google something like `github app sonarqube cloud site:github.com/apps` to land directly on the `github.com/apps/<slug>` page. Or, you can also look it up from an existing installation using [Option 2](#option-2-find-it-from-existing-org-installations) below.
{: .prompt-warning }

### Option 2: Find It from Existing Org Installations

If the app is already installed in one of your organizations, you can pull the `client_id` from there:

```bash
gh api orgs/my-org/installations --paginate --jq '.installations[] | {app_slug, client_id}' | grep azure-boards
```
{: .nolineno}

This returns something like:

```json
{
  "app_slug": "azure-boards",
  "client_id": "Iv1.27a3a93db4db56bb"
}
```
{: .nolineno}

## Common App Client IDs

To save you the trouble, here are the slugs and `client_id` values for some commonly used Marketplace apps:

| App | Slug | Client ID |
|---|---|---|
| Atlassian (Jira/Bitbucket) | `atlassian` | `Iv1.45aafbb099e1c1d7` |
| Azure Boards | `azure-boards` | `Iv1.27a3a93db4db56bb` |
| Azure Pipelines | `azure-pipelines` | `Iv1.3b08983b5115f30d` |
| CircleCI | `circleci-app` | `Iv1.7da85473579584a7` |
| Docker | `docker` | `Iv1.75f4ac19dd8b9b71` |
| Microsoft Teams | `microsoft-teams-for-github` | `Iv1.2b3f60f5d77cb30d` |
| Slack | `slack` | `Iv1.37ba7cd714104547` |
| SonarQube Cloud | `sonarqubecloud` | `Iv1.dc610988574d1724` |
| Terraform Cloud | `terraform-cloud` | `Iv1.a2b16df02bc2b121` |

## Installing the Marketplace App

Once you have the `client_id`, installation is straightforward using the same Enterprise installation automation API:

```bash
# Generate token for the enterprise app (with "Enterprise organization installations" write permission)
token=$(gh token generate \
  --app-id 1891481 \
  --installation-id 84179086 \
  --key ~/Downloads/my-enterprise-app.private-key.pem \
  --token-only)

# Install the Marketplace app (e.g., Azure Boards) into an org
GH_TOKEN=$token gh api \
  --method POST \
  /enterprises/avocado-corp/apps/organizations/joshjohanning-org/installations \
  -f 'client_id=Iv1.27a3a93db4db56bb' \
  -f 'repository_selection=all'
```
{: .nolineno}

That's it! The Marketplace app is now installed in the target organization. You can also use `repository_selection=selected` along with a `repositories` array if you only want specific repos - I have examples of this in my [Enterprise GitHub Apps post](/posts/github-enterprise-apps/#installation-automation-api-examples).

> The Enterprise GitHub App performing the installation needs the **Enterprise --> "Enterprise organization installations" (write)** permission. See the [prerequisites](/posts/github-enterprise-apps/#installation-automation-api-examples) in my previous post for full setup details.
{: .prompt-tip }

## Scaling Across Organizations

You can combine this with a simple loop to install the app across all (or a subset of) organizations in your enterprise:

```bash
# Get all orgs and install the app in each one
for org in $(GH_TOKEN=$token gh api /enterprises/avocado-corp/apps/installable_organizations --jq '.[].login' --paginate); do
  echo "Installing azure-boards in $org..."
  GH_TOKEN=$token gh api \
    --method POST \
    "/enterprises/avocado-corp/apps/organizations/$org/installations" \
    -f 'client_id=Iv1.27a3a93db4db56bb' \
    -f 'repository_selection=all'
done
```
{: .nolineno}

> If the app is already installed in an organization, the API will return the existing installation details with a `200` status instead of creating a duplicate. This means the loop is safe to run repeatedly without errors.
{: .prompt-info }

## Summary

Enterprise GitHub Apps can install third-party Marketplace apps just as easily as organization-owned apps. The process is simple: look up the app's `client_id` from its slug, then use the same [Enterprise installation automation API](/posts/github-enterprise-apps/#installation-automation-api-examples) to install it across your organizations.

Happy click saving! 🛒 🚀
