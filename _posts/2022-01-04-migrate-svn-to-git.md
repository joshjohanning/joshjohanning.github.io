---
title: 'Migrate SVN to Git'
author: Josh Johanning
date: 2022-01-04 23:00:00 -0600
description: Migrate SVN to Git using either the import repository feature in GitHub or git-svn
categories: [GitHub, Migrations]
tags: [GitHub, Azure DevOps, SVN, Git, Migrations]
media_subpath: /assets/screenshots/2022-01-04-migrate-svn-to-git
image:
  path: svn-to-git.png
  width: 100%
  height: 100%
  alt: Migrating SVN to Git
---

## Overview

Let's face it: Subversion had its time in the sun, but Git is the more modern source control system. If you want to use GitHub and take advantage of all the collaboration and security features, you're going to want your source code in GitHub. In this post, I describe several options on how to make the jump to Git and GitHub and bring your code (including history!) with you.

## GitHub Importer

Probably the easiest (and yet the least likely you'll be able to use) is the [GitHub Repo Importer](https://docs.github.com/en/enterprise-cloud@latest/github/importing-your-projects-to-github/importing-source-code-to-github/about-github-importer) (you can use this for SVN, Mercurial, TFVC, and of course, Git). When you create a new repository in GitHub, there is a little blue link that allows you to [Import a repository](https://docs.github.com/en/enterprise-cloud@latest/github/importing-your-projects-to-github/importing-source-code-to-github/importing-a-repository-with-github-importer). If you forget to click the link to import a repository at the time you are creating and naming your GitHub repo, you can still import after repo creation if you haven't initialized the repository with a Readme or .gitignore.

The reason why I say least likely to be able to use is that this requires your SVN server to be publicly accessible from GitHub.com. Most Subversion servers I run into our hosted on-premises, which means you're pretty much out of luck.

If this does work for you, provide the repository url, credentials, and if applicable, which project you are importing, and away you go. 

*Note: According to the documentation, the GitHub Repository Importer is not a feature in GitHub Enterprise Server yet.*

## git-svn

This is the tool I have the most experience with. Using `git svn` commands, you can create a Git repo from a repo hosted in Subversion (history included). The larger the repo is and the more history there is, the longer the migration will take. Once the repo has been migrated, it can be pushed to GitHub, Azure DevOps, or any other Git host. 

See the [official documentation](https://git-scm.com/book/en/v2/Git-and-Other-Systems-Migrating-to-Git) for migrating from SVN to Git with the `git svn` commands.

The high-level process is as follows:

1. Extract the authors from the SVN repo to create an `authors.txt`{: .filepath} mapping file
2. Modify the mapping file with the author names and email addresses
3. Run `git svn clone` command
4. Clean up tags and branches
5. Create a Git repo in GitHub / Azure Repos
6. Add the Git repo remote to the local repo and push

### System Pre-Reqs

* Windows:
  * [Git for Windows](https://git-scm.com/download/win)
  * [TortoiseSVN](https://tortoisesvn.net/downloads.html) - When installing, check the box to install the '**command line client tools**' (not checked by default). Modify or uninstall/re-install if you did not do this with your initial installation. This allows you to run the `svn` commands from the command line

* macOS Catalina, Big Sur, Monterey, and greater: 
  * Run this command to install the `git`, `svn`, and `git svn` commands:
`xcode-select --install`
  * `git` should already be installed, so alternatively you can just install `svn` with the corresponding `brew` formulae: `brew install subversion`
      - You can also ensure you have the latest version of `git`: `brew install git` or `brew upgrade git`

### Option 1: Tags as Branches

These commands clone an SVN repository to Git, perform some cleanup, and push it to your Git host of choice. Branches will appear as `/origin/<branch-name>`. In GitHub/Azure DevOps, you can clean this up by re-creating the branch at the root, e.g., creating a new branch `/<branch-name>` based on `/origin/<branch-name>`. You can confirm the commit hashes are the same and then delete the branch under `/origin`. You can delete `/origin/trunk` without re-creating it because trunk should have been re-created as master. 

Tags will appear as branches, e.g.: `/origin/tags/<tag-name>`. You can clean this up by re-creating the tag branch at the root, e.g. `/tags/<tag-name>` or `/<tag-name>`. Otherwise, you can manually create a tag in the tags page in [GitHub](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)/[Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/repos/git/git-tags?view=azure-devops&tabs=browser#create-tag) based off of the `/origin/tags/<tag-name>` branch reference. Branches and tags are just pointers in Git anyway, so whether it appears as a tag or a branch, the referenced commit SHA will be the same.

Note: In GitHub, when you create a release, you must specify a tag. So, creating a [release](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository) in the web interface will create a tag. Otherwise, you can use the [command line](https://stackoverflow.com/a/18223354/4270353) to create tags.

1. Get a list of the committers in an SVN repo:

    ```bash
    svn log -q http://svn.mysvnserver.com/svn/MyRepo | awk -F '|' '/^r/ {sub("^ ", "", $2); sub(" $", "", $2); print $2" = "$2" <"$2">"}' | sort -u > authors-transform.txt
    ```
    {: .nolineno}

1. Modify each line to map the SVN username to the Git username, e.g.: `josh = Josh <josh@example.com>`
    - Make sure the file is encoded as [UTF-8](https://stackoverflow.com/a/57892875/4270353)

1. Clone an SVN repo to Git:

    ```bash
    git svn clone http://svn.mysvnserver.com/svn/MyRepo --authors-file=authors-transform.txt --trunk=trunk --branches=branches/* --tags=tags MyRepo
    ```
    {: .nolineno}

    Note: In case of a non-standard layout, replace `trunk`, `branches`, and `tags` with appropriate names

1. Git Tags cleanup (creating local tags off of the `remotes/tags/<tag-name>` reference so that we can push them):

    ```bash
    git for-each-ref refs/remotes/tags | cut -d / -f 4- | grep -v @ | while read tagname; do git tag "$tagname" "tags/$tagname"; git branch -r -d "tags/$tagname"; done
    ```
    {: .nolineno}

1. Git Branches cleanup (creating local branches off of the `remotes/<branch-name>` reference so that we can push them):

    ```bash
    git for-each-ref refs/remotes | cut -d / -f 3- | grep -v @ | while read branchname; do git branch "$branchname" "refs/remotes/$branchname"; git branch -r -d "$branchname"; done
    ```
    {: .nolineno}

1. Add the remote:

    ```bash
    git remote add origin https://github.com/<user-or-org>/<repo-name>.git
    ```
    {: .nolineno}

1. Push the local repo to Git host:

    ```bash
    git push -u origin --all
    ```
    {: .nolineno}

This is what you can expect tags to look like in [GitHub](https://github.com/joshjohanning/GitSvn-TagsAsBranches/branches/all?query=origin%2F) after running the migration (as branches):
![Option 2 - Tags as Branches in GitHub](option1-github-tags-as-branches.png){: .shadow }
_How tags appear in GitHub (as branches) - You can even see that Dependabot created a few branches!_

And in Azure DevOps:
![Option 2 - Tags as Branches in Azure DevOps](option1-azdo-tags-as-branches.png){: .shadow }
_How tags appear in Azure DevOps (as branches)_

### Option 2: Tags as Tags

When following the [above instructions](#option-1-tags-as-branches), tags will appear as a branch `/origin/tags/<tag-name>`. This is usually fine since branches and tags are just pointers in Git anyway, so whether it appears as a tag or a branch, the referenced commit SHA will be the same. 

If you want to see the tags show under the tags page instead of the branches page in [GitHub](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)/[Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/repos/git/git-tags?view=azure-devops&tabs=browser#create-tag), you can manually create a new tag based on the branch in `/origin/tags/`, or follow the alternative commands below (particularly step `#4`).

Note: In GitHub, when you create a release, you must specify a tag. So, creating a [release](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository) in the web interface will create a tag. Otherwise, you can use the [command line](https://stackoverflow.com/a/18223354/4270353) to create tags.

1. Get a list of the committers in an SVN repo:

    ```bash
    svn log -q http://svn.mysvnserver.com/svn/MyRepo | awk -F '|' '/^r/ {sub("^ ", "", $2); sub(" $", "", $2); print $2" = "$2" <"$2">"}' | sort -u > authors-transform.txt
    ```
    {: .nolineno}

1. Modify each line to map the SVN username to the Git username, e.g.: `josh = Josh <josh@example.com>`
    - Make sure the file is encoded as [UTF-8](https://stackoverflow.com/a/57892875/4270353)

1. Clone an SVN repo to Git:

    ```bash
    git svn clone http://svn.mysvnserver.com/svn/MyRepo --authors-file=authors-transform.txt --trunk=trunk --branches=branches/* --tags=tags MyRepo
    ```
    {: .nolineno}

    Note: In case of a non-standard layout, replace `trunk`, `branches`, and `tags` with appropriate names

1. Create Git Tags based on the message that was originally in SVN.

    ```bash
    git for-each-ref --format="%(refname:short) %(objectname)" refs/remotes/origin/tags \
    | while read BRANCH REF
      do
            TAG_NAME=${BRANCH#*/}
            BODY="$(git log -1 --format=format:%B $REF)"
    echo "ref=$REF parent=$(git rev-parse $REF^) tagname=$TAG_NAME body=$BODY" >&2
    git tag -a -m "$BODY" $TAG_NAME $REF^  &&\
            git branch -r -d $BRANCH
      done
    ```

1. Git Branches cleanup (creating local branches off of the `remotes/<branch-name>` reference so that we can push them):

    ```bash
    git for-each-ref refs/remotes | cut -d / -f 3- | grep -v @ | while read branchname; do git branch "$branchname" "refs/remotes/$branchname"; git branch -r -d "$branchname"; done
    ```
    {: .nolineno}

1. Add the remote:

    ```bash
    git remote add origin https://github.com/<user-or-org>/<repo-name>.git
    ```
    {: .nolineno}

1. Push the local repo to Git host:

    ```bash
    git push -u origin –all
    ```
    {: .nolineno}

1. Push the tags to Git host:

    ```bash
    git push --tags
    ```
    {: .nolineno}

This is what you can expect tags to look like in [GitHub](https://github.com/joshjohanning/GitSvn-Tags/tags) after running the migration (as tags):
![Option 2 - Tags as Tags in GitHub](option2-github-tags-as-tags.png){: .shadow }
_How tags appear in GitHub (as tags)_

And in Azure DevOps:
![Option 2 - Tags as Tags in Azure DevOps](option2-azdo-tags-as-tags.png){: .shadow }
_How tags appear in Azure DevOps (as tags)_

### Clone partial history from SVN

This can be useful if you only want/need history from the last X months or last N revisions cloned from the SVN repository. This can help to speed up the conversion as well as potentially bypassing any errors (such as server timeout). You must pick/find what revision you want to start with manually, though. In this example I am getting everything from revision 3000 to current (HEAD):

```bash
git svn clone -r3000:HEAD http://svn.mysvnserver.com/svn/MyRepo --authors-file=authors-transform.txt --trunk=trunk --branches=branches/* --tags=tags MyRepo
```

You can use an SVN client ([TortoiseSVN](https://tortoisesvn.net/downloads.html) on Windows, [SmartSVN](https://www.smartsvn.com/download/) on Mac) or [git svn log](https://svnbook.red-bean.com/en/1.7/svn.ref.svn.c.log.html) to help you with finding out what revision to start with. Alternatively, if you want to precisely find the previous N revision, you can use the 3rd party scripts found [here](https://github.com/jonathancone/svn-utils).

### Metadata

The `--no-metadata` option can be used in the `git svn` command (steps `#3` above) for one-shot imports, like we are essentially what we are doing here, but it won't include the git-svn-id (url) in the new git commit message. If this is a one-shot import, and you don't want to be cluttered with the old git-svn-id (url), include this option. 

From the [git-svn documentation](https://git-scm.com/docs/git-svn#Documentation/git-svn.txt-svn-remoteltnamegtnoMetadata): 

> Set the `noMetadata` option in the [svn-remote] config. This option is not recommended.
> 
> This gets rid of the `git-svn-id`: lines at the end of every commit.
> 
> This option can only be used for one-shot imports as `git svn` will not be able to fetch again without metadata. Additionally, if you lose your `$GIT_DIR/svn/**/.rev_map.*` files, `git svn` will not be able to rebuild them.

You can compare the difference between adding `--no-metadata` and not in the examples of my migration runs:
- [Tags as Branches](https://github.com/joshjohanning/GitSvn-TagsAsBranches) (with `--no-metadata`)
- [Tags as Tags](https://github.com/joshjohanning/GitSvn-Tags) (without `--no-metadata`)

Note that my initial commit in SVN didn't have a commit message, that's why it's showing "No commit message" for most of the files. `git svn` migrates commit messages with or without `--no-metadata`.

### Resources / Bookmarks

This is my stash of references I used that may be helpful for you:

* [Converting a Subversion repository to Git](https://john.albin.net/git/convert-subversion-to-git) and [cleaning up binaries in the process](https://meejah.ca/blog/migrating-svn-to-git#remove-all-the-things)
* [tortoise svn giving me "Redirect cycle detected for URL 'domain/svn'"](https://stackoverflow.com/questions/10524646/tortoise-svn-giving-me-redirect-cycle-detected-for-url-domain-svn)
* [Why do I get “svn: E120106: ra_serf: The server sent a truncated HTTP response body” error?](https://stackoverflow.com/questions/27267742/why-do-i-get-svn-e120106-ra-serf-the-server-sent-a-truncated-http-response-b)
* [How to import svn branches and tags into git-svn?](https://stackoverflow.com/a/27210896/4270353) and [convert git-svn tag branches to real tags](https://gitready.com/advanced/2009/02/16/convert-git-svn-tag-branches-to-real-tags.html)
* [What is the format of an authors file for git svn, specifically for special characters like backslash or underscore?](https://stackoverflow.com/questions/2159567/what-is-the-format-of-an-authors-file-for-git-svn-specifically-for-special-char)
* [Git svn clone with author name like "/CN=myname"](https://stackoverflow.com/questions/16687094/git-svn-clone-with-author-name-like-cn-myname/25012409#25012409)
* [Author not defined when importing SVN repository into Git](https://stackoverflow.com/questions/11037166/author-not-defined-when-importing-svn-repository-into-git/28303400) (make sure the file is encoded as [UTF-8](https://stackoverflow.com/a/57892875/4270353))
* [git svn --ignore-paths regex](https://github.com/git/git-scm.com/issues/698#issuecomment-239408209), and [How is ignore-paths evaluated in git svn?](https://stackoverflow.com/questions/10448431/how-is-ignore-paths-evaluated-in-git-svn)
* [SVN and KeepAlive (svn: E175002: Connection reset)](https://marioharvey.com/svn-and-keepalive/)
* [How to git-svn clone the last n revisions from a Subversion repository?](https://stackoverflow.com/a/35661167/4270353) and [Git Svn clone certain revision, and continue cloning other revisions in the future](https://stackoverflow.com/questions/17689582/git-svn-clone-certain-revision-and-continue-cloning-other-revisions-in-the-futu/17768204)

## svn2git

GitHub's [importing source code to GitHub documentation](https://docs.github.com/en/enterprise-cloud@latest/github/importing-your-projects-to-github/importing-source-code-to-github/source-code-migration-tools#importing-from-subversion) mentions another tool you can use as well - [svn2git](https://github.com/nirvdrum/svn2git). I do not have any experience with this tool but wanted to call it out here as another option.

## Tip Migration

I'd be remiss if I did not mention that there's always the option of *just migrating the tip* - meaning, grab the latest code from SVN and *start fresh with a new repo in GitHub*. Leave all of the history in SVN and start fresh in GitHub by coping in the files, creating a gitignore to exclude any binaries and other unwanted files, and pushing. Ideally, you could keep the SVN server around for a while or make an archive somewhere that it would still be possible to view / recover the history.

Understandably, this won't work for everyone, but it is always an option if the migration options aren't worth the effort, and you really just care about your most recent code being in GitHub. 

## Wrap-up

Now that you have your code migrated to Git, the hard part of moving to GitHub is behind you. Even if you're not using GitHub, migrating from SVN to Git certainly has its advantages.

I will note that once the code is in GitHub, it is technically possible to use [svn clients](https://docs.github.com/en/github/importing-your-projects-to-github/working-with-subversion-on-github/support-for-subversion-clients) to connect to repositories on GitHub, if you're in GitHub I think it is wise to use Git like everyone else in GitHub :).

Did I miss anything, or have you any improvements to be made? Let me know in the comments!
