---
title: 'GitHub Actions: Create a Docker Container Action Hosted in a Private Image Registry'
author: Josh Johanning
date: 2024-05-20 20:30:00 -0500
description: Create a Docker container action that is hosted in a private image registry
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Docker]
media_subpath: /assets/screenshots/2024-05-20-github-actions-docker-actions-private-registry
image:
  path: docker-action-composite-action.png
  width: 100%
  height: 100%
  alt: Using a Docker container action that's hosted in a private image registry
---

## Overview

Docker container actions can be awesome! They very neatly encapsulate an entire step's logic into a container image so that no matter what host the Actions runner is running on[^footnote-1], you get the same result. There are a few popular actions in the marketplace that are Docker container actions, such as the [SonarQube Scan action](https://github.com/SonarSource/sonarqube-scan-action) and the [Super-Linter Action](https://github.com/super-linter/super-linter). You can easily tell that these are Docker container actions because their `action.yml`{: .filepath} file has a `using: 'docker'` line in it. You can find other examples of Docker container actions by using [GitHub's search](https://github.com/search?q=%22using%3A+docker%22+language%3AYML+path%3A**%2Faction.yml&type=code).

Sometimes, it makes a lot of sense to use Docker container actions, especially in the SonarQube and Super-Linter action examples above. These actions rely on source files / binaries to exist to run the scan or linting. However, there are a fair share of Docker container actions on the marketplace that didn't really need to be Docker container actions. Part of this is likely due that JavaScript and Docker actions were the only type of action originally (Actions was launched in 2018). Composite actions came around [originally in August 2020](https://github.com/actions/runner/issues/646) and were limited to only using `run:` command steps.

However, sometimes I don't think it makes sense to create a Docker action. I don't necessarily like when I see simple actions calling a REST API endpoint run as a Docker action due to some of the [limitations](#fn:footnote-1) and theoretical overhead of running a Docker container.

With that caveat out of the way, let's dive into creating Docker container actions, as well as how to use a private image registry to host the Docker image.

> Container jobs are different than Docker actions! If you are using a container job, there are ways to natively provide authentication. See [my post on container jobs](/posts/github-container-jobs) for more information!
{: .prompt-info }

## Docker Action Hosting Options

When [creating a Docker action](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action), you can either specify a local `Dockerfile`{: .filepath} (that means this has to be built every time the Action runs) or you can specify a **public** Docker registry like Docker Hub or GitHub Container Registry. The [SonarQube Scan action uses a local `Dockerfile`{: .filepath}](https://github.com/SonarSource/sonarqube-scan-action/blob/d3ca1743de4293fc030a2e1ded1fb44088b80b76/action.yml#L9). Conversely, the [SILE Typesetter action uses a public GitHub Container Registry image](https://github.com/sile-typesetter/sile/blob/b2cc0841ff603abc335c5e66d8cc3c64b65365eb/action.yml#L10).

You'll note the key word here: **public**. If you want to use a private image registry, you'll need to authenticate to that registry. This is where things get a bit more complicated. If you pay attention to the workflow's logs, you'll notice that the Docker container actions are pulled or built as the first step of the job, in the initialization step. This runs before any of the steps in the job run. This is important to note because of course, you typically need to authenticate to a private image registry. But how can we provide authentication if we can't otherwise run a step before the initialization step?

I have been asked this by a few customers as well as [provided steps to implement this in a GitHub Discussion thread](https://github.com/orgs/community/discussions/45981), so I wanted to capture this in a blog post so that others can benefit from this knowledge.

## Connecting to a Private Image Registry

Somehow, we need to inject authentication before the job runs, or at least be able to dynamically provide credentials to the private image registry. There are a few ways to do this:

1. If you are running this in a self-hosted environment, you could pre-bake the Docker credentials onto the self-hosted runner host. This is not ideal because this could be a security risk if the runner host is compromised and sort of defeats the purpose of completely portal and ephemeral runner environments.
2. You could add a runner [pre-job step](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/running-scripts-before-or-after-a-job). This would only work on self-hosted runners, and the authentication mechanism between your runner and the secret store would still provide a challenge.
3. You could [customize with the container commands](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/customizing-the-containers-used-by-jobs#container-customization-commands) by using the [Runner Container Hooks](https://github.com/actions/runner-container-hooks). This also only works for self-hosted runners and has a moderate amount of setup required.
4. Use a composite action to wrap/hide and reference the Docker action as a [local action reference](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#runsstepsuses). The local action then references the real Docker container action that needs authentication to the private image registry. This would work with GitHub-hosted runners and self-hosted runners!

I'll be going over the later two options in detail in the next sections.

### Using the Container Customization Commands

The [container customization commands](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/customizing-the-containers-used-by-jobs#container-customization-commands) are really powerful, and if we're using self-hosted runner infrastructure, this can be a great way to inject credentials into the runner environment. Specifically, we can use the [`run_container_step`](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/customizing-the-containers-used-by-jobs#run_container_step) hook that is called once for every container action in the job. This method uses [Runner Container Hooks](https://github.com/actions/runner-container-hooks).

Here is an example of running container hooks. I am only [adding logging](https://github.com/search?q=repo%3Ajoshjohanning-org%2Factions-runner-container-hooks-tracer%20HOOK%3A&type=code) to show how the hooks are called. You would need to add the logic to authenticate to your private image registry.

![Using container hooks to customize container commands](container-hooks-logs-example.png){: .shadow }
_Using container hooks to customize container commands_

To set this up, you need:

1. Follow the [instructions in the repository](https://github.com/joshjohanning-org/actions-runner-container-hooks-tracer/blob/main/README.md#running) to set this up on your self-hosted runner
  - Clone the repository to your self-hosted runner
  - Run the `npm install` and `npm run build` commands in the designated folders in the repository
  - Set a `.env`{: .filepath} file with the `GITHUB_ACTIONS_RUNNER_HOOKS` variable set to the absolute path of the `./packages/docker/lib/index.js`{: .filepath} file
2. The next time you run a container Action, you should see similar logs to what I have above!

> Credits to my co-worker, [@tspascoal](https://pascoal.net/), for much of the help on this method!
{: .prompt-info }

### Using a Composite Action and a Local Action Reference

We can use a composite action (or, several composite actions I should say) to reference a Docker container action behind a local action reference. I admit that this is a bit of a hack, but it works, and it works with both GitHub-hosted and self-hosted runners! The idea is that at workflow compile time, Actions doesn't know about the Docker action since it is only being referenced locally from within a composite action. Therefore, it doesn't authenticate to the image registry / pull the image until the composite action is rendered and we can authenticate to the private image registry ahead of time accordingly.

You can't simply use a composite action to reference a Docker action normally, Actions is smart enough to render the Docker action as a Docker action and attempt to pull the image before the job starts. The action referenced by the composite action must be a local action reference. Also, the local action itself cannot be a Docker action, it still needs to reference the Docker action separately (AKA, in my tests, a minimum of 2 composite actions are needed here).

Let's set this up! In my example, I am storing a Docker action in a private Azure Container Registry.

> Note: My [repository](https://github.com/joshjohanning-org/composite-action-private-registry-docker-actions) has 2 examples, so you'll notice that I have a subfolder `nested-composite-action-better`{: .filepath}. You don't have to do it this way; `nested-composite-action-better/action.yml`{: .filepath} could live in the root of the repo as `./action.yml`{: .filepath} The `get-path/action.yml`{: .filepath} and the `3-action-implementation/action.yml`{: .filepath} files have to live in subfolders since we can only have one `action.yml`{: .filepath} file in a directory. And technically, `get-path/action.yml`{: .filepath} could live in a separate repo altogether.
{: .prompt-info }

Before diving into the YML, and with the note above in mind, let's take a look at the file structure of the core components of this solution as I think it will make slightly easier to understand:

```text
joshjohanning-org/call-private-registry-docker-actions@main/ # "app" repo in this scenario
└── .github/
    └── workflows/
        └── my-awesome-workflow-testing-docker-private-action.yml

joshjohanning-org/composite-action-private-registry-docker-actions@main/ # orchestration repo
└── nested-composite-action-better/
    ├── 1-action/
    │   └── action.yml
    ├── 2-get-path/
    │   └── action.yml
    └── 3-action-implementation/
        └── action.yml

joshjohanning-org/simple-docker-action@main/ # Docker container action repo
└── private/
    └── action.yml
```
{: .nolineno}

I've broken up the components into three separate composite actions and the one Docker container action:

1. The main orchestration composite action (called by `my-awesome-workflow-testing-docker-private-action.yml`{: .filepath} in the example above)
2. The get-path action that retrieves the dynamic local file path (the actions branch/tag/ref is part of the file path) (this action could live in a separate repository)
3. The local action that references the Docker container action in another repository
4. In a separate repository (requirement), the Docker container action (`joshjohanning-org/simple-docker-action/private@main`{: .filepath} in the example above)

For workflows referencing this solution, they would reference the `1-action/action.yml`{: .filepath} action [as such](https://github.com/joshjohanning-org/call-private-registry-docker-actions/blob/main/.github/workflows/ci-7-nest-composite-action-in-remote-composite-action-better.yml):

{% raw %}
```yml
name: ci-7-nest-composite-action-in-remote-composite-action-better
run-name: nest composite action in remote composite action better

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: simple-docker-action
        uses: joshjohanning-org/composite-action-private-registry-docker-actions/nested-composite-action-better/1-action@main
        with:
          password:  ${{ secrets.ACR_PASSWORD }}
```
{: file='.github/workflows/my-awesome-workflow-testing-docker-private-action.yml'}

Now, here are the rest of the workflow files as promised.

This is the [main orchestration composite action](https://github.com/joshjohanning-org/composite-action-private-registry-docker-actions/blob/main/nested-composite-action-better/1-action/action.yml), and the one that users would reference directly in their workflows:

```yml
name: 'private registry docker action test'
description: 'private registry docker action test'
inputs:
  password:  
    required: true
    description: acr password
runs:
  using: "composite"
  steps:
    - name: back up docker creds
      run: cp ${{ runner.workspace }}/../../.docker/config.json ${{ runner.workspace }}/../../.docker/config.json.bak
      id: docker-backup
      shell: bash
    - name: auth to acr
      run: echo ${{ inputs.password }} | docker login --username test123j --password-stdin test123j.azurecr.io
      shell: bash

    # get and output path to composite action
    - uses: joshjohanning-org/composite-action-private-registry-docker-actions/nested-composite-action-better/2-get-path@main
      id: composite-path

    # copy composite action to tmp folder - the action-implementation folder is where you define what docker action to reference
    - run: |
        mkdir -p __tmp
        cp -r '${{ steps.composite-path.outputs.path }}/3-action-implementation' __tmp/action-implementation
      shell: bash

    # run the local composite action w/ docker private registry
    - uses: ./__tmp/action-implementation

    - run: rm -rf __tmp/action-implementation
      name: cleanup action
      shell: bash
      if: always()

    - run: mv ${{ runner.workspace }}/../../.docker/config.json.bak ${{ runner.workspace }}/../../.docker/config.json
      name: restore original docker creds
      shell: bash
      if: always() && steps.docker-backup.conclusion == 'success'

    - run: cat ${{ runner.workspace }}/../../.docker/config.json
      name: print docker registries
      shell: bash
      if: always() && steps.docker-backup.conclusion == 'success'
```
{: file='nested-composite-action-better/action.yml'}

This is an action that gets the dynamic file path to the local composite action (this action doesn't technically have to exist in the same repository, unlike the others, but it could). Since actions are cloned during the job initialization step, we can take advantage of this so that we don't have to provide any additional Git authentication to retrieve these files. Tiago [talks about this trick more in-depth here](https://pascoal.net/2023/12/14/gha-accessing-content-from-other-repos-1/)!


```yml
name: 'private registry docker action test - get path'
description: 'private registry docker action test - get path'
outputs:
  path:
    description: action path
    value: ${{ steps.get-path.outputs.path }}
runs:
  using: "composite"
  steps:
    - run: echo 'path=${{ github.action_path }}/..' >> $GITHUB_OUTPUT
      id: get-path
      shell: bash
```
{: file='nested-composite-action-better/2-get-path/action.yml'}

This is the 🪄 - the local action that then references an external Docker container action that is using a private image registry:

```yml
name: 'private registry docker action test - call the private docker action'
description: 'private registry docker action test - call the private docker action'
runs:
  using: "composite"
  steps:
    - uses: joshjohanning-org/simple-docker-action/private@main
```
{: file='nested-composite-action-better/3-action-implementation/action.yml'}

And finally, this is the [Docker action that is referencing a private container registry](https://github.com/joshjohanning-org/simple-docker-action/blob/main/private/action.yml):

```yml
name: 'Hello World'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The time we greeted you'
runs:
  using: 'docker'
  image: 'docker://test123j.azurecr.io/actions/simple-docker-action:1'
  args:
    - ${{ inputs.who-to-greet }}
```
{: file='private/action.yml'}
{% endraw %}

For the [workflow](https://github.com/joshjohanning-org/call-private-registry-docker-actions/actions/workflows/ci-7-nest-composite-action-in-remote-composite-action-better.yml) (e.g.: app repo) calling this, this is all that they would see:

![Using composite actions to call a docker container action hosted in a private image registry](docker-action-composite-action.png){: .shadow }
_Using composite actions to call a docker container action hosted in a private image registry_

😮‍💨 That's a lot of steps, I know! Ideally you can centrally store the `2-get-path/action.yml`{: .filepath}. You could also enhance this so that the `3-action-implementation/action.yml`{: .filepath} has the downstream Docker container action repository/ref dynamically injected so you don't have to create this same exact structure for every Docker action you want to reference a private image registry. You could use bash to modify the referenced action in `3-action-implementation/action.yml`{: .filepath} before it's called, thus only requiring one copy of the main orchestration bits. But the concept is at least proven out with the above example!

> Thanks to my co-worker, [@tspascoal](https://pascoal.net/), for a ton of help on this method as well!
{: .prompt-info }

> I do have another example of this [here](https://github.com/joshjohanning-org/composite-action-private-registry-docker-actions/blob/main/nested-composite-action/action.yml) that only uses two composite actions as opposed to three composite actions like I use in the example above. It would work just as well, it just isn't as "fancy" as the example above in setting an output parameter and ensuring proper handling of the Docker credentials. Feel free to borrow upon this idea as well!
{: .prompt-tip }

## Summary

We sometimes forget that GitHub Actions serves both the opensource and enterprise community. Some consider it a missing feature or limitation that you can't authenticate to a private image registry for Docker container actions, but I can understand how the experience using marketplace actions would be hindered if some actions had a requirement to authenticate to a private image registry before running. However, with a little creativity, we can still use Docker container actions with a private image registry without compromising too much on the user experience and security of your workflows.

If you were only using self-hosted runners with no possibility of ever using GitHub-hosted runners, customizing the container commands is probably the best way to go. But if you want to use GitHub-hosted runners and be able to take advantage of the benefits of not having to maintain your own runner infrastructure, the composite action method is your best bet.

And remember, sometimes Docker container actions are the exact right tool for the job, but sometimes they introduce unnecessary complexity and overhead using containers for container's sake to run simple commands.

Let me know if you have any feedback or suggestions in the comments ⬇️. Happy Actioning!! 🚀 🚢

## Footnotes

[^footnote-1]: Docker container Actions only work on Linux runners, not Windows or macOS runners. Docker container Actions also don't work on Actions-Runner-Controller using Kubernetes mode (only Docker-in-Docker mode).
