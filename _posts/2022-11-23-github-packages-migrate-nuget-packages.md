---
title: 'GitHub Packages: Migrate NuGet Packages Between GitHub Instances'
author: Josh Johanning
date: 2022-11-23 13:30:00 -0500
description: Migrating NuGet packages stored in GitHub Packages from one instance to another
categories: [GitHub, Packages]
tags: [GitHub, Scripts, GitHub Packages, gh cli, NuGet, Migrations]
media_subpath: /assets/screenshots/2022-11-23-github-packages-migrate-nuget-packages
image:
  path: github-packages.png
  width: 100%
  height: 100%
  alt: NuGet packages in GitHub Packages
---

## Overview

I recently had a customer ask me how they could migrate their NuGet packages from one GitHub instance to another (e.g.: from GitHub Enterprise Server to GitHub Enterprise Cloud). I wasn't aware of any tooling that did this, so I decided to write my own.

> See my other NuGet package migration posts:
>
> - [Quickly Migrate NuGet Packages to a New Feed](/posts/nuget-pusher-script/)
> - [Migrate NuGet Packages to GitHub Packages](/posts/github-packages-migrate-nuget-packages-to-github-packages/)
{: .prompt-info }
> See my other GitHub Package --> GitHub Package migration posts:
>
> - [Migrate npm Packages Between GitHub Instances](/posts/github-packages-migrate-npm-packages/)
> - [Migrate Maven Packages Between GitHub Instances](/posts/github-packages-migrate-maven-packages/)
> - [Migrate Docker containers Between GitHub Instances](/posts/github-packages-migrate-docker-containers/)
{: .prompt-info }

## The script

The repo and docs can be found here:

- **[https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances)**

I decided to store the script in a separate GitHub repo than my [github-misc-scripts](/posts/github-misc-scripts/) repo to better facilitate any feedback/suggestions/improvements I might get - feel free to submit a PR if you can improve things 🚀!

## Running the script

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed
2. Set the source GitHub PAT env var: `export GH_SOURCE_PAT=ghp_abc` (must have at least `read:packages`, `read:org` scope)
3. Set the target GitHub PAT env var: `export GH_TARGET_PAT=ghp_xyz` (must have at least `write:packages`, `read:org` scope)

Notes:

- This script installs [gpr](https://github.com/jcansdale/gpr) locally to the `./temp/tools`{: .filepath} directory
- This script assumes that the target org's repo name is the same as the source
- If the repo doesn't exist, the package will still import but won't be mapped to a repo

### Usage

You can call the script via:

```bash
./migrate-nuget-packages-between-orgs.sh \
  <source-org> 
  <source-host> \
  <target-org>
```

### Example

An example of this in practice:

```bash
export GH_SOURCE_PAT=ghp_abc
export GH_TARGET_PAT=ghp_xyz

./migrate-nuget-packages-between-orgs.sh \
  joshjohanning-org-packages \
  github.com \
  joshjohanning-org-packages-migrated
```

## Notes

- The script assumes that the target org's repo has the same name as the source - it's not required, but the package won't be mapped to a repo if the target repo doesn't exist
- The script uses [`gpr`](https://github.com/jcansdale/gpr) to re-push the packages to the target org
  + I initially tried writing this with [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push), but that doesn't seem to work since the package's `<RepositoryUrl>` element would still be referencing the original repository. See error:

    ```
    dotnet nuget push \
      -s github \
      -k ghp_pat \
      NUnit3.DotNetNew.Template_1.7.1.nupkg

    Pushing NUnit3.DotNetNew.Template_1.7.1.nupkg to 'https://nuget.pkg.github.com/joshjohanning-org-packages-migrated'...
      PUT https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/
    warn : Source owner 'joshjohanning-org-packages-migrated' does not match repo owner 'joshjohanning-org-packages' in repository element.
      BadRequest https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/ 180ms
    error: Response status code does not indicate success: 400 (Bad Request).
    ```
    {: file='\'dotnet nuget push\' error'}

  + [`gpr`](https://github.com/jcansdale/gpr) works because it rewrites the `<repository url="..." />` element in the `.nuspec`{: .filepath} file in the `.nupkg`{: .filepath} before pushing
  +  Update Dec 2022: Now that NuGet Packages has supports granular permissions and organization sharing ([GitHub's roadmap item](https://github.com/github/roadmap/issues/589)), [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push) should work - but using [`gpr`](https://github.com/jcansdale/gpr) for mapping convenience
    - [`gpr`](https://github.com/jcansdale/gpr) still might be preferred since you would have to tie the NuGet package to the repository manually post-migration
    - If attempting to use [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push), you will have to add the feed first using this command:
      ```bash
      dotnet nuget add source \
        --username my-github-username \
        --password "ghp_pat" \
        --store-password-in-clear-text \
        --name github \
        "https://nuget.pkg.github.com/OWNER/index.json"
      ```
      {: .nolineno}
- Also, in the script, I had to delete `_rels/.rels`{: .filepath} and `[Content_Types].xml`{: .filepath} because there was somehow two copies of each file in the package and it causes gpr to fail when extracting/zipping the package to re-push
- To clean up the working directory when done, run this one-liner: 
  ```bash
  rm -rf ./temp
  ```
  {: .nolineno}

## Improvement Ideas

* [x] Add a source folder input instead of relying on current directory (just using `./temp`{: .filepath})
* [ ] Map between repositories where the target repo is named differently than the source repo (likely this isn't needed since if repo doesn't exist, packages will still be pushed, the package just won't be linked to a repository)
* [x] Dynamically determine out where [`gpr`](https://github.com/jcansdale/gpr) is installed instead of passing in a parameter (right now we are passing the `gpr` path in as a parameter explicitly because sometimes `gpr` is aliased to `git pull --rebase`) (installing `gpr` locally to the `./temp/tools`{: .filepath} directory)
* [x] Update script because of GitHub Packages GraphQL [deprecation](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/)

## Summary

Drop a comment here or an issue or PR on the [repo](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances) if you have any feedback or suggestions! Happy packaging! 📦
