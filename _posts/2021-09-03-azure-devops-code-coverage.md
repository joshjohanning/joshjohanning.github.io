---
title: 'The Easiest Way to Generate and Publish .NET Code Coverage in Azure DevOps'
author: Josh Johanning
date: 2021-09-03 4:00:00 -0500
description: Publishing Code Coverage and making it look pretty in Azure DevOps is way harder than it should be
categories: [Azure DevOps, Pipelines]
tags: [Azure DevOps, Code Coverage]
image:
  path: /assets/screenshots/2021-09-03-azure-devops-code-coverage/good-code-coverage.png
  width: 816   # in pixels
  height: 582   # in pixels
  alt: Cobertura Code Coverage in Azure DevOps
---

## Overview

Publishing code coverage in Azure DevOps and making it look pretty is way harder than it should be. It's something that sounds simple, oh just check the box on the task - but nope you have to make sure to read the notes and add the additional parameter to the test task. Okay great, now you have a code coverage tab, but what is this .coverage file and how do I open it? That's not very user friendly. And don't get me started on having to wait for the *entire* pipeline to finish before you can even *see* the code coverage tab - nonsensical.

If you want to navigate to the solution, [scroll down](#the-better-way). 

## Not Good: The Out of the Box Way

If using the out of the box `dotnet` task with the `test` command, simply add the `publishTestResults` argument (or if using the task assistant, check the `Publish test results and code coverage` checkbox):
![adding dotnet test task](/assets/screenshots/2021-09-03-azure-devops-code-coverage/adding-test-task.png){: width="350" }{: .shadow }
_.NET Core Task Assistant UI_

However, if you read the information on the `publishTestResults` argument from the [.NET Core CLI task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops#arguments) (or clicking on the `(i)` on the `Publish test results and code coverage` option in the task assistant), it says:

> Enabling this option will generate a test results TRX file in `$(Agent.TempDirectory)` and results will be published to the server.
> This option appends `--logger trx --results-directory $(Agent.TempDirectory)` to the command line arguments.
>
> **Code coverage can be collected by adding `--collect "Code coverage"` option to the command line arguments. This is currently only available on the Windows platform.**

Emphasis: mine. So even if you check the box, you need to ensure you add the `--collect "Code coverage"` argument. Oh, and you have to run this on a Windows agent, so no `ubuntu-latest` for us.  

This produces code coverage that looks like the following in Azure DevOps:
![.coverage file code coverage](/assets/screenshots/2021-09-03-azure-devops-code-coverage/bad-code-coverage.png ){: width="400" }{: .shadow }
_How the default code coverage in Azure DevOps looks_

It's a link to a .coverage file..which is great if you 1) have Visual Studio installed and 2) are on Windows (can't open .coverage file on Mac).

## The Better Way

The better way is to add the [coverlet.collector](https://github.com/coverlet-coverage/coverlet) NuGet package to (each of) the test project(s) that you run `dotnet test` against.

The easiest way to do this is to run the `dotnet package add` command targeting the test project:

`dotnet add <TestProject.cspoj> package coverlet.collector`

![dotnet add package](/assets/screenshots/2021-09-03-azure-devops-code-coverage/dotnet-add-package.png){: .shadow }
_Adding coverlet.collector with dotnet add package_

For those who can't run the `dotnet` command, add the [following](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/commit/43067b4e035eb45899e185e701bd4aaf8575514b) under the `ItemGroup` block in the `.csproj`{: .filepath} file:

```xml
    <PackageReference Include="coverlet.collector" Version="3.1.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
```

See the [following page](https://www.nuget.org/packages/coverlet.collector/) for the latest version.

Next, update the pipeline to ensure your `dotnet test` command looks like mine, and adding the `PublishCodeCoverageResults@1` task.

```yml
    # Add coverlet.collector nuget package to test project - 'dotnet add <TestProject.cspoj> package coverlet.collector'
    - task: DotNetCoreCLI@2
      displayName: dotnet test
      inputs:
        command: 'test'
        projects: '**/*.Tests.csproj'
        arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'
        publishTestResults: true

    # Publish code coverage report to the pipeline
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: Cobertura
        summaryFileLocation: $(Agent.TempDirectory)/*/coverage.cobertura.xml # using ** instead of * finds duplicate coverage files
```

The `--collect:"XPlat Code Coverage"` argument is what tells `dotnet test` to use the `coverlet.collector` package to generate us a cobertura code coverage report. As you can guess by the `XPlat` in the argument, this runs cross platform on both Windows and Ubuntu.

This argument creates a `$(Agent.TempDirectory)/*/coverage.cobertura.xml`{: .filepath} code coverage report file. This folder is default output folder since Azure DevOps adds `--results-directory /home/vsts/work/_temp` to the command.

Next, we have to specifically add the `PublishCodeCoverageResults@1` task to publish the code coverage output to the pipeline. It seems like at least with my project, it produces 2 `coverage.cobertura.xml`{: .filepath} files and that throws a warning in the pipeline, so that's why I used `$(Agent.TempDirectory)/*/coverage.cobertura.xml`{: .filepath} and NOT `$(Agent.TempDirectory)/**/coverage.cobertura.xml`{: .filepath}.

![duplicate code coverage reports](/assets/screenshots/2021-09-03-azure-devops-code-coverage/find-code-coverage.png){: .shadow }
_Duplicate coverage.cobertura.xml code coverage results_

Now, after the *entire* pipeline has finished (including any of the deployment stages), we will have a code coverage tab with a way more visually appealing code coverage report:
![cobertura code coverage in azure devops](/assets/screenshots/2021-09-03-azure-devops-code-coverage/good-code-coverage.png){: .shadow }
_Cobertura Code Coverage Report in Azure DevOps_

> Here's the [full sample YML pipeline](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/blob/main/azure-pipelines.yml#L36-L55) for this example.
{: .prompt-tip }

## Why not ReportGenerator?

I've seen many blog posts that are similar to mine, except that they have the `reportgenerator@4` task. I used to think this was required, too! But I have recently found out it is not - at least not if you have **ONE** test project you are publishing results for.

If you have multiple test projects you would like code coverage published for, then yes, the `reportgenerator@4` task is needed.

I have started to use the command line instead of the actual `reportgenerator@4` task itself, though, as not every organization has the [marketplace extension installed](https://marketplace.visualstudio.com/items?itemName=Palmmedia.reportgenerator) (and some have stricter policies about adding extensions than others). Additionally, if you use the CLI, you have more control over which version of ReportGenerator is ultimately used.

```yml
    # First install the tool on the machine, then run it
    - script: |
        dotnet tool install -g dotnet-reportgenerator-globaltool
        reportgenerator -reports:$(Agent.WorkFolder)/**/coverage.cobertura.xml -targetdir:$(Build.SourcesDirectory)/CodeCoverage -reporttypes:'HtmlInline_AzurePipelines;Cobertura'
        # IMPORTANT - set `disable.coverage.autogenerate` to true if you want to use older reportgenerator version
        echo "##vso[task.setvariable variable=disable.coverage.autogenerate;]true"
      displayName: Create code coverage report
```

`reportgenerator@4` equivalent:

```yml
      # ReportGenerator extension to combine code coverage outputs into one
      - task: reportgenerator@4
        inputs:
          reports: '$(Agent.WorkFolder)/**/coverage.cobertura.xml'
          targetdir: '$(Build.SourcesDirectory)/CoverageResults'
```

One of these steps needs to run before the `PublishCodeCoverageResults@1` task, and that task needs to be updated just a little bit (the xml file path is different). See:

```yml
      # Publish the combined code coverage to the pipeline
      - task: PublishCodeCoverageResults@1
        displayName: 'Publish code coverage report'
        inputs:
          codeCoverageTool: 'Cobertura'
          summaryFileLocation: '$(Build.SourcesDirectory)/CoverageResults/Cobertura.xml'
          reportDirectory: '$(Build.SourcesDirectory)/CoverageResults'
```

> Here's the [full sample YML pipeline](https://github.com/joshjohanning/PrimeService-unit-testing-using-dotnet-test/blob/reportgenerator-v4.6.1/azure-pipelines.yml#L47-L62) for this example calling `ReportGenerator` ourselves.
{: .prompt-tip }

## Code Coverage Tab Not Showing Up?

This doesn't really solve the [problem](https://developercommunity.visualstudio.com/t/code-coverage-does-not-show-up-until-multistage-pi/786733) where Azure DevOps will not show the Code Coverage tab until the *entire* pipeline (including all deployments) has completed. In cases where code coverage is important, I either:

1. Change the report output from `$(Build.SourcesDirectory)` to `$(Build.ArtifactStagingDirectory)` and [publish](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema#publish) the report as a pipeline artifact.
1. Create a *separate* build pipeline than deployment pipeline. This harkens back to the day where we didn't have multi-stage YAML builds and we had separate Build Definitions and Release Definitions. We can set up the deployment yml pipeline to be [triggered](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/pipeline-triggers?tabs=yaml&view=azure-devops#configure-pipeline-resource-triggers) from the completion of the build yml pipeline.

## Conclusion

Now you know the ins and outs of adding Code Coverage to your .NET (Core) projects in Azure DevOps.

Stay tuned to my next post on what we can do with [code coverage in GitHub Actions](/posts/github-code-coverage/)!
