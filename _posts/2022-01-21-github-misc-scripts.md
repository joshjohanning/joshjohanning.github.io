---
title: 'Miscellaneous GitHub API/GraphQL/CLI Automation Scripts'
author: Josh Johanning
date: 2022-01-20 13:00:00 -0600
description: Miscellaneous GitHub scripts written against GitHub API's, GraphQL, GitHub CLI, etc. for automation
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, gh cli]
---

## Overview

I have a large Postman workspace for all my API calls, but it’s sometimes hard to share an example of an API or script with someone. Thus, I decided to create a repo that consolidates my random GitHub scripts into one central spot. Now, I can simply send a link!

Here's the repo: [joshjohanning/github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts)

## Layout

I have them categorized by type:

* [api](https://github.com/joshjohanning/github-misc-scripts/tree/main/api)
* [gh-cli](https://github.com/joshjohanning/github-misc-scripts/tree/main/gh-cli)
* [graphql](https://github.com/joshjohanning/github-misc-scripts/tree/main/graphql)
* [scripts](https://github.com/joshjohanning/github-misc-scripts/tree/main/scripts)

I have readme's in each of the folders with a brief description of the enclosed scripts.

## Script Examples

Here's an example of some of the scripts I have populated in there so far:

- [download file from github packages](https://github.com/joshjohanning/github-misc-scripts/blob/main/api/download-file-from-github-packages.sh) (api) - (and my [blog post](https://josh-ops.com/posts/github-download-from-github-packages/)!)
- [create repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/api/create-repo.sh) (api)
- [download file from private repo](https://github.com/joshjohanning/github-misc-scripts/blob/main/api/download-file-from-private-repo.sh) (api)
- [download workflow artifacts](https://github.com/joshjohanning/github-misc-scripts/blob/main/api/download-workflow-artifacts.sh) (api)
- [get enterprise id](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/get-enterprise-id.sh) (graphql)
- [create organization](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/create-organization.sh) (graphql)
- [delete repository branch policy](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/delete-repository-branch-policy.sh) (graphql)
- [get issue id](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/get-issue-id.sh) (graphql)
- [get repository branch policies](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/get-repository-branch-policies.sh) (graphql)
- [get repository id](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/get-repository-id.sh) (graphql)
- [transfer issue](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/transfer-issue.sh) (graphql)
- [download specific version from github packages](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/download-specific-version-from-github-packages.sh) (graphql) - (and my [blog post](https://josh-ops.com/posts/github-download-from-github-packages/)!)
- [download latest version from github packages](https://github.com/joshjohanning/github-misc-scripts/blob/main/graphql/download-latest-version-from-github-packages.sh) (graphql) - (and my [blog post](https://josh-ops.com/posts/github-download-from-github-packages/)!)
- [get sso credential authorizations (PATs, SSH Keys)](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-sso-credential-authorizations.sh) (gh-cli)

## Overview

Let me know if you have found any of these useful and/or have improved them!
