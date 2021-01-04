---
title: 'GitHub: Block Pull Request if Code Scanning Alerts Are Found'
author: Joshua Johanning
date: 2020-12-16 20:00:00 -0600
categories: [GitHub, Pipelines]
tags: [GitHub, pull request, CodeQL]
---

## Overview

After virtually attending GitHub Universe last week and watching the [GitHub Advanced Security round-up](https://githubuniverse.com/GitHub-Advanced-Security-round-up/) and [Catching vulnerabilities early with GitHub](https://githubuniverse.com/Catching-vulnerabilities-early-with-GitHub/) sessions, it got me thinking: How do I block a pull request from being merged if the scans detect issues? I didn't think the [GitHub Docs](https://docs.github.com/en/free-pro-team@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/enabling-code-scanning-for-a-repository#understanding-the-pull-request-checks) were incredibly straight forward on how this works. 

I knew how to configure a branch protection rule in GitHub that enforces things such as a GitHub Action or Azure DevOps stage completes successfully, but what about code scanning? How configurable is it?

## How To

If you already have a Code Scanning alert setup, skip to step #7.

1. The first thing you need is a public repository or a private repository with the GitHub Advanced Security license purchased (as of Dec 2020)
1. In the Security tab on the repository, navigate to 'Code scanning alerts' page
1. I'm using the native 'CodeQL Analysis' workflow by GitHub - there are 3rd party analysis engines here too!
1. Take a look at the workflow file - I didn't need to make any changes, but one can modify the language matrix if you want/don't want scans to run for a particular language
1. There's an `Autobuild` step here that works most of the time, but for some repositories I found I had to build my app manually
1. Commit the workflow to your branch and merge it into Main - for best results we want an initial scan in the default branch before we test out the PR process
1. Under the Settings tab in the repository, navigate to Branches
1. Create a [rule](https://docs.github.com/en/free-pro-team@latest/github/administering-a-repository/enabling-required-status-checks) for your default branch - check the 'Require status checks to pass before merging' box
1. If you used the GitHub CodeQL workflow, check the `CodeQL` status check and save the rule (Note 1/4/20: I have noticed that the `CodeQL` won't appear as an option until you initiate at least one PR on the repository that triggers and completes the `Analyze (csharp)` job)

This last step was the part I wasn't clear on from the GitHub docs. The other entry, such as `Analyze (javascript)`, is only the *scan* job for that language. It should succeed irregardless of if vulnerabilities are found. If it fails, the `autobuild` task might not be able to compile your code.

After configuring the code scanning workflow, you should be all set!

## Testing It Out

Another thing the GitHub Docs do not do a good job of spelling out is that only *Errors* are going to fail the Pull Request status check. *Warnings*, out of the box, do not block the PR.

Alright, so let's introduce an error...does anyone know of an easy vulnerability we can put in our code? Well neither do I, but we don't have to with the help of our friend, the [Semmle vulnerability database](https://web.archive.org/web/20200929073843/https://help.semmle.com/wiki/label/js/path-problem) (Note: The day before I wrote this, this link started redirecting to GitHub, but I found an archive.org link to use for the purposes of this demo).

I'm going to use an incorrect suffix vulnerability. The easiest way to introduce this is to: 1) make sure `javascript` is in the language matrix in our CodeQL workflow like so: `language: [ 'csharp', 'javascript' ]` and 2) check in a simple `.js` file somewhere in the repository with the bad code:

```javascript
function endsWith(x,y) {

  let index = x.lastIndexOf(y);
  return x.lastIndexOf(y) === x.length - y.length;

}
```

Make sure to commit this in a branch because we want to test out the PR flow!

Afterwards, create the PR and wait for the job to run:

![pr](/assets/screenshots/2020-12-16-github-codeql-pr/pr.png)

Success! Or, failure, just as we wanted. For me, the fact that GitHub gives this away for free for all public repositories is incredible! There is a charge for the Enterprise, but the setup is so simple and integrations so robust - and we're only scratching the surface - makes it well worth it.

## Extra Credit - Fine Tuning

Okay, what if we were to have a repository with Terraform code and used the ShiftLeft Analysis marketplace code scanning workflow? Or, we used the native GitHub CodeQL workflow but want it to block merges when it finds any result, including warnings?

Well, in the case of the ShiftLeft Analysis workflow, there is a [config file](https://slscan.io/en/latest/integrations/tips/#config-file) that can be uploaded to the root of the repository to define some of this, but I haven't played around much for this. For the GitHub CodeQL workflow, there is no fine-tuning configuration file that we can easily use (that I know of at this date).

For this, I wrote a script that examines the `.sarif` and added another job to the workflow, like so:

{% raw %}
```yaml
  Detect-Errors:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        language: [ 'csharp', 'javascript' ]
    needs:
      - analyze
    steps:
    - name: Download Sarif Report
      uses: actions/download-artifact@v2
      with:
        name: sarif-report

    - name: Detect Errors
      run: |
        repo=$(echo ${{ github.repository }} | awk -F'/' '{print $2}')
        results=$(cat $repo/results/${{ matrix.language }}-builtin.sarif | jq -r '.runs[].results[].ruleId')

        resultsArray=($results)
        
        echo "${resultsArray[*]}"

        errorCount=0
        warningCount=0
        noteCount=0

        for var in "${resultsArray[@]}"
        do
          severity=$(cat $repo/results/${{ matrix.language }}-builtin.sarif | jq -r '.runs[].tool.driver.rules[] | select(.id=="'$var'").properties."problem.severity"')
          echo "${var} | $severity"
          if [ "$severity" == "warning" ]; then let warningCount+=1; fi
          if [ "$severity" == "error" ]; then let errorount+=1; fi
          if [ "$severity" == "note" ]; then let noteount+=1; fi
        done

        echo ""
        echo "Error Count: $errorCount"
        echo "Warning Count: $warningCount"
        echo "Note Count: $noteCount"
        echo ""

        if (( $errorCount > 0 )); then
            echo "errors found - failing detect error check..."
            exit -1
        fi

        if (( $warningCount > 0 )); then
            echo "warnings found - failing detect warning check..."
            exit -1
        fi
```

This will fail if any findings were found, including warnings - modify the script as needed.

Note that we also have to add an `Upload Build Artifact` step in the `Analyze` job, like so:

```yaml
    - name: Upload Sarif Report to Workflow
      uses: actions/upload-artifact@v2
      with:
        name: sarif-report-${{ matrix.language }}
        path: /home/runner/work/**/*.sarif
```
{% endraw %}

Depending on the workflow, you may have to modify the `path` in the Upload task as well as the script. You can find out the relative path of the .sarif report by viewing the Actions' logs.

The entire workflow can be found in my [GitHub branch](https://github.com/soccerjoshj07/tailspin-spacegame-web-deploy/blob/2d4955b668ffde45a2f4ea6e742268a536249b27/.github/workflows/codeql-analysis.yml).

Because the .sarif produced by the ShiftLeft analysis is slightly different and by default *doesn't* fail the job even with *errors*, I created a different workflow you can use to block pull requests if errors or warnings are found - see for [example](https://github.com/soccerjoshj07/azdo-terraform-tailspin/blob/05151b64818db1c4cabf5aaf51f0024c779d81f5/.github/workflows/shiftleft-analysis.yml).

Now just like we did above, we can modify our branch rule to require the "Detect-Errors" job to finish successfully, as this job will run successfully if there are no errors/warnings.

![pr-detected-errors-job](/assets/screenshots/2020-12-16-github-codeql-pr/pr-detected-errors.png)

![pr-blocked](/assets/screenshots/2020-12-16-github-codeql-pr/pr-blocked.png)

I'm sure there is probably a better way to do this (using the API?). I know back in the LGTM / Semmle days, there was also a config file you could commit to the root of the repository to more precisely define rules. Either way, let me know in the comments if you have any other ideas or improvements!
