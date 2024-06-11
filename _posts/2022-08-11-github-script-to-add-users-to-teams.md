---
title: 'GitHub: Script to Mass Add Users to a Team'
author: Josh Johanning
date: 2022-08-11 16:00:00 -0500
description: Add users to a GitHub org team programmatically from a CSV file
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, gh cli]
media_subpath: /assets/screenshots/2022-08-11-github-script-to-add-users-to-teams
image:
  path: github-team.png
  width: 100%
  height: 100%
  alt: Adding a user to a team in GitHub
---

## Overview

If you've ever had to add several users to a team in a GitHub organization, you know it can be a pain as it's one user at a time and multiple clicks per add. This script aims to simplify that by adding users to a GitHub org team programmatically from a CSV file.

## The Script

This script is in my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo:

- [`add-users-to-team-from-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/add-users-to-team-from-list.sh)

## Using the Script

Prerequisite: You need to make sure to have the [`gh cli`](https://cli.github.com/) installed and authorized (`gh auth login`).

1. Create a `users.csv`{: .filepath} with the list of users to add to the team, one per line, and leave a trailing empty line/whitespace at the end of the file. The file should look like: 
    ```
    user1
    user2

    ```
    {: file='users.csv'}
2. From there, it's pretty simple - run the script, passing in the `users.csv`{: .filepath} file, org name, and team name:
    ```bash
    ./add-users-to-team-from-list.sh users.csv <org> <team>
    ```
    {: .nolineno}

## Summary

Hopefully this saves you some time in the UI when adding multiple users to a team!
