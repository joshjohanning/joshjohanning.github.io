---
title: 'Migrating Git Repos with LFS Artifacts'
author: Josh Johanning
date: 2023-09-07 14:30:00 -0500
description: How to migrate Git LFS artifacts from one repository to another
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

Git Large File Storage (LFS) replaces large files such as audio samples, videos, datasets, and graphics with text pointers inside Git, while storing the file contents on a remote server like GitHub. Git LFS works seamlessly, you hardly know it's there when interacting with Git. However, there are special instructions that you need to follow if you are migrating Git repositories that contain LFS artifacts.

> See my other Git LFS posts:
> - [Migrating Large Files in Git to Git LFS](/posts/migrate-to-git-lfs/)
> - [Adding Files to Git LFS](/posts/add-files-to-git-lfs/)
{: .prompt-info }

## Prerequisites

1. Install `git lfs`
  - On macOS, you can install via `brew install git-lfs` 
  - On Windows, `git lfs` should be installed with [Git for Windows](https://gitforwindows.org/), but you can install an updated version via `choco install git-lfs`
2. Once installed, you need to run `git lfs install` to configure Git hooks to use LFS.

## Migrating a Repository with LFS Artifacts

If you are migrating a repository that contains LFS artifacts, you need to make sure that you migrate the LFS artifacts as well. If you don't, you will end up with a repository that contains text pointers to files that don't exist. This is because the LFS artifacts are stored in a separate location from the repository itself.

Instructions:

```bash
git clone --mirror <source-url> temp
cd temp
git push --mirror <target-url>
git lfs fetch --all
git lfs push <target-url> --all
```

For Git repos with LFS, the two `git lfs` commands are key. This `git lfs fetch --all` command will download all the LFS artifacts from the source repository and store them in the target repository. The `git lfs push <target-url> --all` command will push all of the LFS artifacts to the target repository. After the push, the text pointers will be updated - including the same commit hash!

Here is an example of migrating a repository that contains LFS artifacts:
![Git LFS commands](git-lfs-commands.png){: .shadow }
_Running the migration commands to migrate a Git repository including Git LFS artifacts_

The repository has been migrated, including the LFS artifacts:
![Git LFS file in GitHub](git-lfs-light.png){: .shadow }{: .light }
![Git LFS file](git-lfs-dark.png){: .shadow }{: .dark }
_File stored in Git LFS file in GitHub_

## Alternative Method

The method above worked well for me in my tests, even in tests with 1,000 LFS artifacts each 1mb in size. However, I have seen issues running these commands for some customers with larger repositories. Sometimes the `git lfs fetch` or `git lfs push` will hang or not appear to migrate all of the LFS artifacts. If you run into issues, you can run this script to more thoroughly (but slower) way to migrate the LFS artifacts. It works by looping through each LFS object and pushing it individually pushing it.

The alternative script to migrate the LFS artifacts to another repository:

```bash
git clone --mirror <source-url> temp
cd temp
for object_id in $(git lfs ls-files --all --long | awk '{print $1}'); do
    git lfs push --object-id remote "$object_id"
done
```

> Source for the [script](https://github.com/git-lfs/git-lfs/issues/4899#issuecomment-1688588756).
{: .prompt-info }

## Summary

Migrating Git repositories is simple. However, if the repository contains LFS artifacts, you need to make sure to run the `git lfs fetch` and `git lfs push` commands to migrate them along with the repository. If you don't, your Git LFS files will be missing.

Now that your Git repo, including LFS artifacts, is migrated, you can delete the temp folder, update remotes on your local repo, and continue working as normal! 🚀
