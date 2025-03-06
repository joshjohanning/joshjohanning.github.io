---
title: 'GitHub Apps: Configuring the Git Email for Commits'
author: Josh Johanning
date: 2024-11-22 12:00:00 -0600
description: A guide on how to set up the proper Git email address for commits made by your GitHub App to ensure proper commit attribution
categories: [GitHub, Apps]
tags: [GitHub, GitHub Actions, GitHub Apps, GitHub Issues, Git]
media_subpath: /assets/screenshots/2024-11-22-github-apps-commit-email
image:
  path: github-app-commit-light.png
  width: 100%
  height: 100%
  alt: A commit from a GitHub app in a GitHub repository with the commit being attributed to the app
---

## Overview

I recently was working with a customer who had just discovered [GitHub Apps](/posts/github-apps/) as a replacement to a service account user created in GitHub. Using a GitHub App has a few benefits:

1. You don't have to manage a separate user account, including username, password, MFA settings, etc.
2. A GitHub App doesn't consume a GitHub license
3. A GitHub App has a higher rate-limit
4. A GitHub App's token that's generated expires after a maximum of 1 hour, so it's more secure than a user's long-lived token

The customer was using this [Action](https://github.com/stefanzweifel/git-auto-commit-action) to auto commit changes made in the workflow back to the repository. When using a GitHub user account, they were simply using the email address (or in this case, the noreply email address) associated with the GitHub account. With a GitHub app, we have to configure the emails in a slightly different format that's not easily documented or readily available. So this is where this post comes in!

You can technically commit using any email address when committing to GitHub (assuming you don't have verified commits required). However, if the committing email address isn't associated directly with a GitHub user's (or app's) email address, the profile picture/author's icon will just be a gray GitHub logo. You also can't filter commits by that user/app in the UI. So, if you are committing with an app, you might as well make the author look like the app in GitHub. 🤖

> If you are new to GitHub Apps, check out my [other post on getting started](/posts/github-apps/)! It's really much easier than you think. 🚀
{: .prompt-info }

## Email Format

If you use the [API to commit as a GitHub App](https://github.com/orgs/community/discussions/50055), you will see the following commit email address format used:

```text
149130343+josh-issueops-bot[bot]@users.noreply.github.com
```

Where does that `149130343` come from? You might think it's the GitHub App ID, made readily available in the app's management page. But, sadly, you would be incorrect. 🤦‍♂️

The ID field here is actually the *user ID* of the GitHub App.

We can retrieve this in one of two ways:

1. Open up the REST API endpoint in your browser and grab the ID field. The format of the URL will be:

    ```text
    https://api.github.com/users/josh-issueops-bot[bot]
    ```

2. Use the GitHub CLI and `--jq` to grab the ID field:

    ```bash
    gh api '/users/josh-issueops-bot[bot]' --jq .id
    ```

This returns the ID of the user, which is `149130343`. Once you have that, you can plug it into the email address format and you're good to go! 🚀

> Email address format:
>
> - `USERID`+`APP-NAME`[bot]@users.noreply.github.com
{: .prompt-tip }

## Committing via Git Command Line in Actions

Here's a simple example of how you could commit changes back to the repository using the `git` command line in a GitHub Actions workflow and have the commit attributed to the GitHub App:

```yml
jobs:
  generate-changelog:
    runs-on: ubuntu-latest
    permissions:
      contents: write # this allows you to write back to repo
    steps:
    - uses: actions/checkout@v4
    # - do stuff -
    - name: push to git repo
      run: |
        git config --global user.name 'josh-issueops-bot[bot]'
        git config --global user.email '149130343+josh-issueops-bot[bot]@users.noreply.github.com'
        git add .
        git commit -m "ci: updating changelog"
        git push
```
{: file='.github/workflows/commit-with-github-app.yml'}

{% raw %}
This still uses the `${{ github.token }}` (the Actions user) to authenticate, but the commit / commit author is being attributed to the app.

If you wanted to use the GitHub App's token for authentication, you could do something like this instead:

```yml
jobs:
  generate-changelog:
    runs-on: ubuntu-latest
    permissions:
      contents: none # technically no permissions required since we are using the App's auth token here 
    steps:
    - uses: actions/create-github-app-token@v1
      id: app-token
      with:
        app-id: ${{ vars.APP_ID }}
        private-key: ${{ secrets.APP_PRIVATE_KEY }}
        owner: ${{ github.repository_owner }}
    - uses: actions/checkout@v4
      with:
        token: ${{ steps.app-token.outputs.token }} # using the app's token to establish auth
        repository: ${{ github.repository }} # default is to checkout repo of the workflow
    - name: push to git repo
      run: |
        git config --global user.name 'josh-issueops-bot[bot]'
        git config --global user.email '149130343+josh-issueops-bot[bot]@users.noreply.github.com'
        git add .
        git commit -m "ci: updating changelog"
        git push
```
{: file='.github/workflows/commit-with-github-app.yml'}

{% endraw %}

## Using the Git Auto Commit Action

Since I mentioned [this action](https://github.com/stefanzweifel/git-auto-commit-action) earlier, here's what an example workflow would look like using it:

```yml
jobs:
  generate-changelog:
    runs-on: ubuntu-latest
    permissions:
      contents: write # this allows you to write back to repo
    steps:
    - uses: actions/checkout@v4
    # - do stuff -
    - uses: stefanzweifel/git-auto-commit-action@v5
      with:
        commit_user_name: josh-issueops-bot[bot]
        commit_user_email: 149130343+josh-issueops-bot[bot]@users.noreply.github.com
        commit_message: "ci: updating changelog"
        # use this input if you don't want it to default the author to the user triggering the workflow
        commit_author: josh-issueops-bot[bot] <149130343+josh-issueops-bot[bot]@users.noreply.github.com>
```
{: file='.github/workflows/commit-with-github-app.yml'}

## Signing Commits from GitHub App

If you use the examples above using the `Git` command line, your commits will not be signed. If you want the commits from the GitHub App to be signed, we have to commit via the API by [creating a tree](https://docs.github.com/en/rest/git/trees?apiVersion=2022-11-28#create-a-tree), [creating a commit](https://docs.github.com/en/rest/git/commits?apiVersion=2022-11-28#create-a-commit), and [updating a reference](https://docs.github.com/en/rest/git/refs?apiVersion=2022-11-28#update-a-reference) to tie the tree to the commit.

For a detailed example, see this helpful Community Discussions post from [@loujr](https://github.com/loujr/):

- [How to Use Commit Signing with GitHub Apps](https://github.com/orgs/community/discussions/50055)

I also have an end-to-end example of using a GitHub App to sign a commit in GitHub Actions for reference:

- [commit-sign.yml - GitHub App commit signing example in Actions](https://github.com/joshjohanning-org/commit-sign-app/blob/main/.github/workflows/commit-sign.yml#L80-L153)

## Summary

In summary, if you are using a GitHub App to commit changes back to the repository, you will need to use the email address format of `USERID`+`APP-NAME`[bot]@users.noreply.github.com. This will allow the commit to be attributed to the GitHub App, and the author's icon to be the App's icon (or the org's icon if there is no app icon!). 🤖
