---
title: 'Finding deprecated set-output and save-state commands in GitHub Actions'
author: Josh Johanning
date: 2023-03-13 3:00:00 -0500
description: A bash script to find usage of deprecated set-output and save-state commands as well as finding deprecated Node.js 12 actions in GitHub Actions workflows
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, gh cli, Scripts]
media_subpath: /assets/screenshots/2023-03-13-deprecated-github-actions-commands
image:
  path: deprecated-workflow-command.png
  width: 100%
  height: 100%
  alt: Deprecated workflow command in GitHub Actions
---

## Overview

You might have been noticing some of these warnings in your GitHub Actions workflows:

> Warning: The `set-output` command is deprecated and will be disabled soon. Please upgrade to using Environment Files. For more information see: [https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/)
{: .prompt-warning }

~~GitHub is disabling the `set-output` and `save-state` commands in GitHub Actions on May 31, 2023, so it's crunch time in ensuring your workflows are updated to use the new command syntax.~~ The [changelog post](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/) has been updated to note that telemetry shows that there is still high usage of the deprecated workflow commands and they won't be disabled just yet.

For actions that you are authoring, the fix is relatively simply. 

Old: 

```yml
- name: Save state
  run: echo "::save-state name={name}::{value}"

- name: Set output
  run: echo "::set-output name={name}::{value}"
```

New:

```yml
- name: Save state
  run: echo "{name}={value}" >> $GITHUB_STATE

- name: Set output
  run: echo "{name}={value}" >> $GITHUB_OUTPUT
```

You could simply perform a code search and update instances of `set-output` and `save-state` to the new syntax. However, this wouldn't work for actions that you are consuming from the marketplace. For example, if you were using `actions/stale@v5.2.0` in your workflows, you would see this warning in your workflow run logs.

Thanks to [@teddyteh](https://github.com/teddyteh) in this [PR](https://github.com/joshjohanning/github-actions-log-warning-checker/pull/6), this script now also searches for deprecated [Node.js 12 actions](https://github.blog/changelog/2022-09-22-github-actions-all-actions-will-begin-running-on-node16-instead-of-node12/). The fix is relatively simply here too, in the `action.yml`{: .filepath} file, use `'node16'` instead of `'node12'`:

```yml
runs:
  using: 'node16'
  main: 'main.js'
```
{: file='action.yml'}

## Finding Deprecation Warning Command Usage

I created a [bash solution](https://github.com/joshjohanning/github-actions-log-warning-checker) to audit a list of repositories for deprecated workflow commands. It checks through recent workflow runs (configurable) to see if there are any `set-output`, `save-state`, or `Node.js 12` deprecation warnings in the workflow run logs. The outputs are stored to a specified CSV file and denotes if the deprecation message was found in the most recent workflow run or not. The reason why I'm searching through multiple workflow runs is that there is a possibility that certain actions aren't run on every workflow run, such as a PR build, so I wanted to ensure proper coverage.

### Usage

Repository: [joshjohanning/github-actions-log-warning-checker](https://github.com/joshjohanning/github-actions-log-warning-checker)

1. Run `gh auth login` to authenticate with GitHub CLI
2. Run `./generate-repos.sh <org> > repos.csv` 
    - Modify list as needed
    - Or create a list of repos in a csv file, `<org/<repo>`, 1 per line, with a trailing empty line at the end of the file
3. Run: `./github-actions-log-warning-checker.sh repos.csv output.csv`

### Example Output

```
repo,workflow_name,workflow_url,finding,found_in_latest_workflow_run
joshjohanning-org/actions-linter-testing,CI,https://github.com/joshjohanning-org/actions-linter-testing/blob/main/.github/workflows/blank.yml,Workflow command,no
joshjohanning-org/actions-linter-testing,new-workflow,https://github.com/joshjohanning-org/actions-linter-testing/blob/main/.github/workflows/new-file.yml,Workflow command,yes
joshjohanning-org/actions-linter-testing,node12,https://github.com/joshjohanning-org/actions-linter-testing/blob/main/.github/workflows/node12.yml,Node.js 12 action,yes
```
{: file='output.csv'}

### Sample repos.csv file to use for testing

You can use this `repos.csv`{: .filepath} file for testing. It has a finding for a result for a deprecated Node.js 12 action as well as a deprecated workflow command. 

```
joshjohanning-org/actions-linter-testing
joshjohanning-org/actions-linter-testing-clean

```
{: file='repos.csv'}

### To do

- [x] [Find deprecated Node.js 12 actions](https://github.com/joshjohanning/github-actions-log-warning-checker/issues/2) (thanks to [@teddyteh](https://github.com/teddyteh) in this [PR](https://github.com/joshjohanning/github-actions-log-warning-checker/pull/6))
- [ ] [Use annotations to be able to return workflow run log line links](https://github.com/joshjohanning/github-actions-log-warning-checker/issues/3) (for easier discoverability of which action is using the deprecated command(s))

## Summary

As an Enterprise/Organization owner, this tool can be useful to find repositories that are using deprecated workflow commands. You can then reach out to the owners of the repositories to let them know that they need to update their workflows. This can also be useful for repository admins as well, and just feed in the list of repositories that you manage. 

Going forward, make sure to set up [`dependabot.yml`{: .filepath}](https://josh-ops.com/posts/github-dependabot-for-actions/#marketplace-actions) file to keep your actions up to date. This also applies for actions that are internal to your enterprise/organization, but with additional [configuration required](https://josh-ops.com/posts/github-dependabot-for-actions/#custom-actions-in-organization) (see my [post](https://josh-ops.com/posts/github-dependabot-for-actions/#custom-actions-in-organization) for more details!).

Also, check out [another user's solution](https://github.com/orgs/community/discussions/49405#discussioncomment-5227815) to the problem I'm solving here! 
