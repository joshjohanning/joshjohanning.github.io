---
title: 'GitHub Packages: Migrate Maven Packages Between GitHub Instances'
author: Josh Johanning
date: 2023-09-25 14:30:00 -0500
description: Migrating Maven packages stored in GitHub Packages from one instance to another
categories: [GitHub, Packages]
tags: [GitHub, Scripts, GitHub Packages, gh cli, Maven, Migrations]
media_subpath: /assets/screenshots/2023-09-25-github-packages-migrate-maven-packages
image:
  path: maven-packages-light.png
  width: 100%
  height: 100%
  alt: Maven packages in GitHub Packages
---

## Overview

I have been working with more customers who are migrating GitHub instances and want to be able to migrate GitHub Packages. There is not an easy lift-and-shift approach to migrate GitHub Packages between instances; each ecosystem is independent. To go along with my [NuGet](/posts/github-packages-migrate-nuget-packages/) and [npm](/posts/github-packages-migrate-npm-packages/) solutions, I also scripted out the Maven package migration. Take a look and let me know what you think!

> See my other GitHub Package --> GitHub Package migration posts:
>
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
> - [Migrate npm Packages Between GitHub Instances](/posts/github-packages-migrate-npm-packages/)
> - [Migrate Docker containers Between GitHub Instances](/posts/github-packages-migrate-docker-containers/)
{: .prompt-info }

## The script

The script can be found in my [github-misc-scripts](/posts/github-misc-scripts/) repo here:

- **[https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-maven-packages-between-github-instances.sh](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-maven-packages-between-github-instances.sh)**

## Running the script

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed
2. Set the source GitHub PAT env var: `export GH_SOURCE_PAT=ghp_abc` (must have at least `read:packages`, `read:org` scope)
3. Set the target GitHub PAT env var: `export GH_TARGET_PAT=ghp_xyz` (must have at least `write:packages`, `read:org`, `repo` scope)

Notes:

- Until Maven supports the new GitHub Packages type, `mvnfeed` requires the target repo to exist 
- Link to [GitHub public roadmap item](https://github.com/github/roadmap/issues/578)
- This scripts creates the repo if it doesn't exist
- Otherwise, if the repo doesn't exist, receive "`example-1.0.5.jar`{: .filepath} was not found in the repository" error
- Currently [`mvnfeed-cli`](https://github.com/microsoft/mvnfeed-cli) only supports migrating `.jar`{: .filepath} and `.pom`{: .filepath} files (not `.war`{: .filepath})

### Usage

You can call the script via:

```bash
./migrate-maven-packages-between-github-instances.sh \
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

./migrate-maven-packages-between-github-instances.sh \
  joshjohanning-org \
  github.com \
  joshjohanning-org-packages \
  github.com
```

## Notes

- This script assumes that the target org's repo name is the same as the source
- If the repo doesn't exist, the package will still import but won't be mapped to a repo
- The script uses [`mvnfeed-cli`](https://github.com/microsoft/mvnfeed-cli) to migrate Maven packages to the target org
- Currently `mvnfeed-cli` only supports migrating `.jar`{: .filepath} + `.pom`{: .filepath} files (not `.war`{: .filepath})
- To clean up the working directory when done, run this one-liner: 
  ```bash
  rm -rf ./temp
  ```
  {: .nolineno}

## Improvement Ideas

* [x] Add a source folder input instead of relying on current directory (just using `./temp`{: .filepath})
* [ ] Map between repositories where the target repo is named differently than the source repo
* [x] Update script because of GitHub Packages GraphQL [deprecation](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/)
* [ ] Fork [`mvnfeed-cli`](https://github.com/microsoft/mvnfeed-cli) to support migrating `.war`{: .filepath} files ([issue](https://github.com/microsoft/mvnfeed-cli/issues/16)) - this only supports `.jar`{: .filepath} files currently

## Summary

Drop a comment here or an issue or PR on my [github-misc-scripts repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-maven-packages-between-github-instances.sh) if you have any feedback or suggestions! Happy packaging! 📦
