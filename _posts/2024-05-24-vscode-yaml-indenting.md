---
title: 'Preventing Auto-Indentation on Paste in VS Code for YAML Files'
author: Josh Johanning
date: 2024-05-24 16:00:00 -0500
description: Stop VS Code from messing up your YAML indentation on paste
categories: [macOS, Development Environment]
tags: [VS Code, Development Environment]
media_subpath: /assets/screenshots/2024-05-24-vscode-yaml-indenting
image:
  path: yaml-pasting-indenting-broken.gif
  width: 100%
  height: 100%
  alt: VS Code erroneously auto-indenting YAML on paste
---

## Overview

This is a quick post on making your VS Code better and to stop it from auto-indenting your YAML indentation when pasting. Without this, every time we paste in YAML content into a YAML file in VS Code, it tries to auto-indent which usually ends up messing up the indentation. Unless you like using `Shift` + `Tab` every time you paste into a GitHub Actions workflow file, this is a must-have setting.

If you're unsure of what I'm talking about, look at the GIF above and see a simply copy/paste in the same file adds an additional level of indenting that I do not want in my YAML file.

## The Fix

In VS Code, open the command palette (`CMD`/`CTRL` + `Shift` + `P`) and type `> Preferences: Open User Settings (JSON)`.

Add the following setting to the end of your `settings.json`{: .filepath} file:

```json
  {
    "[yaml]": {
        "editor.autoIndent": "keep",
        "editor.tabSize": 2,
    },
    "[github-actions-workflow]": {
        "editor.autoIndent": "keep",
        "editor.tabSize": 2,
      }
  }
```
{: file='settings.json'}

This setting will prevent auto-indentation on paste for YAML files. You'll notice that there's 2 blocks: the `[yaml]` block and the `[github-actions-workflow]` block. If you have the VS Code extension for GitHub Actions, the `[github-actions-workflow]` is required. With this, you can set different settings GitHub Action YAML files and other YAML files.

I'm also setting the tab size to 2 spaces for YAML files, but you can adjust this to your preference.

Now, when we paste, we see that the indentation is preserved:

![With editor.autoIndent enabled, paste works again!](yaml-pasting-indenting-fixed.gif){: .shadow }
_With editor.autoIndent enabled, paste works again!_

Alternatively, you can prevent the auto-indentation on paste for all file types by adding the following setting:

```json
    "editor.autoIndent": "keep"
```
{: file='settings.json'}

Here's more background on the `editor.autoIndent` setting:

```text
  // Controls whether the editor should automatically adjust the indentation when users type, paste, move or indent lines.
  //  - none: The editor will not insert indentation automatically.
  //  - keep: The editor will keep the current line's indentation.
  //  - brackets: The editor will keep the current line's indentation and honor language defined brackets.
  //  - advanced: The editor will keep the current line's indentation, honor language defined brackets and invoke special onEnterRules defined by languages.
  //  - full: The editor will keep the current line's indentation, honor language defined brackets, invoke special onEnterRules defined by languages, and honor indentationRules defined by languages.
```
{: .nolineno}

## Summary

When copying and pasting YAML code, such as GitHub Actions workflows or steps, into VS Code, this is a must have setting. I hope this helps! 🚀
