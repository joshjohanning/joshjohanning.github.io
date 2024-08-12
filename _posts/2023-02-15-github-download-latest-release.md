---
title: 'Programmatically Download Latest Release from GitHub Repo'
author: Josh Johanning
date: 2023-02-15 11:30:00 -0600
description: Programmatically download the latest release from a GitHub Repo without having to hardcode the version or use separate API calls
categories: [GitHub, Scripts]
tags: [GitHub, Scripts, Releases]
media_subpath: /assets/screenshots/2023-02-15-github-download-latest-release
image:
  path: release.png
  width: 100%
  height: 100%
  alt: The Latest Release in a GitHub Repo
---

## Overview

I recently needed to download the latest release from a GitHub repo but didn't want to hardcode the version number or use separate API calls to get the latest release. I found a few [one-liner solutions online](https://gist.github.com/steinwaywhw/a4cd19cda655b8249d908261a62687f8), but most of these made two separate calls: 
1. an [API call to get the latest release](https://docs.github.com/en/rest/releases/releases?apiVersion=2022-11-28#get-the-latest-release)
2. then another to download that release asset

I swore there was a way to get this in one call, but it took a little bit of digging for me to find it. I'm posting this here for posterity and future searchers 😀.

## The script

Well, it's not much of a script, more of a command, but here it is for both `wget` and `curl`. The only thing you need to know is the filename of the asset you want to download. In this case, it's `tfsec-linux-amd64`{: .filepath} . If the maintainers chose to add in version numbers in the filename, like `tfsec-linux-amd64-v1.28.1`{: .filepath} , you would need to use an [alternative one-liner](https://gist.github.com/steinwaywhw/a4cd19cda655b8249d908261a62687f8) to make that second API call to get the version number.

How to download the latest version of a release asset:

### wget

```sh
wget https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64
```
{: .nolineno}

### curl

```sh
curl -LO https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64
```
{: .nolineno}

> The `-O` (case sensitive) saves the file as the same name specified in the URL, and `-L` follows redirects.
{: .prompt-tip }

## Download a Specific Version

I'm posting this here in case you have the _exact opposite_ problem and need to download a specific version of a release asset.

If you need to download a specific version of a release, you can use:

```sh
wget https://github.com/aquasecurity/tfsec/releases/download/v1.28.1/tfsec-linux-amd64
```
{: .nolineno}

## Summary

To make your CI jobs _repeatable_, it makes more sense to have a hardcoded version of a release asset. Just make sure to update the version every now and then 😁. If you just need to download the latest version of a release asset, then the above solution will be perfect!
