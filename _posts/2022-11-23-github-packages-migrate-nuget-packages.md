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

### Prerequisites

1. [`gh cli`](https://cli.github.com) installed and logged in to be able to access the source GitHub instance:
   ```bash
   gh auth login
   ```
   {: .nolineno}
2. Auth to read packages from the source GitHub instance with `gh`:
   ```bash
   gh auth refresh -h github.com -s read:packages # update -h with source github host
   ```
   {: .nolineno}
3. [`gpr`](https://github.com/jcansdale/gpr) installed:
   ```bash
   dotnet tool install gpr -g
   ```
   {: .nolineno}
4. Can use this one-liner to find the absolute path for [`gpr`](https://github.com/jcansdale/gpr) for the `<path-to-gpr>` parameter: 
   ```bash
   find / -wholename "*tools/gpr" 2> /dev/null
   ```
   {: .nolineno}
5. `<target-pat>` must have `write:packages` scope
6. This assumes that the target org's repo name is the same as the source.

We are passing [`gpr`](https://github.com/jcansdale/gpr) in as a parameter explicitly because sometimes [`gpr`](https://github.com/jcansdale/gpr) is aliased to `git pull --rebase` and that's not what we want here.

### Usage

You can call the script via:

```bash
./migrate-nuget-packages-between-orgs.sh \
  <source-org> 
  <source-host> \
  <target-org> \
  <target-pat> \
  <path-to-gpr>
```

### Example

An example of this in practice:

```bash
./migrate-nuget-packages-between-orgs.sh \
  joshjohanning-org-packages \
  github.com \
  joshjohanning-org-packages-migrated \
  ghp_xyz \
  /home/codespace/.dotnet/tools/gpr
```

> Use this one-liner to copy all `.nupkg`{: .filepath} files to the current working directory before running the script: 
> ```bash
> find / -name "*.nupkg" -exec cp "{}" ./  \;
> ```
> {: .nolineno}
{: .prompt-tip }

## Notes

- The script uses [`gpr`](https://github.com/jcansdale/gpr) to re-push the packages to the target org. It assumes that the target org's repo has the same name as the source.
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
  + There is an item on [GitHub's roadmap](https://github.com/github/roadmap/issues/589) to support pushing packages directly to an organization; this should allow [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push) to work instead of [`gpr`](https://github.com/jcansdale/gpr)
    - [`gpr`](https://github.com/jcansdale/gpr) still might be preferred since you would have to tie the NuGet package to the repository manually post-migration
    - For [`dotnet nuget push`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-nuget-push) to work, you will have to add the feed first using this command:
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
  rm *.nupkg *.zip
  ```
  {: .nolineno}

## Improvement Ideas

* [ ] Add a source folder input instead of relying on current directory
* [ ] Map between repositories where the target repo is named differently than the source repo
* [ ] Dynamically determine out where [`gpr`](https://github.com/jcansdale/gpr) is installed instead of passing in a parameter (right now we are passing the `gpr` path in as a parameter explicitly because sometimes `gpr` is aliased to `git pull --rebase`)

## Summary

Drop a comment here or an issue or PR on the [repo](https://github.com/joshjohanning/github-packages-migrate-nuget-packages-between-github-instances) if you have any feedback or suggestions! Happy packaging! 📦
