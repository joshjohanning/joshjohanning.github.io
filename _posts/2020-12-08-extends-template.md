---
title: 'Azure DevOps: Extends Template with Build and Deployment Templates'
author: Joshua Johanning
date: 2020-12-10 22:00:00 -0600
categories: [Azure DevOps, Pipelines]
tags: [azure devops, extends, templates]
---

## Scenario

I had a client that wanted to integrate a secret scanning utility (among other checks) into the pipeline, and enforce this control. Colin Dembovsky (my co-worker) and I typically recommend creating and referencing job templates for each environment. Job templates are very flexible, allowing for re-use across an organization but still allowing for differences between applications through the parameters passed into the template.

A very abbreviated example of this would look like:

```yaml
resources:
  repositories:
  - repository: templates
    type: github
    name: soccerjoshj07/pipeline-templates
    endpoint: soccerjoshj07

stages:
- stage: 'Build'
  jobs: 
  - template: build.yml@templates
    parameters:
      buildConfiguration: 'release'
- stage: deployDev
  jobs: 
  - template: deploy.yml@templates
    parameters:
      environment: 'dev'
- stage: deployProd
  jobs: 
  - template: deploy.yml@templates
    parameters:
      environment: 'prod'
```

However, there is nothing here that *enforces* a developer to use these templates - they could either write their own or just create their pipeline inline. This is where [Extends](https://docs.microsoft.com/en-us/azure/devops/pipelines/security/templates?view=azure-devops#use-extends-templates) comes into play!

I remember when this was first announced in a [sprint release note](https://docs.microsoft.com/en-us/azure/devops/release-notes/2019/sprint-162-update#use-extends-keyword-in-pipelines) in December 2019, we tried it and couldn't really get it to work the way we wanted to. But with the client's requirements, this seemed like a perfect time to give it another shot. I wanted to reference a separate configuration repository where the secret scanning config would be stored without the developer having to worry or care about it, and we found a way to do just that using Extends.

## The Code

In this demo scenario, my code is stored in GitHub, but this could work just as well with code in Azure Repos as well.

### [`azure-pipelines.yml`](https://github.com/soccerjoshj07/secrets-scanning-poc/blob/f07a2eae415f38933f506d4ed0c69f75df2ffb91/azure-pipelines.yml)

In the root `azure-pipelines.yml` file, you'll notice that the `extends` keyword is at the same level as `trigger` and `resources`. This was the tricky part - how does one use `extends` AND job templates? The approach is to use a *steps* template for the build where we want the extra steps injected, and for deployment we can use our *job* templates like normal. We will add an Environment check that ensures that the extends template is being used. If the Extends template isn't used, the check fails and the deployment isn't allowed.

The deployment stages and jobs are defined in this file as well - this should look very familiar to regular deployment jobs except that they are being referenced as a parameter.

{% raw %}
```yaml
trigger:
  - main
  
resources:
  repositories:
  - repository: templates
    type: github
    name: soccerjoshj07/pipeline-templates
    endpoint: soccerjoshj07

extends:
  template: secret-scanning/secret-scanning-extends.yml@templates
  parameters:
    buildSteps:
    # use steps template for build
    - template: secret-scanning/sample-build-steps.yml@templates
      parameters:
        whatToBuild: 'Hello world'
    deployStages:
    - stage: dev
      displayName: deploy to dev
      jobs: 
      # use job templates as normal for deployment
      # bug using github as template repo?: must use ../
      - template: ../secret-scanning/sample-deployment-job.yml@templates
        parameters:
          environment: github-secret-scanning-test-gate-dev
    - stage: prod
      displayName: deploy to prod
      jobs:
      - template: ../secret-scanning/sample-deployment-job.yml@templates
        parameters:
          environment: github-secret-scanning-test-gate-prod
```

### [`secret-scanning-extends.yml`](https://github.com/soccerjoshj07/pipeline-templates/blob/master/secret-scanning/secret-scanning-extends.yml)

The `parameters` passed into the extends template include a `stepList` type for the `buildSteps` and a `stageList` for the `deployStages`.

The `resource` and `- template: secret-scanning-steps.yml` here is the configuration repository I was mentioning before. For your use case, you may not need this, you would just need the steps in `- ${{ parameters.buildSteps }}`.

The build stage and job is defined in this file.

```yaml
parameters:
- name: buildSteps # the name of the parameter is buildSteps
  type: stepList # data type is StepList
  default: [] # default value of buildSteps
- name: deployStages
  type: stageList
  default: [] 

resources:
  repositories:
  - repository: secretscanning
    type: github
    name: soccerjoshj07/secret-scanning-config
    endpoint: soccerjoshj07

stages:
- stage: secure_buildstage
  jobs:
  - job: secure_buildjob
    steps:

    - template: secret-scanning-steps.yml

    - ${{ parameters.buildSteps }}

- ${{ parameters.deployStages }}
```

### [`sample-build-steps.yml`](https://github.com/soccerjoshj07/pipeline-templates/blob/master/secret-scanning/sample-build-steps.yml)

Not much crazy here - this is a *steps* template (as opposed to a *job* template). This is injected into the extends template in the `- ${{ parameters.buildSteps }}` line of code.

```yaml
parameters:
  whatToBuild: ''

steps:
- script: |
    echo "${{ parameters.whatToBuild }}, here is where I do my build!"

```

### [`sample-deployment-job.yml`](https://github.com/soccerjoshj07/pipeline-templates/blob/master/secret-scanning/sample-deployment-job.yml)

This is pretty vanilla as well - this *job* template is injected into the extends template in the `- ${{ parameters.deployStages }}` line of code.


```yaml
parameters:
  environment: ''
  pool: 
    vmImage: 'ubuntu-latest'
jobs:
- deployment: deploy
  displayName: deploy
  pool: ${{ parameters.pool }}
  environment: ${{ parameters.environment }}
  strategy:
    runOnce:
      deploy:
        steps:
        - script: |
            echo "deploy hello world"
          displayName: deploy
```
{% endraw %}
## Configuration in Azure DevOps

The **Required YAML Template** check is added to the environment just as an Approval would be:

![Azure DevOps required check](/assets/screenshots/2020-12-08-extends-template/required-check.png)

*Note here if you are storing the code in Azure Repos - the example in this screenshot mentions `project/repository-name`. If the repository is in the same project, DO NOT include the project name in the path otherwise it won't work.*

Now, if you try to deploy to an environment while not using this extends template, it fails:
![failed stage](/assets/screenshots/2020-12-08-extends-template/failed-stage.png)

If you click the 0/1 checks passed, it shows the check that failed and hyperlinks to the checks for that environment:
![failed check](/assets/screenshots/2020-12-08-extends-template/failed-check.png)

Once you properly use the extends template - success!
![successful stage](/assets/screenshots/2020-12-08-extends-template/successful-stage.png)

## Conclusion and Next Steps

This 'Required Checks' concept works really well for environments that are defined ahead of time as a way to manage logical groupings of like deployments and add manual approval points.

Did you know that you can also add these same type of required checks on Service Connections? Yep! Therefore, you can configure your production Azure Service Connection such that ONLY certain users can make the approval, irregardless of how the environment and pipeline is set up.

Completing this circle, we can ensure that protected resources in Azure DevOps - environments AND service connections - extend from a particular template, ensuring compliance and standardization across the organization.

Happy templating!
