---
title: 'GitHub Packages: Migrate NuGet Packages to GitHub Packages'
author: Josh Johanning
date: 2022-11-23 13:30:00 -0500
description: Migrating NuGet packages stored in GitHub Packages from one instance to another
categories: [GitHub, Migrations]
tags: [GitHub, Scripts, GitHub Packages, gh cli, NuGet, Migrations]
mermaid: true
img_path: /assets/screenshots/2022-11-23-github-packages-migrate-nuget-packages
image:
  path: github-packages.png
  width: 100%
  height: 100%
  alt: NuGet packages in GitHub Packages
---

## Overview

To complete my NuGet Package migration series, I wanted to demonstrate how one could migrate NuGet packages to GitHub Package. The other system we are migrating from (whether it be Azure Artifacts, Artifactory, etc.) doesn't so much matter, as long as we are able to download/access the `.nupkg`{: .filepath} files on a system so that we can re-push to GitHub Packages.

> See my other NuGet package migration posts:
> - [Quickly Migrate NuGet Packages to a New Feed](/posts/nuget-pusher-script/)
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
{: .prompt-info }

## The script

The repo and docs can be found here: 
- **[https://github.com/joshjohanning/github-packages-migrate-nuget-packages-to-github-packages](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-to-github-packages)**

I decided to store the script in a separate GitHub repo than my [github-misc-scripts](/posts/github-misc-scripts/) repo to better facilitate any feedback/suggestions/improvements I might get - feel free to submit a PR if you can improve things 🚀!

## Update the mappings file

Your `csv`{: .filepath} file should look something like this:

```
Package,Target GitHub Repo
./mypkg.11.0.1.nupkg,my-org/my-repo
./mypkg.11.0.2.nupkg,my-org/my-repo

```
{: file='packages.csv'}

> Leave a trailing space at the end of the `csv`{: .filepath} file.
{: .prompt-info }

## Running the script

There are two scripts that need to be ran:

1. Generate the list of packages to migrate in the current directory and creating a mappings `csv`{: .filepath} file
  - We need to fill out the GitHub repository mapping for each package in the form of `owner/repo`
1. Migrate the packages to GitHub Packages

> Use this one-liner to copy all `.nupkg`{: .filepath} files to a directory before `./generate-nuget-package-mappings.sh`{: .filepath}: 
> ```bash
> find / -name "*.nupkg" -exec cp "{}" ./new-folder  \;
> ```
> {: .nolineno}
{: .prompt-tip }

### Generate the Mappings File

This finds all `.nupkg`{: .filepath} files in the current directory and generates a mappings `csv`{: .filepath} file.

```bash
./generate-nuget-package-mappings.sh \
  . \
  > <mappings-file>
```

Afterwards, you need to edit the `csv`{: .filepath} file to add the target GitHub repo reference, in the form of `owner/repo`.

> Leave a trailing space at the end of the `csv`{: .filepath} file.
{: .prompt-info }

### Migrate the Packages

This pushes the packages to the mapped GitHub repo:

```bash
./migrate-nuget-packages-to-github.sh \
  <mappings-file> \
  <pat> \
  <path-to-gpr>
```

### Complete Example

An example of this in practice:

```bash
# 1. generate mappings file
./generate-nuget-package-mappings.sh \
  . \
  > packages.csv

# 2. edit the mappings file to add the GitHub repo in the form of `owner/repo`

# 3. push packages
./migrate-nuget-packages-between-orgs.sh \
  packages.csv \
  ghp_xyz \
  /home/codespace/.dotnet/tools/gpr
```

## Notes

The script uses [gpr](https://github.com/jcansdale/gpr) to re-push the packages to the target org. It assumes that the target org's repo has the same name as the source.

I initially tried writing this with [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push), but that doesn't seem to work since the package's `<RepositoryUrl>` element would still be referencing the original repository. See error:

```
Pushing NUnit3.DotNetNew.Template_1.7.1.nupkg to 'https://nuget.pkg.github.com/joshjohanning-org-packages-migrated'...
  PUT https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/
warn : Source owner 'joshjohanning-org-packages-migrated' does not match repo owner 'joshjohanning-org-packages' in repository element.
  BadRequest https://nuget.pkg.github.com/joshjohanning-org-packages-migrated/ 180ms
error: Response status code does not indicate success: 400 (Bad Request).
```

[gpr](https://github.com/jcansdale/gpr) works because it rewrites the `<repository url="..." />` element in the `.nuspec`{: .filepath} file in the `.nupkg`{: .filepath} before pushing.

## Summary

Drop a comment here or an issue or PR on the [repo](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-to-github-packages) if you have any feedback or suggestions! Happy packaging! 📦
