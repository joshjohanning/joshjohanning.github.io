---
title: 'GitHub: Script to Mass Delete Repos'
author: Josh Johanning
date: 2022-10-06 11:00:00 -0500
description: Delete GitHub repositories programmatically from a CSV file
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, gh cli]
media_subpath: /assets/screenshots/2022-10-06-github-script-to-delete-repos
image:
  path: delete-repo.png
  width: 100%
  height: 100%
  alt: Deleting a repo in GitHub
---

## Overview

If you've ever had to delete several repositories in GitHub, you know it can be a pain as you have to copy/paste the name of each repo in the verification prompt. This script aims to simplify that by providing a list of repositories that you want to delete in a CSV, and then looping through each repo and deleting it with the [`gh cli`](https://cli.github.com/).

## The Scripts

These scripts are in my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo:

- [`generate-repositories-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/generate-repositories-list.sh)
- [`delete-repositories-from-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/delete-repositories-from-list.sh)

## Using the Scripts

### Prerequisites

- You need to make sure to have the [`gh cli`](https://cli.github.com/) installed and authorized (`gh auth login`).
- Add the `delete_repo` scope:
  ```bash
  gh auth refresh -h github.com -s delete_repo
  ```
  {: .nolineno}

### Usage

1. Prepare a list of repositories that you want to delete and place in a CSV file, one per line, with the last line empty.
    - You can use the [`generate-repositories-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/generate-repositories-list.sh) script to generate a list of repos in a GitHub org, and then modify accordingly: 
      ```bash
      ./generate-repositories-list.sh joshjohanning-org > repos.csv
      ```
      {: .nolineno}
    - Or, create your own CSV file with the list of repos you want to delete, one per line, and leave a trailing empty line/whitespace at the end of the file. The file should look like: 
      ```sh
      org/repo1
      org/repo2

      ```
      {: file='repos.csv'}
2. From there, it's pretty simple - run the script, passing in the `repos.csv`{: .filepath} file:
    ```bash
    ./delete-repositories-from-list.sh repos.csv
    ```
    {: .nolineno}

## Summary

If you accidentally delete a repository that you didn't mean to, the good news is that you have [90 days to recover it](https://docs.github.com/en/repositories/creating-and-managing-repositories/restoring-a-deleted-repository) in most cases (with some caveats for forks). 

Hopefully this script saves you some time in the UI copy/pasting repo names when you want to do some spring cleaning!
