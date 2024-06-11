---
title: 'Adding Files to Git LFS'
author: Josh Johanning
date: 2023-09-07 16:00:00 -0500
description: Tracking new files in Git LFS
categories: [GitHub, Commits]
tags: [GitHub, Git, Git LFS, Commits]
media_subpath: /assets/screenshots/2023-09-07-add-files-to-git-lfs
image:
  path: ./../2023-09-07-migrate-to-git-lfs/git-lfs-light.png
  width: 100%
  height: 100%
  alt: Git LFS file in GitHub
---

## Overview

Git LFS can be very helpful to store large files along with your code in Git repositories. Git LFS artifacts are treated like any other file in Git, but the contents of the file are stored in a separate location. This allows you to store large files in Git without bloating the size of your Git repository. Tracking new files in Git LFS is easy, but there are a few steps you need to follow when using LFS for the first time.

> See my other Git LFS posts:
> - [Migrating Git Repos with LFS Artifacts](/posts/migrate-git-lfs-artifacts/)
> - [Migrating Large Files in Git to Git LFS](/posts/migrate-to-git-lfs/)
{: .prompt-info }

## Prerequisites

1. Install `git lfs`
  - On macOS, you can install via `brew install git-lfs` 
  - On Windows, `git lfs` should be installed with [Git for Windows](https://gitforwindows.org/), but you can install an updated version via `choco install git-lfs`
2. Once installed, you need to run `git lfs install` to configure Git hooks to use LFS.

## Adding a New File to Git LFS

Here is how to add/track a new file to Git LFS:

```bash
git lfs install # if git config is not currently configured for LFS
git lfs track "*.jpg" "*.png" # track all jpg and png files (recursively)
git add .gitattributes **/*.jpg **/*.png # add .gitattributes and LFS artifacts (recursively)
git lfs ls-files # optional: ensure your LFS files are tracked before committing
git commit -m "Tracking and adding LFS artifacts"
git push
```

The `git lfs install` command is important otherwise the files will not be pushed to LFS, even with a `.gitattributes`{: .filepath} file. If you don't want to install globally, you can run `git lfs install --local` to only configure the LFS Git hooks for the current repository. You can always run `git lfs uninstall` to unconfigure. 

Once you track the file(s) you want to add to LFS with the `git lfs track` command, you simply have to stage with `git add`, `commit`, and `push` as normal. Now, every additional `.jpg`{: .filepath} and `.png`{: .filepath} added to the repository will automatically be tracked by LFS.

Of course, you can track specific files without using wildcards, as well as only tracking a specific directory. For example:

```bash
git lfs track "file.exe" # track a single file
git lfs track "bin/*" # track a directory
```

Here is an example of adding files to Git LFS:
![Adding files to Git LFS](git-lfs-add.png){: .shadow }
_Adding files to Git LFS_

After pushing to GitHub, the GitHub UI shows the file is stored with Git LFS:
![Git LFS file in GitHub](./../2023-09-07-migrate-git-lfs-artifacts/git-lfs-light.png){: .shadow }{: .light }
![Git LFS file in GitHub](./../2023-09-07-migrate-git-lfs-artifacts/git-lfs-dark.png){: .shadow }{: .dark }
_File stored in Git LFS file in GitHub_

## Summary

Like anything, there is a ton of documentation on the internet but sometimes it is a bit scattered. I created this post to help with those who are new to Git LFS and want to start tracking files. 

There are some useful tips I've picked up along the way, like using `git lfs ls-files` before committing to verify files are being tracked, and also avoiding a common pitfall of not running `git lfs install` before tracking files.

I hope this helps! ✨
