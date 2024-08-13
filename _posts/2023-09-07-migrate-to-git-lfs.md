---
title: 'Migrating Large Files in Git to Git LFS'
author: Josh Johanning
date: 2023-09-07 15:00:00 -0500
description: How to migrate Git LFS artifacts from one repository to another
categories: [GitHub, Migrations]
tags: [GitHub, Migrations, Git, Git LFS]
media_subpath: /assets/screenshots/2023-09-07-migrate-to-git-lfs
image:
  path: git-lfs-light.png
  width: 100%
  height: 100%
  alt: Git LFS file in GitHub
---

## Overview

Git LFS can be very helpful to store large files along with your code in Git repositories. Git LFS artifacts are treated like any other file in Git, but the contents of the file are stored in a separate location. This allows you to store large files in Git without bloating the size of your Git repository.

If you've ever tried migrating or pushing a repository that contains a file larger than 100mb, you might see something like this:

> remote: error: File <file> is 116.10 MB; this exceeds GitHub's file size limit of 100.00 MB
> remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.

This post will show you how to migrate large blobs in Git to Git LFS.

> See my other Git LFS posts:
> - [Migrating Git Repos with LFS Artifacts](/posts/migrate-git-lfs-artifacts/)
> - [Adding Files to Git LFS](/posts/add-files-to-git-lfs/)
{: .prompt-info }

## Prerequisites

1. Install `git lfs`
  - On macOS, you can install via `brew install git-lfs` 
  - On Windows, `git lfs` should be installed with [Git for Windows](https://gitforwindows.org/), but you can install an updated version via `choco install git-lfs`
2. Once installed, you need to run `git lfs install` to configure Git hooks to use LFS.

## Migrating a File to Git LFS

If you have a file in the Git history is larger than 100mb, you can migrate it to Git LFS. This will allow you to store the file in Git LFS and keep the file in your Git repository.

> Following these instructions will **rewrite Git history** (new commit hashes)! Make sure to have a COMPLETE backup of the repository before proceeding (`git clone --mirror <url>`)
{: .prompt-danger }

> For those using **commit signing**: I have noticed that since the `git lfs migrate import` rewrites history, the commits that it recreates aren't signed.
{: .prompt-warning }

An example script to migrate  `.exe`{: .filepath} and `.iso`{: .filepath} files to Git LFS:

```bash
cd ~/Repos/my-git-repo-with-large-files
git lfs migrate import --include="*.exe, *.iso" --everything
git lfs ls-files # list LFS files
git push --all --force # force push all branches to remote
```

This will migrate all files with the extensions `.exe`{: .filepath} and `.iso`{: .filepath} to Git LFS. The `--everything` option will run the migration in all local and remote Git refs (branches, tags). Additionally, this will also create and commit a `.gitattributes`{: .filepath} file that will tell Git to store all files with the extensions `.exe`{: .filepath} and `.iso`{: .filepath} in Git LFS.

The nice thing about the `--everything` option is that it also works for files that have been committed to history and subsequently deleted, so you don't have to use any additional tools to rewrite history.

If you only want to migrate and rewrite a single branch, you can use `--include-ref refs/heads/main` to specify a specific ref. You can add multiple `--include-ref` options to rewrite and migrate multiple refs.

Read more on the `git lfs migrate` command in the [docs](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-migrate.adoc#options).

Here is an example of these commands in action. Note after the `git push --all --force` command, we can see that LFS objects are being uploaded:
![Git LFS commands](git-lfs-migrate-commands.png){: .shadow }
_Running the migration commands to migrate large/binary files to Git LFS_

Afterwards, the GitHub UI shows the file is now stored with Git LFS:
![Git LFS file in GitHub](./../2023-09-07-migrate-git-lfs-artifacts/git-lfs-light.png){: .shadow }{: .light }
![Git LFS file in GitHub](./../2023-09-07-migrate-git-lfs-artifacts/git-lfs-dark.png){: .shadow }{: .dark }
_File stored in Git LFS file in GitHub_

## Summary

Since [Git LFS 2.2.0](https://github.blog/2017-06-27-git-lfs-2-2-0-released/), the `git lfs migrate` command makes it easy to migrate large files in a repository to Git LFS. Before, you would have had to use `git filter-repo` or [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) to 1) remove the file from the Git history, then 2) add the files to be tracked by LFS by running `git lfs track`, 3) staging the `.gitattributes`{: .filepath} file and pushing.

Now, with the simple `git lfs migrate` command, this takes care of most of the dirty work for us! 🎉
