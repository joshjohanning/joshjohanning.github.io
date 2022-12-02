---
title: 'GitHub Packages: Migrate NuGet Packages Between GitHub Instances'
author: Josh Johanning
date: 2022-11-23 13:30:00 -0500
description: Migrating NuGet packages stored in GitHub Packages from one instance to another
categories: [GitHub, Migrations]
tags: [GitHub, Scripts, GitHub Packages, gh cli, NuGet, Migrations]
img_path: /assets/screenshots/2022-11-23-github-packages-migrate-nuget-packages
image:
  path: github-packages.png
  width: 100%
  height: 100%
  alt: NuGet packages in GitHub Packages
---

## Overview

I recently had a customer ask me how they could migrate their NuGet packages from one GitHub instance to another (e.g.: from GitHub Enterprise Server to GitHub Enterprise Cloud). I wasn't aware of any tooling that did this, so I decided to write my own.

> See my other NuGet package migration posts:
> - [Quickly Migrate NuGet Packages to a New Feed](/posts/nuget-pusher-script/)
> - [Migrate NuGet Packages to GitHub Packages](/posts/github-packages-migrate-nuget-packages-to-github-packages/)
{: .prompt-info }

## The script

The repo and docs can be found here: 
- **[https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances)**

I decided to store the script in a separate GitHub repo than my [github-misc-scripts](/posts/github-misc-scripts/) repo to better facilitate any feedback/suggestions/improvements I might get - feel free to submit a PR if you can improve things 🚀!

## Running the script

You can call the script via:

```bash
./migrate-nuget-packages-between-orgs.sh \
  <source-org> 
  <source-host> \
  <target-org> \
  <target-pat> \
  <path-to-gpr>
```

An example of this in practice:

```bash
./migrate-nuget-packages-between-orgs.sh \
  joshjohanning-org-packages \
  github.com \
  joshjohanning-org-packages-migrated \
  ghp_xyz \
  /home/codespace/.dotnet/tools/gpr
```

## Notes

The script uses [`gpr`](https://github.com/jcansdale/gpr) to re-push the packages to the target org. It assumes that the target org's repo has the same name as the source.

I initially tried writing this with [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push), but that doesn't seem to work since the package's `<RepositoryUrl>` element would still be referencing the original repository. See error:

```
Pushing NUnit3.DotNetNew.Template_1.7.1.nupkg to 'https://nuget.pkg.github.com/joshjohanning-org-packages-migrated'...
  PUT https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/
warn : Source owner 'joshjohanning-org-packages-migrated' does not match repo owner 'joshjohanning-org-packages' in repository element.
  BadRequest https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/ 180ms
error: Response status code does not indicate success: 400 (Bad Request).
```

[gpr](https://github.com/jcansdale/gpr) works because it rewrites the `<repository url="..." />` element in the `.nuspec`{: .filepath} file in the `.nupkg`{: .filepath} before pushing.

Also, in the script, I had to delete `_rels/.rels`{: .filepath} and `[Content_Types].xml`{: .filepath} because there was somehow two copies of each file in the package and it causes gpr to fail when extracting/zipping the package to re-push.

To clean up the working directory when done, run this one-liner: 

```bash
rm *.nupkg *.zip
```
{: .nolineno}

## Summary

Drop a comment here or an issue or PR on the [repo](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances) if you have any feedback or suggestions! Happy packaging! 📦
