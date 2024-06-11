---
title: 'GitHub Packages: Migrate Docker Containers Between GitHub Instances'
author: Josh Johanning
date: 2024-03-21 13:30:00 -0500
description: Migrating Docker containers stored in GitHub Packages / GitHub Container Registry between GitHub instances
categories: [GitHub, Packages]
tags: [GitHub, Scripts, GitHub Packages, gh cli, Docker, Containers, Migrations]
media_subpath: /assets/screenshots/2024-03-13-github-packages-migrate-docker-containers
image:
  path: docker-container-github-packages-light.png
  width: 100%
  height: 100%
  alt: Docker Containers in GitHub Packages
---

## Overview

I have been working with more customers who are migrating GitHub instances and want to be able to migrate GitHub Packages. There is not an easy lift-and-shift approach to migrate GitHub Packages between instances; each ecosystem is independent. To go along with [my other solutions](/categories/packages/), I also scripted out the Docker container migration. Take a look and let me know what you think!

> See my other GitHub Package --> GitHub Package migration posts:
>
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
> - [Migrate npm Packages Between GitHub Instances](/posts/github-packages-migrate-npm-packages/)
> - [Migrate Maven Packages Between GitHub Instances](/posts/github-packages-migrate-maven-packages/)
{: .prompt-info }

## The script

The script can be found in my [github-misc-scripts](/posts/github-misc-scripts/) repo here:

- **[https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-docker-containers-between-github-instances.sh](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-docker-containers-between-github-instances.sh)**

## Running the script

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed
2. Set the source GitHub PAT env var: `export GH_SOURCE_PAT=ghp_abc` (must have at least `read:packages`, `read:org` scope)
3. Set the target GitHub PAT env var: `export GH_TARGET_PAT=ghp_xyz` (must have at least `write:packages`, `read:org` scope)

### Usage

You can call the script via:

```bash
./migrate-docker-containers-between-github-instances.sh \
  <source-org> \
  <source-host> \
  <target-org> \
  <target-host> \
  <link-to-repository: true|false>
```

### Example

An example of this in practice:

```bash
export GH_SOURCE_PAT=ghp_abc
export GH_TARGET_PAT=ghp_xyz

./migrate-docker-containers-between-github-instances.sh \
  joshjohanning-org \
  github.com \
  joshjohanning-org-packages \
  github.com \
  true
```

## Notes

- If mapping to repositories with the 5th input parameter `<link-to-repository: true|false>`:
  - This script assumes that the target org's repo name is the same as the source
  - If the repo doesn't exist, the package will still import but won't be mapped to a repo
- Otherwise, images can be linked to repositories manually afterwords
- The packages API doesn't appear to pull all packages? I am unsure if it is because some of my packages are the same SHA behind the scenes and just published under different names, but double check that all Docker containers are migrated
- Image signatures are not migrated (see improvement ideas below)
- To clean up ALL local images, use this one-liner:
  ```bash
  rmi -f $(docker images -q)
  ```
  {: .nolineno}

## Improvement Ideas

- [ ] Use [ORAS CLI](https://oras.land/docs/commands/use_oras_cli) to migrate (might help with image signatures)
- [ ] Figure out why not all `container` type packages are not being returned by the API
- [ ] Map between repositories where the target repo is named differently than the source repo (likely this isn't needed since if repo doesn't exist, packages will still be pushed, the package just won't be linked to a repository)

## Summary

Drop a comment here or an issue or PR on my [github-misc-scripts repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-docker-containers-between-github-instances.sh) if you have any feedback or suggestions! Happy container migrating! 📦 🐳
