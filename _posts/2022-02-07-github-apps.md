---
title: 'Demystifying GitHub Apps: Using GitHub Apps to Replace Service Accounts'
author: Josh Johanning
date: 2022-02-07 12:00:00 -0600
description: Creating no-code GitHub Apps to install to an organization to replace having to create service accounts or a user PAT for authorization in GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, GitHub Apps]
img_path: /assets/screenshots/2022-02-07-github-apps
image:
  src: github-apps.png
  width: 100%
  height: 100%
  alt: An example GitHub App
---

## Overview

In GitHub Actions, the [GitHub Token](https://dev.to/github/the-githubtoken-in-github-actions-how-it-works-change-permissions-customizations-3cgp) works very well and is convenient for automation tasks that require authentication, but its scope is limited. The GitHub Token is only going to allow us to access data within the repository (such as issues, code, packages), but what if you need to authenticate to another repository, or access organizational information such as teams or member lists? GitHub Token is not going to work for that. Your alternatives are:

1. Use someone's Personal Access Token (PAT) - but what happens if that person leaves? Or if you need to write back to an issue, for example, it's going to look like it came from that user
2. Create a service account - but this is going to consume a license, and you still have to manage with vaulting and storing a long-lived PAT somewhere, and if that PAT gets exposed, you're opening yourselves up to a huge security risk
3. GitHub Apps!

In this post, I will go through the setup and usage of GitHub Apps in an Actions workflow with two scenarios: [Using a GitHub App to grant access to a single repository](#scenario-1-using-a-github-app-to-grant-access-to-a-single-repository) and [Using a GitHub App as a rich comment bot](#scenario-2-using-a-github-app-as-a-rich-comment-bot).

## GitHub Apps

GitHub Apps are certainly preferred and recommended from GitHub. From [GitHub's documentation](https://docs.github.com/en/developers/apps/getting-started-with-apps/about-apps), this fits our exact use case: 

> GitHub Apps are the official recommended way to integrate with GitHub because they offer much more granular permissions to access data. 
>
> GitHub Apps are first-class actors within GitHub. A GitHub App acts on its own behalf, taking actions via the API directly using its own identity, which means you don't need to maintain a bot or service account as a separate user.

### Caveats

- Each organization can only own up to 100 GitHub Apps
- You'll have to be an organization owner to create and install a GitHub app in an organization
- Each installation access token is only valid for a [maximum of 10 minutes](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps#authenticating-as-a-github-app)

## Creating a GitHub App

Creating a GitHub App is pretty straightforward! I'll defer to [GitHub's documentation for the details](https://docs.github.com/en/developers/apps/building-github-apps/creating-a-github-app), but here's a quick overview:
1. Navigate to the organization's settings page
2. Expand the "**Developer Settings**" section at the bottom of the page and navigate to **GitHub Apps**
3. Click "**New GitHub App**" in the upper right-hand corner
4. Start filling in the details! The name and Homepage URL doesn't matter much right now (but it does need a valid URL here) 
5. What does matter is the "**Webhook URL**" - if we want to _use_ this GitHub App in the next section, we'll need to grab the installation ID. The easiest way to do that is to start a new channel at [smee.io](https://smee.io/) and use the URL of the channel as the Webhook URL.
6. Grant it the **repository permissions**, **organization permissions**, **user permissions**, and what events to subscribe to - for the examples in this blog post, we'll grant read access to `repository / contents`, `repository / issues`, and `organization / members` - if you change this after the it's already been installed to an organization, you'll have review and re-approve the permission changes for the GitHub App
7.  After creation, you should see a "ping" entry in your smee.io channel - this is a confirmation that the app was created
8.  On the left-hand menu, you should now have a few options, one of those being "**Install App**" - click it, and install the app to the organization
9.  in your smee.io channel, you should have a new payload from the installation - expand the "installation" property to find your "**installation ID**" - this is the ID that you'll need to use in the next section
10. You'll also want to grab your **App ID** here (although note the App ID can also be found within GitHub)
11. Lastly, navigate back to your GitHub App's administration page and **[generate a private key for the app](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps#generating-a-private-key)** - download the file and grab the contents of the certificate by opening it in VSCode or if you are on macOS: `cat approveops-bot.2022-02-07.private-key | pbcopy`

![Installation and App ID from a payload in smee.io](installation-id.png ){: width="600" }{: .shadow }
_An example of an Installation ID and App ID from a payload in smee.io_

🎉 We now have everything we need to use the app in GitHub Actions! 🥳

## Using the GitHub App in a GitHub Actions workflow

There are a couple different actions to use such as:

- [navikt/github-app-token-generator](https://github.com/marketplace/actions/github-app-installation-access-token-generator)
- [peter-murray/workflow-application-token-action](https://github.com/marketplace/actions/workflow-application-token-action)
- [jnwng/github-app-installation-token-action](https://github.com/marketplace/actions/create-github-app-installation-token) (the one I am using below)

I like jnwng's version because it doesn't require the GitHub App to be installed on the repository that the action is running from whereas peter-murray's does. That's fine, but if the App has to be installed on every repository, we're not saving a ton with the app over the PAT (except that the GitHub App's token has a built-in expiration). navikt's version also works without being installed on the source repository, but it is a Docker-based action whereas jnwng's is a node-based action.

It's really quite simple now that you have the installation ID, the app ID, and the private key. The only prerequisite is to create a secret on the repository (or [organization](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization)) with the private key's contents. I named my secret `PRIVATE_KEY`.

### Scenario 1: Using a GitHub App to grant access to a single repository

A customer had a repository that nearly every Actions workflow was going to need to access at deploy-time. For the proof of concept, one of the admins on the team created a PAT and added it as an organizational secret. The problem is though that if the PAT is compromised, that PAT has access to _all the repositories in the organization_. 

If there's a centralized repository that every team needs to access, you can use the GitHub App to grant access to that repository and that repository alone. Note that you could also use [deploy keys](https://docs.github.com/en/developers/overview/managing-deploy-keys) for this, but that requires you to use the SSH protocol when cloning. We'll continue as if the GitHub App is the preferred way to go so that you can understand the process.

Here's the action code to generate and sign an installation access token for authenticating with GitHub:

{% raw %}
```yml
    steps:
      - uses: jnwng/github-app-installation-token-action@v2
        id: get_installation_token
        with: 
          appId: 170544
          installationId: 23052920
          privateKey: ${{ secrets.PRIVATE_KEY }}

      # clone using the `actions/checkout` step
      - name: Checkout
        uses: actions/checkout@v2.4.0
        with:
          repository: my-org/my-repo
          token: ${{ steps.get_installation_token.outputs.token }}
          path: my-repo

        # run the git clone ourselves
        name: Clone Repository
        run: | 
          mkdir my-repo-2 && cd my-repo-2
          git clone https://x-access-token:${{ steps.get_installation_token.outputs.token }}@github.com/my-org/my-repo.git
```
{: file='.github/workflows/test-permissions.yml'}
{% endraw %}

With the GitHub app installed on the `my-org/my-repo-2` repository, we have access to clone the repository even though it's not the repository that the workflow is running from. We can also use the token generated here as a Bearer token for GitHub API requests, assuming it has the access.

![Successful Git clone using a GitHub App](clone-from-app.png ){: width="500" }{: .shadow }
_Successful Git clone using a GitHup App_

🎉 Repo cloned! 🥳

### Scenario 2: Using a GitHub App as a rich comment bot

I'll often use the [peter-evans/create-or-update-comment](https://github.com/marketplace/actions/create-or-update-comment) action to create a comment on a pull request or issue. Typically, I'll just use the ${{ secrets.GITHUB_TOKEN }} for the `token` which comments as the `github-actions` bot, and it looks great!

{% raw %}
![GitHub Actions Comment Bot](github-actions-bot.png ){: .shadow }
_GitHub Actions Comment Bot using `${{ secrets.GITHUB_TOKEN}}`_
{% endraw %}

However, if you look closely, you might notice something: since the GitHub Token only has access to the repository, it can't create the proper `@team` mention in the comment. There is no hyperlink there. This might not be super important, but in my case the team was going to use their GitHub notifications to check if there were any issues that needed their attention, so this wasn't going to work.

If we use a PAT and a secret on the repository, we get the `@team` mention, but it looks like it came from the user who created the PAT:

![GitHub Actions Comment Bot](pat-bot.png ){: .shadow }
_Issues comment from GitHub Actions using a PAT_

Instead, we can use a GitHub App that with the `organization / members` permissions to create the comment and then we'll have the mention as well as not coming from a regular user:

![GitHub Actions Comment Bot](app-bot.png ){: .shadow }
_Issues comment from GitHub Actions using a GitHub App_

Here's the relevant action code:

{% raw %}
```yml
    steps:
      - uses: jnwng/github-app-installation-token-action@v2
        id: get_installation_token
        with: 
          appId: 170544
          installationId: 23052920
          privateKey: ${{ secrets.PRIVATE_KEY }}
          
      - if: ${{ steps.check-approval.outputs.approved == 'false' }}
        name: Create completed comment
        uses: peter-evans/create-or-update-comment@v1
        with:
          token: ${{ steps.get_installation_token.outputs.token }}
          issue-number: ${{ github.event.issue.number }}
          body: |
            Hey, @${{ github.event.comment.user.login }}!
            :cry:  No one approved your run yet! Have someone from the @joshjohanning-org/approver-team run `/approve` and then try your command again
            :no_entry_sign: :no_entry: Marking the workflow run as failed
```
{: file='.github/workflows/approveops.yml'}
{% endraw %}

🎉 Issue comment with team mentioning success! 🥳

## Summary

{% raw %}
In both scenarios, we use the [jnwng/github-app-installation-token-action](https://github.com/marketplace/actions/create-github-app-installation-token) action and its output of `${{ steps.get_installation_token.outputs.token }}` to obtain the installation access token. [GitHub has sample Ruby code](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps#authenticating-as-a-github-app) for creating a and signing the JWT, but the action is so much simpler!

Check out my next post on 'ApproveOps: Approvals in IssueOps' for more information on the action workflow I'm using in the second scenario.

Let me know what I've missed or if you have any other ideas!

{% endraw %}
