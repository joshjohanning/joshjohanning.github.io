---
title: 'Enhancing GitHub Migrations with Additional Tooling'
author: Josh Johanning
date: 2024-09-18 10:30:00 -0500
description: A collection of additional tools to assist with your GitHub migration experience
categories: [GitHub, Migrations]
tags: [GitHub, Migrations, Azure DevOps, Git, gh cli, Scripts, Bitbucket]
---

## Overview

When migrating to GitHub, there are a couple of options. You could do a [`git clone --mirror / git push`](/posts/migrating-repos-to-github/#option-2-git-clone-mirror), but if you're migrating from another GitHub source, Azure DevOps, or Bitbucket Server, you should be using the official GitHub Enterprise Importer (GEI) tooling. This not only migrates the code with the history, but also additional metadata such as pull requests, issues (if GitHub), existing branch protection settings, and more.

GEI is great, but [not everything is migrated](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-between-github-products/about-migrations-between-github-products#data-that-is-not-migrated-1). Don't worry; there are some additional tools that can help enhance the migration experience and fill in these gaps. This post contains a collection of open-source tools that I've found useful when migrating to GitHub.

## Supplementary Migration Tooling Collection

Based on my migration experience, here are additional tools I've found useful, organized by items not migrated by GEI and the corresponding supplementary tooling:

| Item | Tooling | Notes |
|------|---------|-------|
| **Organization** | | |
| - Metadata | N/a | Name of org, description, settings, OAuth app policy, scheduled reminders, org owners, etc. |
| - Custom repo roles | [Analysis script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-custom-repository-roles-count.sh) | Any custom org roles will need to be migrated as well |
| - Org level webhooks | [Analysis script (count)](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/gh-cli/get-organizations-webhooks-count.sh),<br>[Analaysis script (detailed)](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-webhooks.sh),<br>[gh-organization-webhooks](https://github.com/katiem0/gh-organization-webhooks) | Need to know what webhook secrets are, can't retrieve in UI/API |
| - IP allow list | [Get org IP allow list](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organization-ip-allow-list.sh),<br>[Get enterprise IP allow list](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-enterprise-ip-allow-list.sh),<br>[Set IP allow list rules for](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/set-ip-allow-list-rules.sh) | The get scripts save rules to CSV and the set script sets them in target |
| **Discussions** | [Analysis script (count) for each org](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-discussions-count.sh),<br>[Analysis script (count) for each repo in an org](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-repositories-discussions-count.sh) | Discussions exist in repos, but may have to configure which repo will be used for org discussions |
| **Projects** | | |
| - Projects v2 | [Analysis script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-projects-count.sh),<br>[gh-migrate-project](https://github.com/timrogers/gh-migrate-project?tab=readme-ov-file) | CLI utility can help migrate org-level projects |
| - Org Projects (classic) | [Analysis script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-projects-count-classic.sh) | Deprecated |
| - Repo Projects (classic) | N/a | Deprecated |
| **GitHub apps** | [Analysis script for org apps](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-apps.sh),<br>[Analysis script by org app count](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-apps-count.sh) | The [manifest flow](https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest) help when recreating apps manually |
| **Teams / membership** | [gh-migrate-teams](https://github.com/mona-actions/gh-migrate-teams),<br>[gh-migrate-team-permission](https://github.com/mona-actions/gh-migrate-team-permission),<br>[Recreate security in repos & teams](https://github.com/joshjohanning/github-misc-scripts/tree/main/scripts/recreate-security-in-repositories-and-teams),<br>[Create teams from list](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/create-teams-from-list.sh),<br>[gh-collaborators](https://github.com/katiem0/gh-collaborators) | Use the [recreation script](https://github.com/joshjohanning/github-misc-scripts/tree/main/scripts/recreate-security-in-repositories-and-teams) if wanting to mirror teams/membership |
| **User settings** | N/a | PATs, SSH keys, notification settings |
| **Webhook secrets** | [Script to analyze webhooks](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-repositories-webhooks-csv.sh),<br>[gh-migrate-webhook-secrets CLI](https://github.com/mona-actions/gh-migrate-webhook-secrets) | Repo-level webhooks migrate, but webhook secrets need to be recreated |
| **Actions** | | Action runs don't migrate, workflows will migrate with code, and everything below will need to be recreated |
| - Repo/org secrets | [gh-secrets-migrator](https://github.com/dylan-smith/gh-secrets-migrator),<br>[gh-seva](https://github.com/katiem0/gh-seva?tab=readme-ov-file) | Actions secrets values can only be retrieved during Actions runtime |
| - Environments | [gh-environments](https://github.com/katiem0/gh-environments) | Environments need to be recreated |
| - Variables | [gh-seva](https://github.com/katiem0/gh-seva?tab=readme-ov-file) | Variables need to be recreated |
| - Self-hosted runners | [Analysis script for all org runners](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organization-self-hosted-runners-organization-runners.sh),<br>[Analysis script for all repo runners in org](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organization-self-hosted-runners-repository-runners.sh),<br> [Analysis script for repo+org runners in org](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organization-self-hosted-runners-all-runners.sh),<br>[Analysis script for enterprise runners](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-enterprise-self-hosted-runners.sh) | Runners need to be re-created |
| - Larger runners | N/a | Larger GitHub-hosted runners need to be re-created; no API for large runners |
| **Rulesets** | [gh-migrate-rulesets](https://github.com/katiem0/gh-migrate-rulesets) | Rulesets are not migrated |
| **Packages** | [See package migration posts](/categories/packages/) | Packages are not migrated |
| **Code owners** | [Analysis script](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-organizations-repositories-codeowner-usage.sh),<br>[Code owners mapping helper](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/update-codeowners-mappings.js) | Updating org/team names |
| **LFS** | [Migrate LFS artifacts](/posts/migrate-git-lfs-artifacts/) | LFS is not migrated |
| **Username mapping** | [Getting SAML entities at enterprise](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-saml-identities-in-enterprise.sh),<br>[Getting SAML entities at org](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-saml-identities-in-organization.sh) | Getting SAML identities can help map personal github.com accounts by tying their email to their identity provider credential |
| **Repository visibility** | [gh-repo-visibility](https://github.com/mona-actions/gh-repo-visibility) | Repos migrate as private by [default](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-between-github-products/overview-of-a-migration-between-github-products#setting-repository-visibility) |
| **Deploy keys** | [gh-migrate-deploy-keys](https://github.com/mona-actions/gh-migrate-deploy-keys) | Deploy keys are not migrated |

## Migration Planning Tooling

These tools are more for helping you plan / track a migration. For example, you can use some of these tools to potentially identify problem repositories before you start the migration (e.g. with `git-sizer`, repositories that contain large files committed), or otherwise repositories with a lot of pull requests that will take longer to migrate.

| Tool | Description |
| --- | --- |
| **[gh-repo-stats](https://github.com/mona-actions/gh-repo-stats)** | GitHub CLI extension to pull statistics on repository metadata used in GitHub migrations |
| **[gh-migration-audit](https://github.com/timrogers/gh-migration-audit)** | Audits GitHub repositories to highlight data that cannot be automatically migrated using GitHub's migration tools |
| **[gh ado2gh inventory-report](https://docs.github.com/en/enterprise-cloud@latest/migrations/overview/planning-your-migration-to-github#building-a-basic-inventory-of-the-repositories-you-want-to-migrate)** | Azure DevOps to GitHub inventory report using the GEI commands |
| **[git-sizer](https://github.com/github/git-sizer)** & **[gh-sizer](https://github.com/timrogers/gh-sizer)** | Compute various size metrics for a Git repository, flagging those that might cause problems |
| **[gh-bbs-analyzer](https://github.com/mona-actions/gh-bbs-analyzer)** | GitHub CLI extension for analyzing BitBucket Server to get migration statistics |
| **[gh-gitlab-stats](https://github.com/mona-actions/gh-gitlab-stats)** | GitHub CLI extension to pull statistics on GitLab repository and server metadata |
| **[gh-pma](https://github.com/mona-actions/gh-pma)** | Post-Migration Audit (PMA) Extension For GitHub CLI |
| **[github-migration-monitor](https://github.com/timrogers/github-migration-monitor)** | Monitors GitHub Enterprise Importer (GEI) migrations for an organization through a simple command line tool |

## Non-technical Migration Planning Tips

Here are list of miscellaneous non-technical considerations to keep in mind when planning a migration:

- You should have training material in a wiki or similar for developers to reference. This should include a list of "what's changing" and what things developers need to do in the new environment. Things to consider:
  - Either adding a new remote to a local repository or cloning a new repository from the EMU
  - Recreating PATs and SSH keys (note that SSH keys have to be globally unique in github.com; if they want to re-use an existing key, they will have to remove it from the old account first)
  - For PATs and SSH keys, note that users may have to [authorize tokens for SSO](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-saml-single-sign-on/authorizing-a-personal-access-token-for-use-with-saml-single-sign-on) after creating them, which may be a new step for them
- Creating a migration plan, documenting what repositories are moving and when, as well as who is responsible for each pre and post-migration step
- Running through a dry run first, which helps you identify if there will be any issues ahead of time, as well as getting a sense of the timing
  - Dry run means running a migration, but don't cut over or lock the source (you should be able to run the migration again and again with the same repositories and have the same results)
- If you are migrating from github.com to another enterprise in github.com, the new organization name must be globally unique - however, you can rename the original organization and then right after, rename the new organization to the original name
  - Make sure you don't delete any organization where you want to keep the name, as the name will be locked for 90 days after the deletion
  - It is better to rename and then delete in this case
- Links
  - If org name or repo name changes, existing links will be broken
  - Private/non-public GitHub Pages will have a new URL, so any links to the old pages will be broken

## Special Thanks

Special thanks to everyone who has contributed to these OSS tools!

- [katiem0](https://github.com/katiem0)
- [tspascoal](https://github.com/tspascoal)
- [timrogers](https://github.com/timrogers)
- [mickeygousset](https://github.com/mickeygousset)
- [dylan-smith](https://github.com/dylan-smith)
- [amenocal](https://github.com/amenocal)
- [andyfeller](https://github.com/andyfeller)
- [antgrutta](https://github.com/antgrutta)
- [bryantson](https://github.com/bryantson)
- [pmartindev](https://github.com/pmartindev)
- [samueljmello](https://github.com/samueljmello)
- And many more I am SURE I am missing

## Summary

Hopefully, this collection of migration tooling and tips helps you out! I'll use this post as a living document and update it as I find new tools or tips. If you have any suggestions or additions, please add a comment! 🙌 ✨
