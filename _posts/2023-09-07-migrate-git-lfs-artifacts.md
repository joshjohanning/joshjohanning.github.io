---
title: 'Migrating GitHub Repositories: Tackling Files in Git LFS'
author: Josh Johanning
date: 2023-09-07 14:30:00 -0500
description: How to migrate Git LFS objects from one repository to another
categories: [GitHub, Migrations]
tags: [GitHub, Migrations, Git, Git LFS]
media_subpath: /assets/screenshots/2023-09-07-migrate-git-lfs-artifacts
image:
  path: git-lfs-pointer-light.png
  width: 100%
  height: 100%
  alt: Missing Git LFS file (pointer) in GitHub
---

## Overview

Git Large File Storage (LFS) replaces large files such as audio samples, videos, datasets, and graphics with text pointers inside Git, while storing the file contents on a remote server like GitHub. Git LFS works seamlessly, you hardly know it's there when interacting with Git.

You can migrate repositories to GitHub using [GitHub Enterprise Importer (GEI)](https://docs.github.com/en/migrations/using-github-enterprise-importer/understanding-github-enterprise-importer/about-github-enterprise-importer), however, since Git LFS is an extension to Git, they are [not migrated](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-between-github-products/about-migrations-between-github-products#data-that-is-not-migrated) automatically. There are special instructions that you need to follow if you are migrating Git repositories that contain LFS objects.

This post highlights two methods to migrate Git LFS objects from one repository to another:

- [Method 1: Using `git lfs push --all` to migrate all LFS objects](#method-1-git-lfs-push-all)
- [Method 2: Methodically pushing each LFS object](#method-2-methodically-pushing-each-object) (slower but more reliable for large repos and instances with higher network latency)

> See my other Git LFS posts:
> - [Migrating Large Files in Git to Git LFS](/posts/migrate-to-git-lfs/)
> - [Adding Files to Git LFS](/posts/add-files-to-git-lfs/)
{: .prompt-info }

## Prerequisites

1. Install `git lfs`
  - On macOS, you can install via `brew install git-lfs` 
  - On Windows, `git lfs` should be installed with [Git for Windows](https://gitforwindows.org/), but you can install an updated version via `choco install git-lfs`
2. Once installed, you need to run `git lfs install` to configure Git hooks to use LFS.

## Migrating a Repository with LFS Objects

No matter which method you choose, if you are using GEI to migrate the repository, you should run the migration first. This will migrate the Git repository, including the text pointers for the LFS objects, as well metadata such as pull requests and issues. After the migration is complete, you can then run one of the methods below to migrate the LFS objects.

> If you use GEI, I recommend waiting for the migration to fully complete before pushing LFS objects. However, if time is critical, you can proceed once the repository and code are visible in GitHub.
{: .prompt-tip }

### Method 1: git lfs push --all

If you are migrating a repository that contains LFS objects, you need to make sure that you migrate the LFS objects as well. If you don't, you will end up with a repository that contains text pointers to files that don't exist. This is because the LFS objects are stored in a separate location from the repository itself.

Instructions:

```bash
git clone --mirror <source-url> temp
cd temp
# git push --mirror <target-url> # or migrate with GEI
git lfs fetch --all
git lfs push <target-url> --all
```
{: .nolineno}

For Git repos with LFS, the two `git lfs` commands are key. This `git lfs fetch --all` command will download all the LFS objects from the source repository and store them in the target repository. The `git lfs push <target-url> --all` command will push all of the LFS objects to the target repository. After the push, the text pointers will be updated - including the same commit hash!

Here is an example of migrating a repository that contains LFS objects:
![Git LFS commands](git-lfs-commands.png){: .shadow }
_Running the migration commands to migrate a Git repository including Git LFS objects_

The repository has been migrated, including the LFS objects:
![Git LFS file in GitHub](git-lfs-light.png){: .shadow }{: .light }
![Git LFS file](git-lfs-dark.png){: .shadow }{: .dark }
_File stored in Git LFS file in GitHub_

### Method 2: methodically pushing each object

The method above worked well for me in my tests, even in tests with 1,000 LFS objects each 1mb in size. However, I have seen issues running these commands for some customers with larger repositories. Sometimes the `git lfs fetch` or `git lfs push` will hang or not appear to migrate all of the LFS objects. If you run into issues, you can run this script a more thorough (but slower) way to migrate the LFS objects. It works by looping through each LFS object and individually pushing it.

The alternative script to migrate the LFS objects to another repository:

```bash
SOURCEURL=https://github.com/source-org/source-repo.git
TARGETURL=https://github.com/target-org/target-repo.git
git clone --mirror $SOURCEURL temp
cd temp
# git push --mirror <target-url> # or migrate with GEI
git lfs fetch --all origin
for object_id in $(git lfs ls-files --all --long | awk '{print $1}'); do
    git lfs push --object-id $TARGETURL "$object_id"
done
```
{: .nolineno}

> Source for the [script](https://github.com/git-lfs/git-lfs/issues/4899#issuecomment-1688588756).
{: .prompt-info }

You could speed this up by using `xargs` and parallelization techniques. Something like this:

```bash
git lfs ls-files --all --long | awk '{print $1}' | xargs -P 4 -I {} git lfs push --object-id $TARGETURL {}
```
{: .nolineno}

> Parallelization is much harder to debug! I would recommend starting with the non-parallel version first and only use this sample if you had an extremely large number of LFS objects and are comfortable debugging any issues that might arise.
{: .prompt-warning }

## Other Tips

To speed up the migration process, you can prioritize the main branch since developers typically work with LFS files on the default branch most often. Instead of migrating all LFS objects at once, migrate the main branch first, then handle other branches as needed. You can do this by omitting the `--all` flag to only migrate LFS objects reachable from the current branch:

```bash
git clone --mirror <source-url> temp
cd temp
# git push --mirror <target-url> # or migrate with GEI
git lfs fetch
git lfs push <target-url>
# now use the `--all` option or for loop to migrate all other reachable LFS objects
```
{: .nolineno}

## Summary

Migrating Git repositories is simple. However, if the repository contains LFS objects, you need to make sure to run the `git lfs fetch` and `git lfs push` commands to migrate them along with the repository. If you don't, your Git LFS files will be missing.

Now that your Git repo, including LFS objects, is migrated, you can delete the temp folder (`cd .. && rm -rf temp`), update remotes on your local repo (`git remote remove origin` / `git remote add origin <url>`), and continue working as normal! 🚀
