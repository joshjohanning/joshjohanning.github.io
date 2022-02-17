---
title: 'ApproveOps: GitHub IssueOps with Approvals'
author: Josh Johanning
date: 2022-02-08 8:30:00 -0600
description: Using GitHub Actions to build automation on top of Issues (IssueOps) with Approvals from someone in a designated GitHub team
categories: [GitHub, Actions]
tags: [GitHub, GitHub Issues, GitHub Actions, GitHub Apps, IssueOps]
img_path: /assets/screenshots/2022-02-08-github-approveops
image:
  src: ../2022-02-07-github-apps/app-bot.png
  width: 100%
  height: 100%
  alt: ApproveOps - GitHub IssueOps with Approvals
---

## Overview

_This is follow-up to my previous post, [Demystifying GitHub Apps: Using GitHub Apps to Replace Service Accounts](/posts/github-apps), where I go over the basics of creating a GitHub App and accessing its installation access token in an action_

I was working with a customer and we built out a self-service migration solution to migrate from GitHub Enterprise Server to GitHub Enterprise Cloud. Instead of building our own custom interface, we decided to leverage GitHub Issues as the intake and GitHub Actions as the automation mechanism. This is often referred to as "IssueOps". 

There are several great examples of [IssueOps on GitHub](https://github.com/topics/issueops), as well as my co-worker's [ChatOps](https://colinsalmcorner.com/chatops-with-github-actions-and-azure-web-apps/) workflow. 

The benefit of using GitHub and IssueOps for something like this is the transparency of the process - the Issue is available for everyone to see as well as the logs of the Action. The customer just inputs the Git repository URL's and a few other inputs, the issue body is parsed, and a pre-migration comment is automatically posted back to the issue by our bot with a slash command that is used to trigger the migration.

However, a requirement for the customer was to have a way to approve the migration. We could have had an [approval on an environment on the job](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment), but that interface brought you away from the issue. For example, after someone initiates a deployment, the requestor doesn't necessarily know it's just sitting for an approval. While the approver(s) would get an email and optionally a push notification in the GitHub mobile app, it doesn't seem there is anything that shows up under the [GitHub notifications bell](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/about-notifications). 

If only there was a way to only allow the deployment if someone authorized issued some sort of command ahead of time in the issue... This is where ApproveOps comes in!

## The Solution: ApproveOps

As I hinted, ApproveOps is a simple extension of IssueOps that requires a slash command in the issue comment body from someone in the 'migration approval' team. In our solution, the slash command to run the deployment was `/run-migration` and our approval command we used was `/approve`. 

If a user runs the migration command without someone approving ahead of time, a bot will comment on the issue saying that it isn't yet approved and to have someone in the approval team approve the migration by entering in the approval command. If someone who isn't in the specified approval team tries to approve the migration, their approval comment will simply be ignored because they aren't in the approval team in GitHub.

Here's how it looks and works in the issue:

![ApproveOps](approveops.png ){: .shadow }
_ApproveOps sample - the `/run-migration` command doesn't run the workflow unless someone authorized has commented `/approve`_

And here is how one of the runs looks in GitHub Actions:

![ApproveOps Action Run](approveops-action-run.png ){: .shadow }
_The ApproveOps run in GitHub Actions - the migration job is skipped if no one has approved in the issue yet_

## The Code

I recently created a [GitHub Action published on the marketplace](https://github.com/marketplace/actions/approveops-approvals-in-issueops) that consolidates the various actions and bash commands. If you're not interested in using the marketplace action or want to extend what I've done, my [GitHub Actions workflow sample](https://github.com/joshjohanning/ApproveOps/blob/main/.github/workflows/approveops.yml) is still available in its entirety.

Here's how you can use my Action in a GitHub Action workflow:

{% raw %}
```yml
name: Approve Ops
on:
  issue_comment:
    types: [created, edited]

jobs:
  approveops:
    runs-on: ubuntu-latest
    if: contains(github.event.comment.body, '/run-migration')
    # optional - if we want to use the output to determine if we run the migration job or not
    outputs: 
      approved: ${{ steps.check-approval.outputs.approved }}

    steps:
      - name: ApproveOps - ApproveOps in IssueOps
        uses: joshjohanning/ApproveOps@main
        id: check-approval
        with:
          app-id: 170284
          app-private-key: ${{ secrets.PRIVATE_KEY }}
          team-name: approver-team
          fail-if-approval-not-found: false

  migration:
    runs-on: ubuntu-latest
    needs: approveops
    # optional - if we want to use the output to determine if we run the migration job or not
    if: ${{ steps.approveops.outputs.approved == 'true' }}

    steps:
      - run: echo "run migration!"
```
{: file='.github/workflows/approveops.yml'}
{% endraw %}

## Setup and Explanation

### Prerequisites

- Since accessing team membership is outside the scope of the [GitHub Token](https://dev.to/github/the-githubtoken-in-github-actions-how-it-works-change-permissions-customizations-3cgp), we have to either use our [GitHub App created in this related post](/posts/github-apps/#scenario-2-using-a-github-app-as-a-rich-comment-bot) or create a PAT with `read:org` scope and use it to get the team membership
- At least one member in the approver team, otherwise `jq` won't be able to find the `.[].login` property (this could probably be written more defensively :) )

### Explanation

I am using a [bash script in my composite action](https://github.com/joshjohanning/ApproveOps/blob/main/.github/workflows/approveops.yml#L28:L53) to:

1. Get a list of users in the approval team
2. Get a list of comments on the issue (note that I am converting the comments to base64 otherwise comments that had spaces in them would throw the loop off - this was a [good resource for explaining that](https://www.starkandwayne.com/blog/bash-for-loop-over-json-array-using-jq/))
3. Check that the comment issue body contains the `/approve` command
4. If so, check if the user who posted the `/approve` command is in the approval team (from step 1) by using a `grep` command
5. Setting an [output parameter](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-output-parameter) depending if the migration is authorized to run or not
6. If there aren't any authenticated approvals, I'm also setting a user-friendly [error message](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-error-message) to be helpful when looking at why the action run failed
7. Afterwards, we use the output parameter and `if:` logic to post the write comment on the workflow, either requesting proper approval or informing the user that the migration will now run since it has been approved
8. I'm using additional `if:` logic and the `approveops` jobs' output to determine if the `migration` job should run or not
    - As an alternative, I could see one omitting this using a single job with additional `if:` logic on the rest of the migration steps, but this could be messy
    - If using the same job and you didn't mind seeing failed runs in the UI because of lack of approvals, you could fail the workflow run by setting `fail-if-approval-not-found: true`

## Wrap-up

There's definitely room for improvement here, but I think this is a good starting point for you to get with your own ApproveOps / IssueOps workflow.

If you do what I'm doing here, by creating your own [GitHub App on your organization](/posts/github-apps#scenario-2-using-a-github-app-as-a-rich-comment-bot) and use that identity to write your comment, it will be able to properly `@team` mention in the comment and everything!

Report back if you make any enhancements!
