---
title: 'Creating a Multiline Comment in the GitHub Web Editor'
author: Josh Johanning
date: 2023-12-22 11:40:00 -0600
description: Adding a multiline comment in the GitHub Web Editor; for example when editing a GitHub Actions workflow file
categories: [GitHub, Commits]
tags: [GitHub, Commits]
media_subpath: /assets/screenshots/2023-12-22-github-web-editor-multiline-comment
image:
  path: multiline-comment.gif
  width: 100%
  height: 100%
  alt: Creating a multiline comment in the GitHub Web Editor
---

## Summary

I'm embarrassed to say that every time I was editing a GitHub Actions workflow file with the web editor in GitHub, if I needed to make a multiline comment, I either added each `#` in manually, cloned the repo, or opening the repo with the [github.dev web-based editor](https://docs.github.com/en/codespaces/the-githubdev-web-based-editor) with the `.` shortcut. I knew how to create a multiline comment in VS Code (`Cmd ⌘` + `k`, `Cmd ⌘` + `c`), but I didn't know how to do it in the web editor or if it was even possible.

Well, TIL!

## The Shortcut

That shortcut is simply `Cmd ⌘` + `/` (or `Ctrl` + `/` on Windows). It's that easy! This comments and uncomments the selected lines.

It seems as if `Cmd ⌘` + `/` is a standard keyboard shortcut for commenting/uncommenting in many editors, including VS Code, Atom, and Sublime Text. I'm probably just used to `Cmd ⌘` + `k`, `Cmd ⌘` + `c` from my Visual Studio days!
