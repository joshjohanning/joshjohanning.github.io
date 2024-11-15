---
title: 'GitHub: Script to Add Users to a Project'
author: Josh Johanning
date: 2024-02-03 7:00:00 -0600
description: Add users to a GitHub ProjectV2 programmatically
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, GitHub Projects]
media_subpath: /assets/screenshots/2024-02-03-github-script-to-add-users-to-project
image:
  path: github-project.png
  width: 100%
  height: 100%
  alt: An organization Project in GitHub
---

## Overview

We're often adding new users to [Projects (ProjectsV2) in GitHub](https://docs.github.com/en/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects), and previously, it was all manual work in the UI. I wanted to automate this, but found the [`updateProjectV2Collaborators`](https://docs.github.com/en/graphql/reference/mutations#updateprojectv2collaborators) mutation a little tricky to figure out. After some trial and error, I was able to get it working and wanted to share the script I created to add users to a GitHub Project.

## The Script

The script is in my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo:

- [`add-user-to-project.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/add-user-to-project.sh)

## Using the Script

### Usage

```bash
Usage: ./add-user-to-project.sh <org> <repo> <project-number> <user> <role>
```

Example:

```bash
./add-user-to-project.sh joshjohanning-org my-repo 1234 joshjohanning ADMIN
```

Notes:

- You need the `project` scope, e.g.: `gh auth login -s project`
- Role options: `ADMIN`, `WRITER`, `READER`, `NONE`

## Summary

Hopefully this speeds up your adding of users to an Organization Project in GitHub! 🚀

> See my other ProjectsV2 scripts:
>
> - [`get-projects-in-organization.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-projects-in-organization.sh)
> - [`get-projects-added-to-repository.sh`{: .filepath}](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-projects-added-to-repository.sh)
{: .prompt-info }
