---
title: 'Demystifying GitHub Apps: Using GitHub Apps to Replace Service Accounts'
author: Josh Johanning
date: 2022-02-07 20:00:00 -0600
description: Creating no-code GitHub Apps to install to an organization to replace having to create service accounts or a user PAT for authorization in GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, GitHub Apps, GitHub Issues]
img_path: /assets/screenshots/2022-02-07-github-apps
image:
  path: github-apps.png
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

And don't worry - you don't need any programming experience to create a GitHub App!

## GitHub Apps

GitHub Apps are certainly preferred and recommended from GitHub. From [GitHub's documentation](https://docs.github.com/en/developers/apps/getting-started-with-apps/about-apps), this fits our exact use case: 

> GitHub Apps are the official recommended way to integrate with GitHub because they offer much more granular permissions to access data. 
>
> GitHub Apps are first-class actors within GitHub. A GitHub App acts on its own behalf, taking actions via the API directly using its own identity, which means you don't need to maintain a bot or service account as a separate user.

GitHub Apps also have a [higher API rate limiting threshold](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting) than requests from user accounts. [Installed in an Enterprise](https://github.blog/changelog/2020-07-27-increase-to-api-limits-in-github-enterprise-subscriptions/), GitHub Apps can have up to [15,000 requests per hour](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/rate-limits-for-github-apps#installation-access-tokens-on-github-enterprise-cloud) whereas user-created personal access tokens have a limit of 5,000 requests per hour. For non-Enterprise organizations, there is a [formula](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/rate-limits-for-github-apps#installation-access-tokens-on-githubcom) that is used to calculate the rate limit based on the number of users in the organization, but it's still higher than the 5,000 requests per hour that a user-created personal access token has.

When authenticating with the `GITHUB_TOKEN` in a GitHub Actions workflows in an Enterprise organization, you also have access to the [15,000 requests per repository per hour](https://docs.github.com/en/rest/overview/resources-in-the-rest-api?apiVersion=2022-11-28#rate-limits-for-requests-from-github-actions). Outside of an Enterprise organization, you're limited to 1,000 requests per hour. However, the `GITHUB_TOKEN` in the Actions workflow expires when the workflow is complete and can only access resources inside of the repository calling the workflow. It cannot access resources outside of the repository, such as other repositories or organizational resources.

### Caveats

- Each organization can only own up to [100 GitHub Apps](https://docs.github.com/en/developers/apps/getting-started-with-apps/about-apps#about-github-apps)
- You'll have to be an [organization owner](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners) to create an app that's owned by the organization, and only organization owners can install a GitHub App
    - Non-organization owners can create a personal app and request it to be installed by the org, but it will be owned by the user and not the organization
- Each installation access token is only valid for a [maximum of one hour](https://docs.github.com/en/rest/reference/apps#create-an-installation-access-token-for-an-app)
- GitHub Apps [can't be used to authenticate to GitHub Packages](https://github.com/orgs/community/discussions/26920) (have to use personal access tokens for this)
- If you make a [GitHub App private](https://docs.github.com/en/enterprise-cloud@latest/developers/apps/managing-github-apps/making-a-github-app-public-or-private), other apps can't see it / interact with it
  - Example: You can't use GitHub App A to modify a branch protection policy to let GitHub App B to bypass the policy if GitHub App B is private - GitHub App B would have to be a Public app

## Creating a GitHub App

Creating a GitHub App is pretty straightforward! I'll defer to [GitHub's documentation for the details](https://docs.github.com/en/developers/apps/building-github-apps/creating-a-github-app), but here's a quick overview:

1. Navigate to the organization's settings page
2. Expand the "**Developer Settings**" section at the bottom of the page and navigate to **GitHub Apps**
3. Click "**New GitHub App**" in the upper right-hand corner
4. Start filling in the details! 
    - The name and Homepage URL doesn't matter much right now (but it does need a valid URL here) 
    - *(Optional - for use with webhooks)* - If you want the GitHub app to send webhooks based on events, and want an easy way to inspect the payload that is being sent, you can use something like [smee.io](https://smee.io/) to create a channel and use the [URL of the channel](https://smee.io/cm3xXguj5Ds9hs8) as the Webhook URL
6. Grant it the **repository permissions**, **organization permissions**, **user permissions**, and what events to subscribe to - for the examples in this blog post, we'll grant **read-only** access to `repository / contents`, **read & write** access to `repository / issues`, and **read-only** access to `organization / members` - if you change this after the it's already been installed to an organization, you'll have review and re-approve the permission changes for the GitHub App
8. After creation, you should see the **App ID** - we will need this later on
    - *(Optional - for use with webhooks)* - After creation, you should see a "ping" entry in your [smee.io channel](https://smee.io/cm3xXguj5Ds9hs8) - this is a confirmation that the app was created - the **App ID** is available in this payload as well
9.  On the left-hand menu, you should now have a few options, one of those being "**Install App**" - click it, and install the app to the organization, or alternatively, install the app to selected repositories that you want it to be able to access
10. After installing the application, pay attention to the URL in the browser (see screenshot below) - the number at the end of the URL is the installation ID, which we'll need later on 
    - *(Optional - for use with webhooks)* - In your [smee.io channel](https://smee.io/cm3xXguj5Ds9hs8), you should have a new payload from the installation - expand the "installation" property to find your "**installation ID**" - this is the ID that you'll need to use in the next section
11. Lastly, navigate back to your GitHub App's administration page, scroll down, and **[generate a private key for the app](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps#generating-a-private-key)** - download the file and grab the contents of the certificate by opening it in VSCode or if you are on macOS: `cat approveops-bot.2022-02-07.private-key | pbcopy`

🎉 With the **App ID**, **Installation ID**, and **Private Key**, we now have everything we need to use the app in Actions! 🥳

![Installation ID after installing GitHub App](installation-id-github.png ){: .shadow }
_An example of an Installation ID in the address bar after installing a GitHub app_

![Installation and App ID from a payload in smee.io](installation-id.png ){: width="600" }{: .shadow }
_An example of an Installation ID and App ID from a payload in smee.io_

## Using the GitHub App in a GitHub Actions workflow

There are a couple different actions to use such as:

- [navikt/github-app-token-generator](https://github.com/marketplace/actions/github-app-installation-access-token-generator)
- [peter-murray/workflow-application-token-action](https://github.com/marketplace/actions/workflow-application-token-action)
- [jnwng/github-app-installation-token-action](https://github.com/marketplace/actions/create-github-app-installation-token)
- [tibdex/github-app-token](https://github.com/marketplace/actions/github-app-token) (the one I am using below)

I like **navikt's**, **jnwng's**, and **tibdex's** versions because it doesn't require the GitHub App to be installed on the repository that the action is running from whereas **peter-murray's** does. That's fine, but if the App must be installed on every repository, we're not saving a ton with the app over the PAT (except that the GitHub App's token has a built-in expiration). **navikt's** version is a Docker-based action. I'm typically going to prefer a node-based action if given the preference since typically a Docker action takes a little bit longer to initiate and requires one additional component to be installed if using self-hosted runners. **jnwng's** and **tibdex's** actions are node-based actions, and both certainly work. I slightly prefer **tibdex's** because you can either pass in an `installation_id` or not, depending on if the app is installed on the repository or not.

It's really quite simple now that you have the installation ID, the app ID, and the private key. The only prerequisite is to create a secret on the repository (or [organization](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization)) with the private key's contents. I named my secret `PRIVATE_KEY`.

### Scenario 1: Using a GitHub App to grant access to a single repository

A customer had a repository that nearly every Actions workflow was going to need to access at deploy-time. For the proof of concept, one of the admins on the team created a PAT and added it as an organizational secret. The problem is though that if the PAT is compromised, that PAT has access to _all the repositories in the organization_. 

If there's a centralized repository that every team needs to access, you can use the GitHub App to grant access to that repository and that repository alone. Note that you could also use [deploy keys](https://docs.github.com/en/developers/overview/managing-deploy-keys) for this, but that requires you to use the SSH protocol when cloning. We'll continue as if the GitHub App is the preferred way to go so that you can understand the process.

Here's the action code to generate and sign an installation access token for authenticating with GitHub:

{% raw %}
```yml
    steps:
      - uses: tibdex/github-app-token@v1
        id: get_installation_token
        with: 
          app_id: 170544
          # installation_id not needed IF the app is installed on this current repo
          installation_id: 29881931
          private_key: ${{ secrets.PRIVATE_KEY }}

      # example 1a - cloning repo - clone using the `actions/checkout` step
      - name: Checkout
        uses: actions/checkout@v2.4.0
        with:
          repository: my-org/my-repo
          token: ${{ steps.get_installation_token.outputs.token }}
          path: my-repo

      # example 1b - cloning repo - using git clone command
      - name: Clone Repository
        run: | 
          mkdir my-repo-2 && cd my-repo-2
          git clone https://x-access-token:${{ steps.get_installation_token.outputs.token }}@github.com/my-org/my-repo.git

      # example 2a - api - call an api using curl
      - name: Get Repo (curl)
        run: | 
          curl \
            -H "Authorization: Bearer ${{ steps.get_installation_token.outputs.token }}" \
            https://api.github.com/repos/joshjohanning-org/composite-caller-1

      # example 2b - api - call an api using the GitHub CLI
      - name: Get Repo (gh api)
        env:
          GH_TOKEN: ${{ steps.get_installation_token.outputs.token }}
        run: | 
          gh api /repos/joshjohanning-org/composite-caller-1
```
{: file='.github/workflows/test-permissions.yml'}
{% endraw %}

With the GitHub app installed on the `my-org/my-repo-2` repository and passing in the installation ID to the action, we have access to clone the repository even though it's not the repository that the workflow is running from. We can also use the token generated here as a Bearer token for GitHub API requests, assuming it has the access.

![Successful Git clone using a GitHub App](clone-from-app.png ){: width="500" }{: .shadow }
_Successful Git clone using a GitHup App_

🎉 Repo cloned! 🥳

### Scenario 2: Using a GitHub App as a rich comment bot

I'll often use the [peter-evans/create-or-update-comment](https://github.com/marketplace/actions/create-or-update-comment) action to create a comment on a pull request or issue. Typically, I'll just use the ${{ secrets.GITHUB_TOKEN }} for the `token` which comments as the `github-actions` bot, and it looks great!

{% raw %}
![GitHub Actions Comment Bot](github-actions-bot.png ){: .shadow }
_GitHub Actions Comment Bot using GitHub Token from the Action run_
{% endraw %}

However, if you look closely, you might notice something: since the GitHub Token only has access to the repository, it can't create the proper `@team` mention in the comment. There is no hyperlink there. This might not be super important, but in my case the team was going to use their GitHub notifications to check if there were any issues that needed their attention, so this wasn't going to work.

If we use a PAT and a secret on the repository, we get the `@team` mention, but it looks like it came from the user who created the PAT:

![GitHub Actions Comment Bot](pat-bot.png ){: .shadow }
_Issues comment from GitHub Actions using a PAT_

Instead, we can use a GitHub App that with **read-only** permissions on `Organization / Members` and **read & write** on `Repository / Issues` to create the comment and then we'll have the mention as well as not coming from a regular user:

![GitHub Actions Comment Bot](app-bot.png ){: .shadow }
_Issues comment from GitHub Actions using a GitHub App_

Here's the relevant action code:

{% raw %}
```yml
    steps:
      - uses: tibdex/github-app-token@v1
        id: get_installation_token
        with: 
          app_id: 170544
          private_key: ${{ secrets.PRIVATE_KEY }}
          
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

You'll notice that we didn't have to pass in an installation ID this time to the action. This is because the GitHub App is installed on the repository, and the action can therefore lookup the installation ID dynamically to get the token.

🎉 Issue comment with team mentioning success! 🥳

## Summary

When I first learned about GitHub Apps, I was like, "This is cool, but I'm not going to be writing an app and creating code just for authentication, that's too much work, I'll just use a PAT." However, as you just saw, we created a GitHub App and used it for authentication without tying it to any code. 

{% raw %}
In both scenarios, we use the [tibdex/github-app-token](https://github.com/marketplace/actions/github-app-token) action and the installation access token that is an output parameter: `${{ steps.get_installation_token.outputs.token }}`. We use this token to make authenticated requests to the API or as the password in Git clones. Alternatively, [GitHub has sample Ruby code](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps#authenticating-as-a-github-app) for creating a and signing the JWT and retrieving an installation ID, but the action is so much simpler!

Check out my next post, [ApproveOps: Approvals in IssueOps](/posts/github-approveops), for more information on the action workflow I'm using in the second scenario.

Happy App-ing!

{% endraw %}
