---
title: 'GitHub: Script to Add dependabot.yml to a List of Repos'
author: Josh Johanning
date: 2024-01-08 12:30:00 -0600
description: Add the dependabot.yml file programmatically to a list of GitHub repositories
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, Octokit, Dependabot]
media_subpath: /assets/screenshots/2024-01-08-github-script-to-add-dependabot-file
image:
  path: dependabot-enable.png
  width: 100%
  height: 100%
  alt: Dependabot configuration for a repository
---

## Overview

I've been exploring how to enable [Dependabot Version Updates](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/about-dependabot-version-updates) across a large set of repositories. Unlike Dependabot Security Alerts or Dependabot Security Updates, Dependabot Version Updates relies on a file existing in the repository: `.github/dependabot.yml`{: .filepath}. Confusingly, there is an "**Enable**" button when configuring Dependabot Version Updates, but that only is a link to be able to create and commit the file manually into the repository.

What I wanted to do was to be able to add the `.github/dependabot.yml`{: .filepath} file to a list of repositories programmatically. I was able to do this using the [Octokit](https://octokit.github.io/rest.js/v18) library and the [GitHub API](https://docs.github.com/en/rest/reference/repos#create-or-update-file-contents). Thankfully, adding or updating a single file in a repository is easy; adding multiple files as part of the same commit is slightly harder with the GitHub API but still doable (have to use the [Git trees API](https://docs.github.com/en/rest/git/trees?apiVersion=2022-11-28); example [here](https://github.com/joshjohanning-org/commit-sign-app/blob/f010f5d8f86655b55166142bf322d5d1b6945b1a/.github/workflows/commit-sign.yml#L72-L121)!).

## The Scripts

These scripts are in my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo:

- [`generate-repositories-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/generate-repositories-list.sh)
- [`add-dependabot-file-to-repositories.js`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/add-dependabot-file-to-repositories.js)

## Using the Scripts

### Prerequisites

- Node.js installed
- Environment variable named `GITHUB_TOKEN` with a GitHub PAT that has `repo` scope (for committing)
- Dependencies installed via `npm i octokit fs`
- Update the `gitUsername`, `gitEmail`, and `overwrite` const at the top of the script accordingly
  - If you want to use a GitHub App to be the commit author:
    - `gitUsername` value: 
      - Should be the GitHub App name with `[bot]` appended
      - Example: `josh-issueops-bot[bot]`
    - `gitEmail` value: 
      - Return the user ID with: `gh api '/users/josh-issueops-bot[bot]' --jq .id`
      - The email will then be: `149130343+josh-issueops-bot[bot]@users.noreply.github.com`

### Usage

1. Prepare a list of repositories that you want to add the `dependabot.yml`{: .filepath} file to and place in a file, one per line
    - You can use the [`generate-repositories-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/generate-repositories-list.sh) script to generate a list of repos in a GitHub org, and then modify accordingly: 
      ```bash
      ./generate-repositories-list.sh joshjohanning-org > repos.txt
      ```
      {: .nolineno}
    - Or, create your own input file with the list of repos you want to add the `dependabot.yml`{: .filepath}, one per line: 
      ```sh
      org/repo1
      org/repo2
      ```
      {: file='repos.txt'}
      {: .nolineno}
2. From there, it's pretty simple - run the script, passing in the `repos.txt`{: .filepath} file:
    ```bash
    export GITHUB_TOKEN=ghp_abc
    npm i octokit fs papaparse
    node ./add-dependabot-file-to-repositories.js ./repos.txt ./dependabot.yml
    ```
    {: .nolineno}

## Future Enhancements

- [ ] Add an option to create a pull request instead of committing directly to the default branch

## Edit: More feature-rich alternative

[@ruzickap pointed out](https://github.com/joshjohanning/joshjohanning.github.io/issues/33#issuecomment-1896339564) that we can also use [`multi-gitter`](https://github.com/lindell/multi-gitter/) to run a script against a set of repositories. This tool already creates pull requests for us, as well as includes a command to track the `status` of the PRs, to `merge` the PRs, and to `close` the PRs ✨.

See my [follow-up comment](https://github.com/joshjohanning/joshjohanning.github.io/issues/33#issuecomment-1951356259) for an example on using [`multi-gitter`](https://github.com/lindell/multi-gitter/) to copy in a `dependabot.yml`{: .filepath} file if it doesn't exist.

There's also a more complex example in my [comment](https://github.com/joshjohanning/joshjohanning.github.io/issues/33#issuecomment-1951356259) that creates a `dependabot.yml`{: .filepath} file if it doesn't exist, but if it does exist, only check to see if there is a `package-ecosystem: github-actions` section and if not, add it.

## Summary

This will speed up the process of adding the `dependabot.yml`{: .filepath} file to a list of repositories. This can be helpful to make sure teams are keeping up to date on their dependencies. I use this all the time especially to keep up with marketplace and internal GitHub Actions that my repositories are referencing. Feel free to let me know if I'm missing anything and/or submit a PR to enhance this further! 🚀

> See my other Dependabot Version Updates posts:
> - [Configure GitHub Dependabot to Keep Actions Up to Date](/posts/github-dependabot-for-actions/)
> - [Configuring Dependabot for Reusable Workflows in GitHub](/posts/dependabot-reusable-workflows/)
{: .prompt-info }
