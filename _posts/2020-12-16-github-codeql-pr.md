---
title: 'GitHub: Block Pull Request if Code Scanning Alerts Are Found'
author: Josh Johanning
date: 2020-12-16 20:00:00 -0600
categories: [GitHub, Advanced Security]
tags: [GitHub, GitHub Actions, Pull Requests, CodeQL, GitHub Advanced Security, Policy Enforcement, Branch Protection Rules]
image:
  path: /assets/screenshots/2020-12-16-github-codeql-pr/pr.png
  width: 918   # in pixels
  height: 390   # in pixels
  alt: Pull Request that is blocked because Code Scanning Alerts are found
---

## Overview

After virtually attending GitHub Universe last week and watching the [GitHub Advanced Security round-up](https://www.youtube.com/watch?v=T_-Tn81b4lc) and [Catching vulnerabilities early with GitHub](https://www.youtube.com/watch?v=l2epzyytPGE) sessions, it got me thinking: How do I block a pull request from being merged if the scans detect issues? I didn't think the [GitHub Docs](https://docs.github.com/en/free-pro-team@latest/github/finding-security-vulnerabilities-and-errors-in-your-code/enabling-code-scanning-for-a-repository#understanding-the-pull-request-checks) were incredibly straight forward on how this works.

I knew how to configure a branch protection rule in GitHub that enforces things such as a GitHub Action or Azure DevOps stage completes successfully, but what about code scanning? How configurable is it?

## How To

If you already have a Code Scanning workflow configured, skip to step #7.

1. The first thing you need is a public repository (GHAS is free for public repos) or a private repository with the GitHub Advanced Security license enabled
1. In the Security tab on the repository, navigate to 'Code scanning alerts' page
1. I'm using the native 'CodeQL Analysis' workflow by GitHub - there are 3rd party analysis engines here too!
1. Take a look at the workflow file - I didn't need to make any changes, but one can modify the language matrix if you want/don't want scans to run for a particular language
1. There's an `Autobuild` step here that works most of the time, but for some repositories I found I had to build my app manually - further reading on [build steps for compiled languages](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-the-codeql-workflow-for-compiled-languages#adding-build-steps-for-a-compiled-language)
1. Commit the workflow to your branch and merge it into Main - for best results we want an initial scan in the default branch before we test out the PR process
1. Under the Settings tab in the repository, navigate to Branches
1. Create a [branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule#creating-a-branch-protection-rule) for your default branch - check the 'Require status checks to pass before merging' box
1. If you used the GitHub CodeQL workflow, add the `CodeQL` status check and save the rule
   - You don't want the `Analyze (javascript)` status check; that just will show if the particular scan has completed, not that it found any vulnerabilities
   - If you don't see the `CodeQL` to add as a status check to the branch protection, **it won't appear as an option until you initiate at least one PR on the repository that triggers and completes the entire CodeQL scan** (meaning all of the `Analyze` jobs have finished) - as of December 2021, this is still an issue. It is vaguely alluded to in this [tidbit in the documentation](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/setting-up-code-scanning-for-a-repository#understanding-the-pull-request-checks) - emphasis mine: 
   > - When the code scanning jobs complete, GitHub works out whether any alerts were added by the pull request and adds the "Code scanning results / TOOL NAME" entry to the list of checks. **After code scanning has been performed at least once**, you can click Details to view the results of the analysis.

Step #9 was the part I wasn't originally confused on. The other entry (eg. `Analyze (javascript)`) is only the *scan job* for the corresponding language. It should succeed irregardless of if vulnerabilities are found. If it fails, the `autobuild` task might not be able to compile your code. The [Understanding the pull request checks
](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/setting-up-code-scanning-for-a-repository#understanding-the-pull-request-checks) GitHub documentation summarizes well.

After configuring the code scanning workflow and branch protection policy, you should be all set!

![branch protection policy configuration](/assets/screenshots/2020-12-16-github-codeql-pr/branch-protection-configuration.png){: .shadow }
_Branch Protection Policy with the CodeQL status check configured_

## Testing It Out

Another thing the GitHub Docs do not do a good job of spelling out is that only *Errors* or a security severity level of *High or Higher* are going to fail the Pull Request status check. *Warnings*, out of the box, do not block the PR.

Alright, so let's introduce an error...does anyone know of an easy vulnerability we can put in our code? Well neither do I, but we don't have to with the help of our friend, the [Semmle vulnerability database](https://web.archive.org/web/20200929073843/https://help.semmle.com/wiki/label/js/path-problem) (Note: This is now the [GitHub Advisory Database](https://github.com/advisories)).

I'm going to use an incorrect suffix vulnerability. The easiest way to introduce this is to: 1) make sure `javascript` is in the language matrix in our CodeQL workflow like so: `language: [ 'csharp', 'javascript' ]` and 2) check in a simple `.js`{: .filepath} file somewhere in the repository with the bad code:

```javascript
function endsWith(x,y) {

  let index = x.lastIndexOf(y);
  return x.lastIndexOf(y) === x.length - y.length;

}
```

Make sure to commit this in a branch because we want to test out the PR flow!

Afterwards, create the PR and wait for the job to run:

![blocked pr](/assets/screenshots/2020-12-16-github-codeql-pr/pr.png){: .shadow }
_Pull Request that is blocked because of a code scanning vulnerability (note that I can still force merge since I am an Administrator)_

Success! Or, failure, just as we wanted - the pull request cannot be merged because the `CodeQL` status check failed, meaning it detected a vulnerability in the code.

Once you fix the vulnerable code and re-push, all of the status checks will be successful and you are free to merge:

![passing pr](/assets/screenshots/2020-12-16-github-codeql-pr/pr-passing.png){: .shadow }
_Pull Request with passing status checks - no vulnerable code has been found_

The fact that GitHub gives this away for free for all public repositories is incredible! There is a licensing upcharge for Enterprises, but the setup is so simple and integrations so robust makes it well worth it (and we're only scratching the surface!).

## Status Check Failure Configuration

By default, the status check will only fail if there is an *Error* or a security severity level of *High or Higher* *High or Higher* vulnerability - *Warnings* or a security level of *Medium* will not fail the status check. However, we can use the [Control which code scanning alerts cause a pull request check to fail](https://github.blog/changelog/2021-06-03-control-which-code-scanning-alerts-cause-a-pull-request-check-to-fail/) feature that was release in June 2021 to configure what alert level will fail the PR:

![defining-the-severities-causing-pull-request-check-failure](/assets/screenshots/2020-12-16-github-codeql-pr/code-scanning-configuration.png){: .shadow }
_Control which code scanning alerts cause a pull request check to fail_

For more information, see the documentation for [Defining the severities causing pull request check failure](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/configuring-code-scanning#defining-the-severities-causing-pull-request-check-failure).

## Summary

Not allowing code that introduces vulnerabilities to be merged in the PR process is crucial to ensuring the integrity of our code. Blocking a PR that contains a code vulnerability is essentially THE use case of GitHub Advanced Security - we're able to see right on our PR in GitHub that there's a vulnerability, with the deep linking and integration that you would expect. Finding out about the issue at PR time shortens the feedback loop - we don't have to scramble before a production deployment if your security scan is occurring too late in the process.

Even without the branch protection configured, the code scanning results will still show that `CodeQL` found a vulnerability, but without the branch protection we would be able to freely merge this into main:
![pr with no branch protection](/assets/screenshots/2020-12-16-github-codeql-pr/pr-no-branch-protection.png){: .shadow }
_Pull Request that shows a code vulnerability found, but since there is no branch protection on this repo, we are free to merge_

This might be enough for some, but if we're going to go through the exercise of creating a code scanning workflow, just as it's a best practice to require at least one other approver on the PR before merging, we should require that there are no vulnerabilities being introduced as well. GitHub Advanced Security prides itself on limiting the signal vs noise ratio, so the chance of getting a 'High' vulnerability result that is a false positive is pretty minimal. And if you do get a false positive or a result you aren't going to fix - just [dismiss](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/managing-code-scanning-alerts-for-your-repository#dismissing-or-deleting-alerts) the alert.

Happy (secure) coding!

---

## Extra Credit - Analyzing a SARIF File

I originally had this section in here to assist with blocking PR's from results other than *Errors* (such as *Warnings*), but GitHub has [implemented this feature now](#status-check-failure-configuration). I will leave the section below as it might be useful in its own right if you are interested in deep-diving or debugging your `SARIF` results file. 

### Original Content - Analyzing a SARIF Results File Manually

Okay, what if we were to have a repository with Terraform code and used the ShiftLeft Analysis marketplace code scanning workflow? Or, we used the native GitHub CodeQL workflow but want it to block merges when it finds any result, including warnings?

Well, in the case of the ShiftLeft Analysis workflow, there is a [config file](https://web.archive.org/web/20210615164536/https://slscan.io/en/latest/integrations/tips/#config-file) that can be uploaded to the root of the repository to define some of this, but I haven't played around much for this. For the GitHub CodeQL workflow, there is no fine-tuning configuration file that we can easily use (that I know of at this date).

For this, I wrote a script that examines the `.sarif`{: .filepath} and added another job to the workflow, like so:

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

The entire workflow can be found in my [GitHub branch](https://github.com/joshjohanning/tailspin-spacegame-web-deploy/blob/2d4955b668ffde45a2f4ea6e742268a536249b27/.github/workflows/codeql-analysis.yml).

Because the `.sarif`{: .filepath} produced by the ShiftLeft analysis is slightly different and by default *doesn't* fail the job even with *errors*, I created a different workflow you can use to block pull requests if errors or warnings are found - see for [example](https://github.com/joshjohanning/azdo-terraform-tailspin/blob/05151b64818db1c4cabf5aaf51f0024c779d81f5/.github/workflows/shiftleft-analysis.yml).

Now just like we did above, we can modify our branch rule to require the "Detect-Errors" job to finish successfully, as this job will run successfully if there are no errors/warnings.

![pr-detected-errors-job](/assets/screenshots/2020-12-16-github-codeql-pr/pr-detected-errors.png){: .shadow }
_Adding the new job to the required status check list_
![pr-blocked](/assets/screenshots/2020-12-16-github-codeql-pr/pr-blocked.png){: .shadow }
_Pull Request that is blocked because of a 'warning' result found in the code scanning results_

I'm sure there is probably a better way to do this (using the API or GraphQL endpoint?). I know back in the LGTM / Semmle days, there was also a config file you could commit to the root of the repository to more precisely define rules. Either way, let me know in the comments if you have any other ideas or improvements!
