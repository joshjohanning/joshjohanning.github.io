---
title: 'Docker Container Jobs in GitHub Actions'
author: Josh Johanning
date: 2022-03-07 13:30:00 -0600
description: Getting started using a Docker Container to run your GitHub Actions Job, tips and tricks, troubleshooting, and caveats
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Docker, Containers, Actions Runner Controller]
media_subpath: /assets/screenshots/2022-03-07-github-container-jobs
image:
  path: container-job-post-image.png
  width: 100%
  height: 100%
  alt: Container Job in GitHub
---

## Overview

GitHub Actions has a relatively little known feature where you can [run jobs in a container](https://docs.github.com/en/actions/using-jobs/running-jobs-in-a-container), specifically a Docker container. I liken it to delegating the entire job to the container, so every step that would run in the job will instead run in the container, including the clone/checkout of the repository. This generally works well, but there are some tips and tricks that I can pass along that may be helpful, especially if running in a self-hosted runner scenario.

Why would you want to use a container job, you may ask? Well imagine you have a Python application that uses a specific version of Python. Okay, simple enough, we can just use the [Setup Python](https://github.com/marketplace/actions/setup-python) action to install the right version. But what if we also require a specific / non-standard version of Node? And MySQL? We could use a script and install all our prerequisites using `apt install`, but this takes time. Over dozens of CI jobs, the extra minutes add up and you might even run against the cap of your limit. So instead, we can use a container job that has all the prerequisites our application needs to build / run already pre-installed.

I won't really be covering it, but there is also the option to run a [service container alongside your job](https://docs.github.com/en/actions/using-containerized-services/about-service-containers). This would be useful if running tests against a containerized copy of a database, for example. The documentation uses Redis as an example. Similar with [Docker Container Actions](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action).

## Caveats

Usually, I put the caveats after the implementation, but there are enough important ones here to lead with. If none of these apply, head to the [Implementation](#implementation) section.

- Containers in GitHub Actions, including [Container Jobs](https://docs.github.com/en/actions/using-jobs/running-jobs-in-a-container), [Service Containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers), and [Docker Container Actions](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action) only work on **Linux** runners - they will not run on Windows runners 😔
    + Some Marketplace actions, such as [Checkmarx](https://github.com/marketplace/actions/checkmarx-cxflow-action), are Docker Container Actions, therefore they won't run on Windows

![Container jobs/actions can't run on windows, macos](container-action-only-windows.png){: .shadow }
_Container jobs/actions can't run on Windows or MacOS_

- If you are using Docker to run the runner without doing the docker-in-docker magic, you might see an error - but if you are using something like [actions-runner-controller](https://github.com/actions-runner-controller/actions-runner-controller), this is a non-issue
    + Error mentioned in this [issue](https://github.com/actions/runner/issues/367#issuecomment-597742895)

![Container jobs/actions can't run within another container](container-cant-run-in-container.png){: .shadow }
_Container jobs/actions can't run within another container unless you have docker-in-docker setup_

- You cannot override the working directory that gets mapped in - the `/_work/`{: .filepath} directory on the host is mapped to `/__w/`{: .filepath} in the container
    + This is only a problem if you had intended to use an alternative work directory with permissions already set up in the container
    + We can pass in [additional options](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idcontaineroptions) to use in GitHub for the container job, but Docker our options get added to the end of the original Docker command and subsequent `--workdir` options are ignored
    + Mentioned in this [issue](https://github.com/actions/runner/issues/878)
    + Relevant errors:
        > ```
        > /usr/bin/git init /__w/container-job-test/container-job-test
        > /__w/container-job-test/container-job-test/.git: Permission denied
        > Error: The process '/usr/bin/git' failed with exit code 1
        > ```

        and
        
        > ```
        > Deleting the contents of '/__w/container-job-test/container-job-test'
        > Error: Command failed: rm -rf "/__w/container-job-test/container-job-test/.git"
        > rm: cannot remove '/__w/container-job-test/container-job-test/.git': Permission denied
        > ```

    + A fix was to `chmod` the `/_work/`{: .filepath} directory on the **host** to [work around this permissions issue](https://github.com/actions/runner/issues/878#issuecomment-1030686369)

- The default shell for `run` steps inside a container is `sh` instead of `bash`. This can be overridden with [`jobs.<job_id>.defaults.run`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_iddefaultsrun) or [`jobs.<job_id>.steps[*].shell`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsshell).
    + This is important because bashisms, such as if statements that contain `[[ ]]`, will not work in a `sh` script
    + As an example, you might see an `[[: not found` error when running the container job that works when not running as a container job

## Implementation

The full syntax for using container jobs, such as specifying ports, volumes, and options, is available [here](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idcontainer).

In [this example](https://github.com/joshjohanning/container-job-test), I am building my own Docker image, publishing to the repository, and using the image in a subsequent workflow as a container job, shown below:

{% raw %}

```yml
name: Container Job

on:
  push:
    branches:
      main

env:
  test: value

jobs:  
  container:
    runs-on: ubuntu-latest
    container:
      # image: 'ubuntu:20.04' # can also use this to test
      image: 'ghcr.io/${{ github.repository }}:latest'
      credentials:
         username: ${{ github.ref }}
         password: ${{ secrets.GITHUB_TOKEN }}
      env: 
        actor: ${{ github.actor }}
        testjob: here is value

    steps:
    - uses: actions/checkout@main
    - name: run ls
      run: ls
    # all these below work
    - name: print actor env var
      run: echo "$actor"
    - name: print directly github context
      run: echo "${{ github.actor }}"
    - name: print repo secret
      run: echo "${{ secrets.TEST_SECRET }}"
    - name: print job env var
      run: echo "$testjob"
    - name: print root env var
      run: echo "$test"
    - name: condition with root env var
      if: ${{ env.test == 'value' }}
      run: echo "$test" 
```
{: file='.github/workflows/container-job.yml'}

{% endraw %}

When you run the workflow, you will notice an additional log entry to initialize the container. Subsequently, all steps in the job will run inside of the container:

![Container job](container-job.png){: .shadow }
_Successful container job_

For those wondering, here is what a sample `DOCKERFILE`{: .filepath} for this [looks like](https://github.com/joshjohanning/container-job-test/blob/main/Dockerfile) - hint there's nothing special. You can also test this with good 'ol `ubuntu:20.04` (or `ubuntu:latest`).

Perhaps more interestingly, my workflow for publishing my Docker image is [here](https://github.com/joshjohanning/container-job-test/blob/main/.github/workflows/docker-image.yml#L34). 

## Summary

I've used Container Jobs in Azure DevOps before, and I was excited to see we had similar functionality in GitHub! This can be much more practical than writing a large script to `apt install` everything for each job run. Just note some of the [caveats](#caveats), most of which only apply to self-hosted and non-Linux runners. 

You can take this to the next step, instead of running your jobs in containers, you could additionally run your runners in containers using something like [actions-runner-controller](https://github.com/actions-runner-controller/actions-runner-controller). actions-runner-controller, which is running a runner in a container in k8s, supports running container jobs with Docker actions no problem! 
