---
title: 'GitHub Actions: Create a Composite Action in Python'
author: Josh Johanning
date: 2024-06-18 19:30:00 -0500
description: Create a composite action in Python 🐍 to reduce code duplication and improve maintainability
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Python, Composite Actions]
media_subpath: /assets/screenshots/2024-06-18-github-composite-action-python
image:
  path: composite-action-light.png
  width: 100%
  height: 100%
  alt: A composite action in GitHub Actions written in Python
---

## Overview

In GitHub Actions, a [composite action is a type of action](https://docs.github.com/en/actions/creating-actions/about-custom-actions#types-of-actions) that allows you to combine multiple steps into a single action. This can help reduce code duplication and improve maintainability of your workflow files. In a composite action, you can combine multiple run steps, multiple marketplace actions, or a combination of both! Composite actions are my favorite type of action because of their flexibility to run *anything* in any language/framework/etc. on any host. If it can run programmatically, you can build it as a composite action. In this post, we'll create a composite action in Python in a way that can be used in Actions as well as preserving the ability to test/run the script locally.

## Composite Action in Python

In a composite action, we have to specify the [shell](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#defaultsrunshell) for each run step. Most commonly, I use `shell: bash`, but there is an option for `shell: python` directly. If it was a 2 line script, sure, use `shell: python`, but for anything more complex, I prefer to use `shell: bash` and call the Python script from shell. This allows me to use the same Python script in the composite action as well as run it locally for testing.

Let's show some examples.

### Preferred Python Composite Action

{% raw %}

This is the preferred way to create a Python composite action. Note how we are storing the Python script in a separate file and not directly inline. This allows you to run the Python script locally for testing as well as using in GitHub Actions as a composite action. And if you ever switch CI systems, it would be easy to port since the only "Actions" specific code is small bit in the `action.yml`{: .filepath} file.

Here's the example:

```yml
name: 'Python composite action'
description: 'call a python script from a composite action'
inputs:
  directory:
    description: 'directory path as an example input'
    required: true
    default: '${{ github.workspace }}'
  token:
    description: 'github auth token (PAT, github token, or GitHub app token)'
    required: true
    default: '${{ github.token }}'
runs:
  using: "composite"
  steps:
    - name: run python
      shell: bash
      run: | 
        python3 ${{ github.action_path }}/main.py ${{ inputs.directory }} ${{ inputs.token }}
```
{: file='action.yml'}

The magic 🪄 is that we are calling the Python script from the shell using the `${{ github.action_path }}` [environment variable](https://help.github.com/en/actions/configuring-and-managing-workflows/using-environment-variables). This variable maps to the local directory the action is cloned to so that we can reference files from the repo.

You can then run/test/debug/develop the Python script locally as you would any other Python script:

```bash
python3 main.py /path/to/directory ghp_abcdefg1234567890
```
{: .nolineno}

Store the Python script in the composite action repository. If this was the only action in the repository, it probably makes the most sense to put in the script in the root along with the `action.yml`{: .filepath} file (this is how I'm doing it in the example above):

```text
.
├── README.md
├── action.yml
└── main.py
```
{: .nolineno}

If you had multiple composite actions in the same repository, you could structure it like so:

```text
.
├── README.md
├── python-action-1/
│   ├── README.md
│   ├── action.yml
│   └── main.py
├── python-action-2/
│   ├── README.md
│   ├── action.yml
│   └── main.py
└── python-other-action/
    ├── README.md
    ├── action.yml
    └── main.py
```
{: .nolineno}

> Note that the entire repository will be versioned together when creating/referencing tags, so you may only want to do this if the actions are closely related.
{: .prompt-tip }

You could also do something like this, creating separate folders for the actions (since only one `action.yml`{: .filepath} can exist in a single directory) and then use a combined `./src`{: .filepath} folder for the Python scripts:

```text
.
├── README.md
├── python-action-1/
│   ├── README.md
│   └── action.yml
├── python-action-2/
│   ├── README.md
│   └── action.yml
├── python-other-action/
│   ├── README.md
│   └── action.yml
└── src/
    ├── action-1.py
    ├── action-2.py
    └── other-action.py
```
{: .nolineno}

Or, of course, a combination of whatever makes the most sense for your use case. 😎

### Non-optimal Python Composite Action

Ideally, you wouldn't do this. We cannot run or test this locally, and especially for a longer script, it makes the `action.yml`{: .filepath} file harder to read and maintain.

```yml
name: 'Python composite action'
description: 'call a python script from a composite action'
inputs:
  directory:
    description: 'directory path as an example input'
    required: true
    default: '${{ github.workspace }}'
  token:
    description: 'github auth token (PAT, github token, or GitHub app token)'
    required: true
    default: '${{ github.token }}'
runs:
  using: "composite"
  steps:
    - name: run python
      shell: bash
      run: | 
        import sys

        def main(filePath, creds):
            print("Hello World")
            print(f"File Path: {filePath}")

        if __name__ == "__main__":
            if len(sys.argv) != 3:
                print("Usage: python3 myfile.py <filePath> <creds>")
                sys.exit(1)
            filePath = sys.argv[1]
            creds = sys.argv[2]
            main(${{ inputs.directory }}, ${{ inputs.token }})
```
{: file='action.yml'}

This certainly works, but you can see that it's not as flexible/portable as the [preferred method above](#preferred-python-composite-action). For one, if you wanted to run this locally, you would have to copy/paste and then swap the hardcoded GitHub Actions-isms, like in this example: `${{ inputs.directory }}` and `${{ inputs.token }}`. It's also harder to read and maintain. 😬

{% endraw %}

## Summary

Composite Actions are great! The barrier to entry for creating custom actions are much less than that of JavaScript actions, and in general, I [don't typically recommend Docker-based actions](/posts/github-actions-docker-actions-private-registry/#overview). This post shows a great, real-world example of creating a composite action in our preferred language. I would even use this method to create composite actions written in Bash. Instead of running `python3 main.py` you would just call the Bash script via `./main.sh`. 🐍 🚀
