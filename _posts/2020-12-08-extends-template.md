---
title: 'Azure DevOps: Extends Template with Build and Deployment Templates'
author: Josh Johanning
date: 2020-12-10 22:00:00 -0600
categories: [Azure DevOps, Pipelines]
tags: [Azure DevOps, Pipeline Templates]
---

## Scenario

I had a client that wanted to integrate a secret scanning utility (among other checks) into the pipeline, and enforce this control. Colin Dembovsky (my co-worker) and I typically recommend creating and referencing job templates for each environment. Job templates are very flexible, allowing for re-use across an organization but still allowing for differences between applications through the parameters passed into the template.

A very abbreviated example of this would look like:

```yaml
resources:
  repositories:
  - repository: templates
    type: github
    name: joshjohanning/pipeline-templates
    endpoint: joshjohanning

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

However, there is nothing here that *enforces* a developer to use these templates - they could either write their own or just create their pipeline inline. This is where [Extends](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/templates?view=azure-devops#use-extends-templates) comes into play!

I remember when this was first announced in a [sprint release note](https://docs.microsoft.com/en-us/azure/devops/release-notes/2019/sprint-162-update#use-extends-keyword-in-pipelines) in December 2019, we tried it and couldn't really get it to work the way we wanted to. But with the client's requirements, this seemed like a perfect time to give it another shot. I wanted to reference a separate configuration repository where the secret scanning config would be stored without the developer having to worry or care about it, and we found a way to do just that using Extends.

## The Code

In this demo scenario, my code is stored in GitHub, but this could work just as well with code in Azure Repos as well.

### [`azure-pipelines.yml`{: .filepath}](https://github.com/joshjohanning/secrets-scanning-poc/blob/f07a2eae415f38933f506d4ed0c69f75df2ffb91/azure-pipelines.yml){: .filepath}

In the root `azure-pipelines.yml`{: .filepath} file, you'll notice that the `extends` keyword is at the same level as `trigger` and `resources`. This was the tricky part - how does one use `extends` AND job templates? The approach is to use a *steps* template for the build where we want the extra steps injected, and for deployment we can use our *job* templates like normal. We will add an Environment check that ensures that the extends template is being used. If the Extends template isn't used, the check fails and the deployment isn't allowed.

The deployment stages and jobs are defined in this file as well - this should look very familiar to regular deployment jobs except that they are being referenced as a parameter.

{% raw %}

```yaml
trigger:
  - main
  
resources:
  repositories:
  - repository: templates
    type: github
    name: joshjohanning/pipeline-templates
    endpoint: joshjohanning

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

### [`secret-scanning-extends.yml`{: .filepath}](https://github.com/joshjohanning/pipeline-templates/blob/main/secret-scanning/secret-scanning-extends.yml)

The `parameters` passed into the extends template include a `stepList` type for the `buildSteps` and a `stageList` for the `deployStages`.

The `resource` and `- template: secret-scanning-steps.yml`{: .filepath} here is the configuration repository I was mentioning before. For your use case, you may not need this, you would just need the steps in `- ${{ parameters.buildSteps }}`.

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
    name: joshjohanning/secret-scanning-config
    endpoint: joshjohanning

stages:
- stage: secure_buildstage
  displayName: 'Secure Build Stage'
  jobs:
  - job: secure_buildjob
    steps:

    - template: secret-scanning-steps.yml

    - ${{ parameters.buildSteps }}

- ${{ parameters.deployStages }}
```

### [`sample-build-steps.yml`{: .filepath}](https://github.com/joshjohanning/pipeline-templates/blob/main/secret-scanning/sample-build-steps.yml)

Not much crazy here - this is a *steps* template (as opposed to a *job* template). This is injected into the extends template in the `- ${{ parameters.buildSteps }}` line of code.

```yaml
parameters:
  whatToBuild: ''

steps:
- script: |
    echo "${{ parameters.whatToBuild }}, here is where I do my build!"

```

### [`sample-deployment-job.yml`{: .filepath}](https://github.com/joshjohanning/pipeline-templates/blob/main/secret-scanning/sample-deployment-job.yml)

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

## Configuration in Azure DevOps

The **Required YAML Template** check is added to the environment just as an Approval would be:

![Azure DevOps required check](/assets/screenshots/2020-12-08-extends-template/required-check.png){: .shadow }
_Adding the required template check to an environment_

*Note here if you are storing the code in Azure Repos - the example in this screenshot mentions `project/repository-name`. If the repository is in the same project, DO NOT include the project name in the path otherwise it won't work.*

Now, if you try to deploy to an environment while not using this extends template, it fails:
![failed stage](/assets/screenshots/2020-12-08-extends-template/failed-stage.png){: .shadow }
_Fails the required template check_

If you click the 0/1 checks passed, it shows the check that failed and hyperlinks to the checks for that environment:
![failed check](/assets/screenshots/2020-12-08-extends-template/failed-check.png){: .shadow }
_More details on the failed required template check_

Once you properly use the extends template - success!
![successful stage](/assets/screenshots/2020-12-08-extends-template/successful-stage.png){: .shadow }
_Passes the required template check_

## Conclusion and Next Steps

This 'Required Checks' concept works really well for environments that are defined ahead of time as a way to manage logical groupings of like deployments and add manual approval points.

Did you know that you can also add these same type of required checks on Service Connections? Yep! Therefore, you can configure your production Azure Service Connection such that ONLY certain users can make the approval, irregardless of how the environment and pipeline is set up.

Completing this circle, we can ensure that protected resources in Azure DevOps - environments AND service connections - extend from a particular template, ensuring compliance and standardization across the organization.

Happy templating!

## Updates - Jan 5 2020 - Using Build Job template instead of Build Steps template

After using the template for a few weeks, I've made an update to be able to pass in *build jobs* instead of *build steps*. Note that with this template, you can pass in either. I prefer using a separate job under the build stage for the secret scanning bit so I can see what failed - the secret scan or the build.

I have an input parameter for `buildStageName` so that this would still make sense in a scenario where there isn't a stage named Build, such as in Terraform deployments. By default, the stage will be named build, but it can be optionally overridden (called something like `secret-scanning` instead, in the Terraform example).

**secret-scanning-extends.yml:**

```yaml
parameters:
- name: buildSteps # the name of the parameter is buildSteps
  type: stepList # data type is StepList
  default: [] # default value of buildSteps
- name: buildJobs # the name of the parameter is buildSteps
  type: jobList # data type is StepList
  default: [] # default value of buildSteps
- name: deployStages
  type: stageList
  default: [] 
- name: 'buildStageName'
  type: string
  default: 'build'

resources:
  repositories:
  - repository: secretscanning
    type: github
    name: joshjohanning/secret-scanning-config
    endpoint: joshjohanning

stages:
- stage: ${{ parameters.buildStageName }}
  displayName: ${{ parameters.buildStageName }}
  jobs:
  - job: secret_scanning
    steps:

    - template: secret-scanning/secret-scanning-steps.yml

    - ${{ parameters.buildSteps }}

  - ${{ parameters.buildJobs }}

- ${{ parameters.deployStages }}

```

**azure-pipelines.yml:**

```yaml
trigger:
  - main
  
resources:
  repositories:
  - repository: templates
    type: github
    name: joshjohanning/pipeline-templates
    endpoint: joshjohanning

extends:
  template: secret-scanning/secret-scanning-extends.yml@templates
  parameters:
    buildJobs:
    # job template
    - template: my-build-job.yml@templates
    deployStages:
    - stage: dev
      displayName: deploy to dev
      jobs: 
      # job template
      - template: secret-scanning/sample-deployment-job.yml@templates
        parameters:
          environment: github-secret-scanning-test-gate-dev
    - stage: prod
      displayName: deploy to prod
      jobs:
      - template: secret-scanning/sample-deployment-job.yml@templates
        parameters:
          environment: github-secret-scanning-test-gate-prod
```

{% endraw %}
