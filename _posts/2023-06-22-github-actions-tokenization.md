---
title: 'Tokenization / Replacing Environment Tokens in GitHub Actions'
author: Josh Johanning
date: 2023-06-22 16:30:00 -0500
description: Replacing environment-specific configuration at deployment time
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions]
media_subpath: /assets/screenshots/2023-06-22-github-actions-tokenization
image:
  path: tokenization.png
  width: 100%
  height: 100%
  alt: Replacing environment-specific configuration at deployment time
---

## Overview

I've been thinking about the "build once, deploy many" concept lately, and how to accomplish this in GitHub Actions. We definitely don't want to be creating a "dev" build, testing it, then creating a new "prod" build and deploying that. They are two separate binaries; there is no way to verify that what we tested in dev is what is shipped to prod. We want to create a single build and deploy that to multiple environments.

In Azure DevOps, I primarily used [Colin's ALM Corner Build & Release Tools](https://marketplace.visualstudio.com/items?itemName=colinsalmcorner.colinsalmcorner-buildtasks) Replace Tokens task or the standalone [Replace Tokens](https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens) extension and task to find/replace environment-specific files during a deployment. I was exploring how to do this in GitHub Actions, and I think I found a way to mostly recreate this pattern.

I'm using the [lindluni/actions-variable-groups](https://github.com/marketplace/actions/github-actions-variable-groups) and [cschleiden/replace-tokens](https://github.com/marketplace/actions/replace-tokens) actions to accomplish this. The `actions-variable-groups` action allows you to store the values of the (non-secret) variables to a file (variables-as-code!) in the repository, and the `replace-tokens` action does the find-and-replacing.

## The Workflow

Here is the workflow:

{% raw %}

```yml
 dev:
    runs-on: ubuntu-latest
    environment: dev
    needs: build

    steps:
    - uses: actions/checkout@v3

    - name: Inject Variables
      uses: lindluni/actions-variable-groups@v2
      with:
        groups: |
        .github/variables/dev.yml

    - uses: cschleiden/replace-tokens@v1
      with:
        tokenPrefix: '#{'
        tokenSuffix: '}#'
        files: '["**/*_tokenized.json"]'

    - name: replace app settings
      run: |
        rm appsettings.json
        mv appsettings_tokenized.json appsettings.json
```

There is an alternative to the `actions-variable-groups` action where you instead use [configuration variables](https://docs.github.com/en/actions/learn-github-actions/variables#creating-configuration-variables-for-an-environment) defined on a repository's environment. I prefer the `actions-variable-groups` action for a few reasons:

1. It allows you to store the variables in the repository as code, so it's easier to diff/review changes
2. Non-admins can view/modify the variables 
3. You have to map in each environment variable to the workflow, which is a bit more cumbersome than just specifying the file to inject

If you wanted to see how it would look using configuration variables, it would look something like this:

```yml
 dev:
    runs-on: ubuntu-latest
    environment: dev
    needs: build

    steps:
    - uses: actions/checkout@v3

    - uses: cschleiden/replace-tokens@v1
      env:
        APIURL: ${{ vars.APIURL }}
        VAR2: ${{ vars.VAR2 }}
        VAR3: ${{ vars.VAR3 }}
      with:
        tokenPrefix: '#{'
        tokenSuffix: '}#'
        files: '["**/*_tokenized.json"]'

    - name: replace app settings
      run: |
        rm appsettings.json
        mv appsettings_tokenized.json appsettings.json
```

## Secrets?

Secrets should not be stored in the repository (obviously). Create these as secrets and map them in as environment variables, like so:

```yml
 dev:
    runs-on: ubuntu-latest
    environment: dev
    needs: build

    steps:
    - uses: actions/checkout@v3

    - name: Inject Variables
      uses: lindluni/actions-variable-groups@v2
      with:
        groups: |
        .github/variables/dev.yml

    - uses: cschleiden/replace-tokens@v1
      env:
        APIKEY: ${{ secrets.APIKEY }}
      with:
        tokenPrefix: '#{'
        tokenSuffix: '}#'
        files: '["**/*_tokenized.json"]'

    - name: replace app settings
      run: |
        rm appsettings.json
        mv appsettings_tokenized.json appsettings.json
```

{% endraw %}

Note that if you are injecting a secret into a raw file, ensure that wherever this file is ending up is not somewhere where it can be accessed by the general viewing public. If using self-hosted runners, it is a good idea to have a step that always runs to delete the tokenized file.

## iOS?

You can even do this with compiled iOS IPA files. An IPA is just a fancy zip, so we can extract it, replace values, re-zip as a `.ipa`{: .filepath} file, and re-sign it.

I'm going to briefly outline this here; I don't have an app to test with at the moment, so some things might need to be tweaked slightly (such as unzip/zip folder paths):

- Here is a sample [gist](https://gist.github.com/joshjohanning/15e2bda76687d353a50211a7477de370#file-deploy-yml-L18:L22) as to not bog down this post too much

> Note: You can see how I did this with a real-world app in Azure DevOps in a [deployment pipeline here](https://github.com/joshjohanning/pipeline-templates/blob/main/ios/ios-deploy.yml).
{: .prompt-info }

## Summary

This is essentially the "GitHub" flavor of Colin's [Config Per Environment vs Tokenization in Release Management](https://colinsalmcorner.com/config-per-environment-vs-tokenization-in-release-management/) and [End to End Walkthrough: Deploying Web Applications Using Team Build and Release Management](https://colinsalmcorner.com/end-to-end-walkthrough-deploying-web-applications-using-team-build-and-release-management/) posts from many years ago. Hopefully this helps give you an idea of how you can accomplish this in GitHub Actions.
