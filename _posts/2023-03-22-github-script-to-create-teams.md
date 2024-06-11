---
title: 'GitHub: Scripts to Mass Create and Delete Teams'
author: Josh Johanning
date: 2023-03-22 14:00:00 -0500
description: Create and delete GitHub teams programmatically from a CSV file
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, gh cli]
media_subpath: /assets/screenshots/2023-03-22-github-script-to-create-teams
image:
  path: create-team.png
  width: 100%
  height: 100%
  alt: Creating a team in GitHub
---

## Overview

If you've ever had to create several teams in GitHub, you know how painful it can be clicking around manually in the GitHub interface, especially if you have to create child teams. These scripts aim to simplify that by providing a list of teams that you want to create (or delete) in a CSV, and then looping through each team and creating (or deleting) it with the [`gh api`](https://cli.github.com/manual/gh_api) command of the [`gh cli`](https://cli.github.com/).

## The Scripts

These scripts are in my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo:

- [`create-teams-from-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/create-teams-from-list.sh)
- [`delete-teams-from-list.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/delete-teams-from-list.sh)

## Using the Scripts

### Prerequisites

- You need to make sure to have the [`gh cli`](https://cli.github.com/) installed and authorized (`gh auth login`).
- Add the `admin:org` scope:
  ```bash
  gh auth refresh -h github.com -s admin:org
  ```
  {: .nolineno}

### Example Input File

An example of input file that can be used for both creating/deleting teams:

```sh
test11-team
test22-team
test11-team/test11111-team
test11-team/test11111-team/textxxx-team
test33-team

```
{: file='teams.csv'}

> Note: Ensure that the input file has a trailing new line
{: .prompt-info }

### Create Teams

1. Prepare a list of teams that you want to create and place in a CSV file, one per line, with the last line empty.
    - Child teams should have a slash in the name, e.g., `test1-team/test1-1-team`
    - Build out the parent structure in the input file before creating the child teams; e.g. have the `test1-team` come before `test1-team/test1-1-team` in the file
2. From there, run the script by passing in the `teams.csv`{: .filepath} file and org name:

```bash
./create-teams-from-list.sh teams.csv my-org
```
{: .nolineno}

> Note: A parent team should exist before creating a child team, or at least should come first in the input file
{: .prompt-info }

### Delete Teams

1. Prepare a list of teams that you want to create and place in a CSV file, one per line, with the last line empty.
    - Child teams should have a slash in the name, e.g., `test1-team/test1-1-team`
    - `!!! Important !!!` Note that if a team has child teams, all of the child teams will be deleted as well
2. From there, run the script by passing in the `teams.csv`{: .filepath} file and org name:

```bash
./delete-teams-from-list.sh teams.csv my-org
```
{: .nolineno}

> Note: All child teams belonging to the parent team will be deleted as well
{: .prompt-danger }

## Summary

I originally built the `create-teams-from-list.sh`{: .filepath} script from a request from a blog reader, and then later built the delete teams script to simplify testing because if I was creating teams in an automated fashion, you know I had no interest in deleting the test teams manually 😀. 

Please feel free to PR or share any improvements to these scripts! The terminal logging certainly isn't the cleanest, but the scripts work and provide a decent amount of information that is relevant to the request. 

Enjoy! 🚀
