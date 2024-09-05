---
title: 'GitHub Packages: Migrate npm Packages Between GitHub Instances'
author: Josh Johanning
date: 2023-09-08 15:00:00 -0500
description: Migrating npm packages stored in GitHub Packages from one instance to another
categories: [GitHub, Packages]
tags: [GitHub, Scripts, GitHub Packages, gh cli, npm, Migrations]
media_subpath: /assets/screenshots/2023-09-08-github-packages-migrate-npm-packages
image:
  path: npm-packages-light.png
  width: 100%
  height: 100%
  alt: npm packages in GitHub Packages
---

## Overview

I have been working with more customers who are migrating GitHub instances and want to be able to migrate GitHub Packages. There is not an easy lift-and-shift approach to migrate GitHub Packages between instances; each ecosystem is independent. To go along with my [NuGet](/posts/github-packages-migrate-nuget-packages/) solution, I also scripted out the npm package migration. Take a look and let me know what you think!

> See my other GitHub Package --> GitHub Package migration posts:
>
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
> - [Migrate Maven Packages Between GitHub Instances](/posts/github-packages-migrate-maven-packages/)
> - [Migrate Docker containers Between GitHub Instances](/posts/github-packages-migrate-docker-containers/)
{: .prompt-info }

## The script

The script can be found in my [github-misc-scripts](/posts/github-misc-scripts/) repo here:

- **[https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh)**

## Running the script

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed
2. Set the source GitHub PAT env var: `export GH_SOURCE_PAT=ghp_abc` (must have at least `read:packages`, `read:org` scope)
3. Set the target GitHub PAT env var: `export GH_TARGET_PAT=ghp_xyz` (must have at least `write:packages`, `read:org` scope)

Notes:

- This script assumes that the target org's repo name is the same as the source
- If the repo doesn't exist, the package will still import but won't be mapped to a repo

### Usage

You can call the script via:

```bash
./migrate-npm-packages-between-github-instances.sh \
  <source-org> \
  <source-host> \
  <target-org> \
  <target-host> \
  | tee output.log
```

> The `| tee output.log` will print the output to the console and also save it to a file. You can refer back to the log file later and search for errors.
{: .prompt-tip }

### Example

An example of this in practice:

```bash
export GH_SOURCE_PAT=ghp_abc
export GH_TARGET_PAT=ghp_xyz

./migrate-npm-packages-between-github-instances.sh \
  joshjohanning-org \
  github.com \
  joshjohanning-org-packages \
  github.com \
  | tee output.log
```

## Notes

- This script assumes that the target org's repo name is the same as the source
- If the repo doesn't exist, the package will still import but won't be mapped to a repo
- This script uses RegEx to find/replace source org with target org in the package's `package.json`{: .filepath} (see the [GitHub docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry#publishing-a-package) for more info)
- To clean up the working directory when done, run this one-liner: 
  ```bash
  rm -rf ./temp
  ```
  {: .nolineno}

## Improvement Ideas

* [x] Add a source folder input instead of relying on current directory (just using `./temp`{: .filepath})
* [ ] Map between repositories where the target repo is named differently than the source repo (likely this isn't needed since if repo doesn't exist, packages will still be pushed, the package just won't be linked to a repository)
* [x] Update script because of GitHub Packages GraphQL [deprecation](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/)

## Summary

Drop a comment here or an issue or PR on my [github-misc-scripts repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh) if you have any feedback or suggestions! Happy packaging! 📦
