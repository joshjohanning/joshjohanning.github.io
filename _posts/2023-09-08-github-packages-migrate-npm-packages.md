---
title: 'GitHub Packages: Migrate npm Packages Between GitHub Instances'
author: Josh Johanning
date: 2023-09-08 15:00:00 -0500
description: Migrating npm packages stored in GitHub Packages from one instance to another
categories: [GitHub, Migrations]
tags: [GitHub, Scripts, GitHub Packages, gh cli, npm, Migrations]
img_path: /assets/screenshots/2023-09-08-github-packages-migrate-npm-packages
image:
  path: npm-packages-light.png
  width: 100%
  height: 100%
  alt: npm packages in GitHub Packages
---

## Overview

I have been working with more customers who are migrating GitHub instances and want to be able to migrate GitHub Packages. There is not an easy lift-and-shift approach to migrate GitHub Packages between instances; each ecosystem is independent. To go along with my [NuGet](/posts/github-packages-migrate-nuget-packages/) solution, I also scripted out npm package migration. Take a look and let me know what you think!

> See my other GitHub Package --> GitHub Package migration post:
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
{: .prompt-info }

## The script

The script can be found in my [github-misc-scripts](/posts/github-misc-scripts/) repo here: 
- **[https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh)**

## Running the script

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed and logged in to be able to access the source GitHub instance:
   ```bash
   gh auth login
   ```
   {: .nolineno}
2. `<source-pat>` must have `read:packages` scope (defined as environment variable `GH_SOURCE_PAT`)
3. `<target-pat>` must have `write:packages` scope (defined as environment variable `GH_SOURCE_PAT`)
4. This assumes that the target org's repo name is the same as the source

### Usage

You can call the script via:

```bash
./migrate-npm-packages-between-github-instances.sh \
  <source-org> \
  <source-host> \
  <target-org> \
  <target-host>
```

### Example

An example of this in practice:

```bash
export GH_SOURCE_PAT=ghp_abc
export GH_TARGET_PAT=ghp_xyz

./migrate-npm-packages-between-github-instances.sh \
  joshjohanning-org \
  github.com \
  joshjohanning-org-packages \
  github.com
```

## Notes

- The script assumes that the target org's repo has the same name as the source
- This script uses RegEx to find/replace source org with target org in the package's `package.json`{: .filepath} (see the [GitHub docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry#publishing-a-package) for more info)
- To clean up the working directory when done, run this one-liner: 
  ```bash
  rm -rf ./temp
  ```
  {: .nolineno}

## Improvement Ideas

* [ ] Add a source folder input instead of relying on current directory
* [ ] Map between repositories where the target repo is named differently than the source repo
* [x] Update script because of GitHub Packages GraphQL [deprecation](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/)

## Summary

Drop a comment here or an issue or PR on the [repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-npm-packages-between-github-instances.sh) if you have any feedback or suggestions! Happy packaging! 📦
