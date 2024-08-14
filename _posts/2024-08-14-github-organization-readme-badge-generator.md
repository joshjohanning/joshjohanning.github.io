---
title: 'GitHub Organization Readme Badge Generator'
author: Josh Johanning
date: 2024-08-14 3:30:00 -0500
description: A GitHub action to create markdown badges for your GitHub organization's README.md file
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions]
media_subpath: /assets/screenshots/2024-08-14-github-organization-readme-badge-generator
image:
  path: markdown-badges-light-header.png
  width: 100%
  height: 100%
  alt: Markdown badges in a GitHub organization's README
---

## Overview

In this post, I will show you how to add an Actions workflow to generate markdown badges to spruce up your organization READMEs. Badges are a great way to provide a quick visual representation of data; you might see them often on your favorite open source repository showing the code quality or number of stars. My action will generate badges for the following (right now):

- Number of repositories
- Number of pull requests open in the last 30 days
- Number of pull requests merged in the last 30 days

Here's an [example](https://github.com/joshjohanning-org#joshjohanning-org) of this in action:
![Markdown badges in a GitHub organization's README](markdown-badges-light.png){: .shadow }{: .light }
![Markdown badges in a GitHub organization's README](markdown-badges-dark.png){: .shadow }{: .dark }
_Markdown badges in a GitHub organization's README_

## What is an organization README?

[Organization READMEs](https://github.blog/changelog/2021-09-14-readmes-for-organization-profiles/) are a great way to showcase your organization to the world. Take [GitHub's organization README](https://github.com/github) as an example. There's a fun picture, a description of what's in the organization, how to contribute, and much more. [Member-only READMEs](https://github.blog/changelog/2022-04-20-organization-profile-updates-member-only-readmes-and-pinned-private-repositories/) are also a great extension of this functionality. Instead of providing information to the general public, you can provide information to your organization's members. For example, when I navigate to GitHub's organization page, I see a different README by default with internal information.

It's really easy to create a public and/or member-only organization README. For a public organization README, you just need to create a `.github` repository and add a `profile/README.md`{: .filepath} file. For an organization member-only README, create a `.github-private` repository and add a `profile/README.md`{: .filepath} file.

## The Workflow

My [action](https://github.com/joshjohanning/organization-readme-badge-generator) is fairly generic in the sense that all it does is generate the markdown badges. It's up to you to decide where to put them in your README. Here's an example of me overwriting some placeholder text by using my action and a bash script:

{% raw %}

```yml
name: update-organization-readme-badges

on:
  schedule:
    - cron: '0 7 * * *' # runs daily at 07:00
  workflow_dispatch:

permissions:
  contents: write

jobs:
  generate-badges:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}

      - name: organization-readme-badge-generator
        id: organization-readme-badge-generator
        uses: joshjohanning/organization-readme-badge-generator@v1
        with:
          organization: ${{ github.repository_owner }}
          token: ${{ steps.app-token.outputs.token }} # recommend to use a GitHub App and not a PAT
    
      - name: write to job summary
        run: |
          echo "${{ steps.organization-readme-badge-generator.outputs.badges }}" >> $GITHUB_STEP_SUMMARY

      - name: add to readme
        run: |
          readme=profile/README.md
          
          # get SHA256 before
          beforeHash=$(sha256sum $readme | awk '{ print $1 }')
          
          # Define start and end markers
          startMarker="<!-- start organization badges -->"
          endMarker="<!-- end organization badges -->"
          
          replacement="${{ steps.organization-readme-badge-generator.outputs.badges }}"
          
          # Escape special characters in the replacement text
          replacementEscaped=$(printf '%s\n' "$replacement" | perl -pe 's/([\\\/\$\(\)@])/\\$1/g')
          
          # Use perl to replace the text between the markers
          perl -i -pe "BEGIN{undef $/;} s/\Q$startMarker\E.*?\Q$endMarker\E/$startMarker\n$replacementEscaped\n$endMarker/smg" $readme
          # get SHA256 after
          afterHash=$(sha256sum $readme | awk '{ print $1 }')
          # Compare the hashes and commit if required
          if [ "$afterHash" = "$beforeHash" ]; then
            echo "The hashes are equal - exiting script"
            exit 0
          else
            git config --global user.name 'github-actions[bot]'
            git config --global user.email 'github-actions[bot]@users.noreply.github.com'
            git add $readme
            git commit -m "docs: update organization readme badges"
            git push
          fi
```
{: file='.github/workflows/update-organization-readme-badges.yml'}

{% endraw %}

> I have this [workflow](https://github.com/joshjohanning-org/.github/actions/workflows/update-organization-readme-badges.yml) running in my public `joshjohanning-org/.github` repository for reference.
{: .prompt-tip }

This workflow runs and generates the badges with my action. Afterwards, it finds placeholder text in the `profile/README.md`{: .filepath} file and updates the markdown badges. The workflow then commits the change and pushes it back to the repository.

Here's an example of the placeholder text in the `profile/README.md`{: .filepath} file that it's expecting (your badges would be dynamically inserted between the tags):

```md
# my-org-name

<!-- start organization badges -->

<!-- end organization badges -->
```
{: file='profile/README.md'}

## Summary

I like the ability to add quick little badges to my organization README to give the public / my members an idea of the status and activity stats for the organization. This action is a great way to do that.

I have this running as a scheduled workflow in both my `.github` and `.github-private` repositories.

I hope you find this useful! Let me know if there are other features or badges that I should add! For example, one of the things I was [envisioning](https://github.com/joshjohanning/organization-readme-badge-generator/issues/4) was number of GitHub Actions workflows ran / successful.. 🚀
