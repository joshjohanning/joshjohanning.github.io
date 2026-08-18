---
title: 'GitHub Issues: Automating Sub-Issues and Issue Types'
author: Josh Johanning
date: 2024-11-15 1:00:00 -0600
description: A guide to automating GitHub sub-issues and issue types with the REST API, GitHub CLI, and GraphQL
categories: [GitHub, Scripts]
tags: [GitHub, GitHub Issues, GitHub Projects]
media_subpath: /assets/screenshots/2024-11-15-github-sub-issues-and-issue-types
image:
  path: sub-issues-issue-types-light.png
  width: 100%
  height: 100%
  alt: Sub-Issues and Issue Types in GitHub Issues
---

## Overview

[Sub-issues and issue types became generally available](https://github.blog/changelog/2025-04-09-evolving-github-issues-and-projects/) in April 2025. They are available without an opt-in, and API calls no longer need feature-preview headers.

Sub-issues let you break work into a hierarchy and track progress from a parent issue. Issue types provide an organization-wide classification shared by every repository in the organization. GitHub now supports both features in the REST API, and GitHub CLI v2.94.0 and later has first-class flags for common workflows.

> Check out [@mickeygousset](https://github.com/mickeygousset)'s videos for working with [sub-issues](https://www.youtube.com/watch?v=F42FN6cZmA4) and [issue types](https://www.youtube.com/watch?v=2wVmcuCC1is)! ✨
{: .prompt-info }

## Sub-Issues

### Current Limits and Behavior

- A parent issue can have up to **100 direct sub-issues**.
- A hierarchy can contain up to **eight levels** of nested sub-issues.
- A sub-issue can itself be a parent issue.
- Existing issues can be added from another repository owned by the same user or organization. The REST `Add sub-issue` endpoint requires the parent and child repositories to have the same owner.
- Each issue can have only one parent. Pass `replace_parent=true` to the REST endpoint when moving a sub-issue to a new parent.
- Sub-issue progress appears on the parent issue and in GitHub Projects. REST issue responses include `sub_issues_summary` and `parent_issue_url`.

See [Adding sub-issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues) for the current product behavior.

> Before adding a sub-issue, choose the relationship that best represents how the issues are connected:
>
> - **Parent/sub-issue:** Use for a hierarchy in which a larger issue contains smaller pieces of work.
> - **Blocking/blocked by:** Use for a dependency in which one issue must be completed before another can proceed. [Issue dependencies are generally available](https://github.blog/changelog/2025-08-21-dependencies-on-issues/).
> - **Relates to:** Use for connected issues when neither hierarchy nor dependency applies. See the [`Relates to` relationship announcement](https://github.blog/changelog/2026-08-07-connecting-issues-and-multi-select-field-support/) for more information.
{: .prompt-info }

### REST API Examples

The examples use [`gh api`](https://cli.github.com/manual/gh_api), so authentication and the recommended REST headers are handled by GitHub CLI.

#### Get the Parent Issue

The dedicated REST endpoint replaces the original GraphQL lookup:

```sh
gh api repos/OWNER/REPO/issues/SUB_ISSUE_NUMBER/parent \
  --jq '{number, title, url: .html_url, issueType: .type.name}'
```
{: .nolineno }

#### List Sub-Issues

Use `--paginate` because the endpoint returns 30 items per page by default:

```sh
gh api --paginate repos/OWNER/REPO/issues/PARENT_ISSUE_NUMBER/sub_issues \
  --jq '.[] | {number, title, url: .html_url, issueType: .type.name}'
```
{: .nolineno }

#### Get the Sub-Issue Progress Summary

The standard REST issue response contains the summary:

```sh
gh api repos/OWNER/REPO/issues/PARENT_ISSUE_NUMBER \
  --jq '.sub_issues_summary'
```
{: .nolineno }

An example response looks like:

```json
{
  "total": 3,
  "completed": 1,
  "percent_completed": 33
}
```
{: .nolineno }

#### Add an Existing Issue as a Sub-Issue

The API requires the database ID returned in the REST issue's `id` field, not the issue number:

```sh
sub_issue_id=$(gh api repos/OWNER/REPO/issues/SUB_ISSUE_NUMBER --jq '.id')

gh api --method POST \
  repos/OWNER/REPO/issues/PARENT_ISSUE_NUMBER/sub_issues \
  -F sub_issue_id="$sub_issue_id"
```
{: .nolineno }

To move an issue that already has a parent, include `-F replace_parent=true`.

#### Remove a Sub-Issue

Removing a sub-issue also uses its REST database ID:

```sh
sub_issue_id=$(gh api repos/OWNER/REPO/issues/SUB_ISSUE_NUMBER --jq '.id')

gh api --method DELETE \
  repos/OWNER/REPO/issues/PARENT_ISSUE_NUMBER/sub_issue \
  -F sub_issue_id="$sub_issue_id"
```
{: .nolineno }

#### Reprioritize a Sub-Issue

Place one sub-issue after another in the parent's ordered list:

```sh
sub_issue_id=$(gh api repos/OWNER/REPO/issues/SUB_ISSUE_NUMBER --jq '.id')
after_id=$(gh api repos/OWNER/REPO/issues/AFTER_ISSUE_NUMBER --jq '.id')

gh api --method PATCH \
  repos/OWNER/REPO/issues/PARENT_ISSUE_NUMBER/sub_issues/priority \
  -F sub_issue_id="$sub_issue_id" \
  -F after_id="$after_id"
```
{: .nolineno }

Use `before_id` instead of `after_id` to position it before another sub-issue. The complete endpoint reference is in [REST API endpoints for sub-issues](https://docs.github.com/en/rest/issues/sub-issues).

## Issue Types

### Current Limits and Behavior

- An organization can have up to **25 issue types**.
- GitHub provides `Task`, `Bug`, and `Feature` by default. Organization owners can edit, disable, or delete them and create custom types.
- An issue can have one issue type, and the type is shared across repositories in the organization.
- Disabling a type prevents it from being selected but preserves it on existing issues. Deleting a type permanently removes it.
- Issue types can be used in issue and project filters, such as `type:Bug` and `no:type`.

See [Managing issue types in an organization](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/managing-issue-types-in-an-organization) for the current product behavior.

### REST API Examples

#### Read an Issue's Type

The standard issue response now includes the `type` object:

```sh
gh api repos/OWNER/REPO/issues/ISSUE_NUMBER \
  --jq '{number, title, url: .html_url, issueType: .type.name}'
```
{: .nolineno }

#### List Organization Issue Types

```sh
gh api orgs/ORG/issue-types \
  --jq '.[] | {id, name, description, color, is_enabled}'
```
{: .nolineno }

#### Set or Remove an Issue Type

REST accepts the issue type name when creating or updating an issue. Set the name to `null` to remove it:

```sh
gh api --method PATCH repos/ORG/REPO/issues/ISSUE_NUMBER -f type='Bug'

gh api --method PATCH repos/ORG/REPO/issues/ISSUE_NUMBER -F type=null
```
{: .nolineno }

#### Create an Organization Issue Type

Organization owners can create and manage issue types:

```sh
gh api --method POST orgs/ORG/issue-types \
  -f name='Epic' \
  -f description='A large body of work spanning multiple issues' \
  -f color='green' \
  -F is_enabled=true
```
{: .nolineno }

The organization endpoints also support updating and deleting types. See [REST API endpoints for issue types](https://docs.github.com/en/rest/orgs/issue-types).

## GitHub CLI Commands

With GitHub CLI v2.94.0 or later, common operations no longer need `gh api` (GitHub Enterprise Server requires GHES 3.17 or later):

```sh
gh issue create --repo OWNER/REPO --title 'Fix login page' --type 'Bug' --parent 5
gh issue edit 5 --repo OWNER/REPO --add-sub-issue 6
gh issue edit 5 --repo OWNER/REPO --remove-sub-issue 6
gh issue edit 6 --repo OWNER/REPO --type 'Task'
gh issue view 5 --repo OWNER/REPO --json type,parent,subIssues
```
{: .nolineno }

## Existing Helper Scripts

The original scripts remain linked below, but most are now **legacy GraphQL implementations** superseded by REST endpoints or first-class `gh issue` options:

| Script | Status |
| --- | --- |
| [`get-parent-issue-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-parent-issue-of-issue.sh) | Legacy GraphQL; use `GET .../parent` |
| [`get-sub-issues-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-sub-issues-of-issue.sh) | Legacy GraphQL; use `GET .../sub_issues` |
| [`get-sub-issues-summary-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-sub-issues-summary-of-issue.sh) | Legacy GraphQL; use `GET .../issues/{number}` and `.sub_issues_summary` |
| [`add-sub-issue-to-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/add-sub-issue-to-issue.sh) | Legacy GraphQL; use `POST .../sub_issues` or `gh issue edit --add-sub-issue` |
| [`remove-sub-issue-from-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/remove-sub-issue-from-issue.sh) | Legacy GraphQL; use `DELETE .../sub_issue` or `gh issue edit --remove-sub-issue` |
| [`get-issue-type-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-issue-type-of-issue.sh) | Legacy GraphQL; use the REST issue's `.type` field |
| [`update-issue-issue-type.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/update-issue-issue-type.sh) | Legacy GraphQL; use `PATCH .../issues/{number}` or `gh issue edit --type` |
| [`remove-issue-issue-type.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/remove-issue-issue-type.sh) | Legacy GraphQL; use `PATCH .../issues/{number}` with `type: null` |

I verified that all linked files still exist and pass Bash syntax and usage checks. They currently use obsolete GraphQL feature headers, so new automation should prefer the REST and GitHub CLI examples above.

## When GraphQL Still Provides Value

REST is now the simplest choice for individual operations. GraphQL remains useful when one request needs a custom-shaped issue hierarchy or related project data that would otherwise require several REST calls. For example, this query retrieves the parent, progress summary, and first page of sub-issues together:

```graphql
query ($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    issue(number: $number) {
      parent {
        number
        title
      }
      subIssuesSummary {
        total
        completed
        percentCompleted
      }
      subIssues(first: 100) {
        nodes {
          number
          title
          issueType {
            name
          }
        }
      }
    }
  }
}
```

This read-only query does not need the old `GraphQL-Features: sub_issues` or `GraphQL-Features: issue_types` feature headers.

## Summary

Sub-issues and issue types are generally available, supported by dedicated REST endpoints, and built into current versions of GitHub CLI. Prefer `gh issue` for interactive command-line workflows and `gh api` with REST for automation. Keep GraphQL for queries that benefit from selecting several related fields in one response.
