---
title: 'Migrating Repos to GitHub'
author: Josh Johanning
date: 2022-09-30 12:00:00 -0500
description: The different options available for migrating repos to GitHub or any other Git provider
categories: [GitHub, Migrations]
tags: [Azure DevOps, GitHub, TFVC, SVN, Git, Migrations]
media_subpath: /assets/screenshots/2022-09-30-migrating-repos-to-github
image:
  path: import-repo-github.gif
  width: 66%
  height: 66%
  alt: Importing a repository to GitHub using the GitHub Importer
---

## Overview

There are several options for migrating repos to GitHub, depending on what Source Control Management (SCM) tool you are coming from, and what you want to migrate. Just the Git repo with all of its history? Because it's Git to Git, it's [easy](#option-2-git-clone-mirror) Only care about the latest changes and want to start fresh? Perhaps even easier, pull or get latest, create a new Git repo, add in the changes, and push. If it's not Git, you must determine if you want to use a specialized tool to migrate or start fresh. This post will cover all of the different options, when you should use them, and any known gotchas.

Often, simply migrating your entire Git repository (with history) is sufficient and the most recommended approach. If you need to persist the metadata of your issues and pull requests comments for audit purposes, one option is to leave around the old system in a read-only state until it can be retired.

## Note on Commit Authors

The way that [GitHub attributes commits to an author](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address#about-commit-email-addresses) is via the [email address in the `git config`](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address#setting-your-commit-email-address-in-git). In GitHub, you can add multiple email addresses, and if you want to see your GitHub avatar show up next to your commits, make sure to [add your email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address#setting-your-commit-email-address-on-github) to your GitHub account.

Look at the example below. The first edit I made in GitHub.com, so it shows up as me. The second edit I made on my local machine, and since my `git config` was set up without using an email address linked to my GitHub account, the commit does not show up as from me:

![Git commits, email addresses, and how GitHub renders the commit author](git-config.png){: .shadow }
_Since the email in my Git config isn't added to my GitHub account, the commit doesn't link to my profile_

To summarize, use `git config --global --edit` to view/update your local Git config and make sure the email address listed here is also [added in GitHub](https://github.com/settings/emails).

## Option 1: Using GitHub Importer

I often forget about this one, and if your repo is on a SAAS service (like Azure DevOps, BitBucket Cloud, etc.), this is going to be the way to go. The [GitHub Importer](https://docs.github.com/en/get-started/importing-your-projects-to-github/importing-source-code-to-github/about-github-importer) works with Git, SVN, and Mercurial. This tool will import the contents of the repository, including history. For repos that require authentication, the wizard walks you through providing that information (typically a personal access token or API key). I've personally used this for migrating Azure DevOps Git repos as well as BitBucket Cloud Git repos to GitHub. 

The downside is that it won't work for those who have self-hosted instances of your source control system (such as Azure DevOps Server or BitBucket Server) since these are likely not internet-facing. It also won't migrate any pull request or issue metadata information.

If using SVN or Mercurial, you will have the ability to [update commit author attribution](https://docs.github.com/en/get-started/importing-your-projects-to-github/importing-source-code-to-github/updating-commit-author-attribution-with-github-importer) during the process. For Git, see the note above on [commit authors](#note-on-commit-authors).

> Note: The [docs](https://docs.github.com/en/get-started/importing-your-projects-to-github/importing-source-code-to-github/about-github-importer) say that this tool supports TFVC, but I've never been able to get it to work, so if you're looking for TFVC, [keep reading](#option-6-tfvc-to-git)!
> 
> > No source repositories were detected at \<url\>/_versionControl. Please check the URL and try again.
{: .prompt-info }

## Option 2: Git Clone Mirror

This is usually what I do when migrating Git repos to GitHub, since it's fast, efficient, and scriptable 😀. This migrates the entire repository, including all history, branches, and tags. The steps are as follows:

```bash
# Pre-requisite: Create repo in GitHub and grab the URL

# migrate the repository
git clone --mirror <source-repo-url>
cd <source-repo-name>
git push --mirror  <github-repo-url>
cd ..
rm -rf <source-repo-name>
```

The [`--mirror`](https://www.git-scm.com/docs/git-clone#Documentation/git-clone.txt---mirror) clones a bare copy of the repository that contains all refs (branches, tags, notes, etc.). With the bare copy of the repository, the files in the directory aren't human readable, that's why we clean up with the `rm -rf` at the end. Run a `git clone <github-repo-url>` to get a local copy of the repository and begin working with the repository in GitHub.

This of course works for transferring Git repos between any Git SCM.

A nice script that can do mass migrations is available [here](https://gist.github.com/dbirks/ed249df1912ec11214327a06e24d816c).

## Option 3: Transfer a Repo in GitHub

If your repository is already in GitHub, and you just need to move it to another user or organization, did you know you can simply [transfer the repo](https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository#transferring-a-repository-owned-by-your-personal-account)?

## Option 4: Get Latest and Start Fresh

Starting fresh might be best for repositories with very old history, and/or repositories with a lot of binaries committed. Here's how this process might look:

1. Create a new EMPTY repo in GitHub - don't initialize with a README or .gitignore
2. Get your latest changes in existing SCM; e.g.: `git pull` or `tf get` to get latest changes
3. Create a new folder and copy in the files from your existing repo - make sure to to NOT copy over the `.git`{: .filepath} or `$tf`{: .filepath} folder
4. Initialize a new Git repo: `git init`
5. **Important**: Add in a [gitignore](https://git-scm.com/docs/gitignore) file - for .NET developers, you can run `dotnet new gitignore`, alternatively see [github/gitignore](https://github.com/github/gitignore) repo for other templates
6. Delete any binaries or other files you don't want committed to the new repo, and/or modify gitignore as needed
7. Add the origin: `git remote add origin <github-repo-url>`
8. Push: `git push -u origin main`

Entirely scripted out, it would look something like this:

```bash
cd old-repo
# grab latest changes
git pull
# copy files to new folder and cd into it
mkdir ./../new-repo && cp -r ./* ./../new-repo && cd ./../new-repo
# initialize new git repo
git init
# add in and populate gitignore - either yourself or use `dotnet new gitignore`
touch .gitignore
# add in files, commit, and make sure we're using the main branch
git add .
git commit -m "Initial commit"
git branch -M main
# add remote and push
git remote add origin <github-repo-url>
git push -u origin main
```

## Option 5: SVN to Git

1. See my [blog post](https://josh-ops.com/posts/migrate-svn-to-git/)! 
2. Otherwise, if your SVN repo is internet-accessible, see: [GitHub Importer](#option-1-using-github-importer)

## Option 6: TFVC to Git

1. Perhaps the simplest is to use the [Azure Repos Git repo importer](https://learn.microsoft.com/en-us/azure/devops/repos/git/import-from-TFVC?view=azure-devops) to convert from TFVC to Git, and then migrate that repo to GitHub using [Option 1](#option-1-using-github-importer) or [Option 2](#option-2-git-clone-mirror) above
    - If self-hosted, requires TFS 2017.2 or later
    - Note that it can only migrate 180 days of history and only supports 1 branch 
2. For more advanced migrations, use the [Git-TFS tool](https://github.com/git-tfs/git-tfs)
    - See this page for more information on the [`git-tfs clone`](https://github.com/git-tfs/git-tfs/blob/master/doc/commands/clone.md) command
    - There are various blog posts [here](https://gist.github.com/AAugustine/268f7eed2043de24526b9254a0881579), [here](https://medium.com/sestek/how-to-migrate-projects-from-tfs-to-git-ff23d6b0c910), and [here](https://gitstack.com/how-to-migrate-from-tfs-to-git/) if you need additional guidance

See [Microsoft's documentation](https://learn.microsoft.com/en-us/devops/develop/git/migrate-from-tfvc-to-git) for more information on this topic.

## Option 7: Other Third-Party Tools

[Search GitHub](https://github.com/search?q=git+migrate+repo+&type=repositories) and see what else is out there! 

## Summary

There is no shortage of options for migrating repositories, and while Git is certainly the easiest, there are still options for SVN, Mercurial, and TFVC out there. 

I hope this post helps you find the right tool for your migration needs! 🥳
