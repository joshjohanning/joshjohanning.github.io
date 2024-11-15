---
title: 'GitHub Issues: Scripts for working with Sub-Issues and Issue Types'
author: Josh Johanning
date: 2024-11-15 1:00:00 -0600
description: A collection of scripts for working with sub-issues and issue types in GitHub Issues
categories: [GitHub, Scripts]
tags: [GitHub, GitHub Issues, GitHub Projects]
media_subpath: /assets/screenshots/2024-11-15-github-sub-issues-and-issue-types
image:
  path: sub-issues-issue-types-light.png
  width: 100%
  height: 100%
  alt: Sub-Issues and Issue Types in GitHub Issues
---

## Overview

Public previews for [Sub-Issues](https://github.com/orgs/community/discussions/139932) and [Issue Types](https://github.com/orgs/community/discussions/139933) have recently shipped, and they are *awesome*! 🎉 I encourage you to sign your org up for the opt-in public preview [here](https://github.com/features/issues/signup).

I was looking to do some automation with sub-issues and issue types, and noticed that right now we have to use the GraphQL API to work with them. To run certain queries and mutations, we need the GraphQL IDs of the fields, which if you don't know GraphQL, can be a bit of a challenge. I created these helper scripts to abstract this process and make automation much easier. 🚀

> Check out [@mickeygousset](https://github.com/mickeygousset)'s videos for working with [sub-issues](https://www.youtube.com/watch?v=F42FN6cZmA4) and [issue types](https://www.youtube.com/watch?v=2wVmcuCC1is)! ✨
{: .prompt-info }

## The Scripts

### Sub-Issue Scripts

- [`get-parent-issue-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-parent-issue-of-issue.sh)
- [`get-sub-issues-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-sub-issues-of-issue.sh)
- [`get-sub-issues-summary-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-sub-issues-summary-of-issue.sh)
- [`add-sub-issue-to-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/add-sub-issue-to-issue.sh)
- [`remove-sub-issue-from-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/remove-sub-issue-from-issue.sh)

### Issue Type Scripts

- [`get-issue-type-of-issue.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/get-issue-type-of-issue.sh)
- [`update-issue-issue-type.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/update-issue-issue-type.sh)
- [`remove-issue-issue-type.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/remove-issue-issue-type.sh)

## Usage

### Sub-Issue Scripts Usage

#### get-parent-issue-of-issue.sh

Gets the parent issue of the specified issue.

Query:

```sh
./get-parent-issue-of-issue.sh joshjohanning-org migrating-ado-to-gh-issues-v2 7
```
{: .nolineno}

Response:

```json
{
  "title": "Website enhancements",
  "number": 5,
  "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/5",
  "id": "I_kwDONO_ztc6eXvAG",
  "issueType": "Feature"
}
```
{: .nolineno}

#### get-sub-issues-of-issue.sh

Gets a list of sub-issues for the specified issue.

Query:

```sh
./get-sub-issues-of-issue.sh joshjohanning-org migrating-ado-to-gh-issues-v2 5
```
{: .nolineno}

Response:

```json
{
  "totalCount": 3,
  "issues": [
    {
      "title": "Fix login page",
      "number": 6,
      "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/6",
      "id": "I_kwDONO_ztc6eXvDa",
      "issueType": "User Story"
    },
    {
      "title": "Increase contrast of members page",
      "number": 7,
      "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/7",
      "id": "I_kwDONO_ztc6eXvGo",
      "issueType": null
    },
    {
      "title": "Add logout button",
      "number": 8,
      "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/8",
      "id": "I_kwDONO_ztc6eXvKm",
      "issueType": null
    }
  ]
}
```
{: .nolineno}

#### get-sub-issues-summary-of-issue.sh

Gets the sub-issues summary for the specified issue.

Query:

```sh
./get-sub-issues-summary-of-issue.sh joshjohanning-org migrating-ado-to-gh-issues-v2 5
```
{: .nolineno}

Response:

```json
{
  "total": 3,
  "completed": 1,
  "percentCompleted": 33
}
```
{: .nolineno}

#### remove-sub-issue-from-issue.sh

Removes the specified sub-issue from the specified parent issue.

Query:

```sh
./remove-sub-issue-from-issue.sh joshjohanning-org migrating-ado-to-gh-issues-v2 5 9
```
{: .nolineno}

Response:

```text
Child issue #9 is a sub-issue of parent issue #5.
{
  "data": {
    "removeSubIssue": {
      "issue": {
        "title": "Website enhancements",
        "number": 5,
        "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/5",
        "id": "I_kwDONO_ztc6eXvAG",
        "issueType": {
          "name": "Feature"
        }
      },
      "subIssue": {
        "title": "task 1",
        "number": 9,
        "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/9",
        "id": "I_kwDONO_ztc6eXvN3",
        "issueType": null
      }
    }
  }
}
Successfully removed issue joshjohanning-org/migrating-ado-to-gh-issues-v2#9 as a sub-issue to joshjohanning-org/migrating-ado-to-gh-issues-v2#5.
```
{: .nolineno}


### Issue Types Scripts Usage

#### get-issue-type-of-issue.sh

Gets the issue type of the specified issue.

Query:

```sh
./get-issue-type-of-issue.sh joshjohanning-org migrating-ado-to-gh-issues-v2 5
```
{: .nolineno}

Response:

```json
{
  "title": "Website enhancements",
  "number": 5,
  "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/5",
  "id": "I_kwDONO_ztc6eXvAG",
  "issueType": "Feature"
}
```
{: .nolineno}

#### update-issue-issue-type.sh

Updates/sets the issue type of the specified issue.

Query:

```sh
./update-issue-issue-type.sh joshjohanning-org migrating-ado-to-gh-issues-v2 6 "user story"
```
{: .nolineno}

Response:

```json
{
  "data": {
    "updateIssueIssueType": {
      "issue": {
        "title": "Fix login page",
        "number": 6,
        "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/6",
        "id": "I_kwDONO_ztc6eXvDa",
        "issueType": {
          "name": "User Story"
        }
      }
    }
  }
}
```
{: .nolineno}

#### remove-issue-issue-type.sh

Removes the issue type from the specified issue.

Query:

```sh
./remove-issue-issue-type.sh joshjohanning-org migrating-ado-to-gh-issues-v2 6
```
{: .nolineno}

Response:

```json
{
  "data": {
    "updateIssueIssueType": {
      "issue": {
        "title": "Fix login page",
        "number": 6,
        "url": "https://github.com/joshjohanning-org/migrating-ado-to-gh-issues-v2/issues/6",
        "id": "I_kwDONO_ztc6eXvDa",
        "issueType": null
      }
    }
  }
}
```
{: .nolineno}

## Summary

Working with GraphQL can sometimes be challenging, especially if you're more familiar with REST APIs. I hope you find these scripts helpful! Please let me know if you have any questions or feedback. 🚀 Keep an eye on [the changelog](https://github.blog/changelog/label/projects/) for new Issues/Projects features!
