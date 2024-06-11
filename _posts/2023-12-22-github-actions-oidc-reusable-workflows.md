---
title: 'Using OIDC with Reusable Workflows to Securely Access Cloud Resources'
author: Josh Johanning
date: 2023-12-22 11:30:00 -0600
description: Using Reusable Workflows in GitHub Actions to standardize and security harden your deployment steps
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, OIDC, Azure, Reusable Workflows]
media_subpath: /assets/screenshots/2023-12-22-github-actions-oidc-reusable-workflows
image:
  path: github-oidc-success-light.png
  width: 100%
  height: 100%
  alt: Using OIDC in GitHub Actions to authenticate to Azure and retrieve secrets from a Key Vault
---

## Overview

OpenID Connect (OIDC) is great for accessing resources by exchanging short-lived tokens directly to the thing you are trying to authenticate with (often a cloud provider but doesn't have to be!). GitHub Actions has [several examples for using OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) in workflows to be able to access resources like Azure, AWS, HashiCorp Vault, etc. Passwordless authentication is game-changing!

In GitHub Actions, Reusable workflows are also great for providing consistency to workflows within an organization. Also, they prevent code duplication and simplifies making changes to workflows.

These two features can be combined to provide a secure and consistent way to access cloud resources.

For example, what if there was a secret that was required in every single workflow (such as a key to access a private Maven/NuGet/npm/Docker/etc. feed)? When using reusable workflows, that secret has to exist *on the caller workflow repo*, not on the **called* aka *reusable* workflow repo*. You either have to create an organization secret that has access to all repositories (which isn't ideal since that means anyone with write access to a repository can write some code to access that secret), or you have to create a secret on each repository that uses the reusable workflow. This is where the magic of [OIDC and reusable workflows](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/using-openid-connect-with-reusable-workflows) meet!

This post will show you how to customize your subject claims on the GitHub repository pass in the reusable workflow to Azure to be able to authenticate to an Azure Key Vault and retrieve a secret.

## The OIDC Subject Claim

Following the [GitHub docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/using-openid-connect-with-reusable-workflows): 

> - Using `job_workflow_ref`:
>   - To create trust conditions based on reusable workflows, your cloud provider must support custom claims for `job_workflow_ref`. This allows your cloud provider to identify which repository the job originally came from.
>   - For clouds that only support the standard claims (audience (`aud`) and subject (`sub`)), you can use the API to customize the sub claim to include `job_workflow_ref`. For more information, see "[About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims)". Support for custom claims is currently available for Google Cloud Platform and HashiCorp Vault.
> - Customizing the token claims:
>   - You can configure more granular trust conditions by customizing the subject (`sub`) claim that's included with the JWT. For more information, see "[About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims)".

> - We can also see which claims are supported here: [https://token.actions.githubusercontent.com/.well-known/openid-configuration](https://token.actions.githubusercontent.com/.well-known/openid-configuration)
{: .prompt-tip }

If you're not an OIDC expert (don't worry, I'm not either), this might not make a ton of sense, but don't worry, let's step through it.

Let's first start by examining the subject (`sub`) claim that GitHub Actions generates by default. We can print out the token by copying a bash script step or using a ready-made action to debug the OIDC token claims. The action is a Docker action, which can make it harder to run on some hosts, so I am including both examples here:

{% raw %}
```yml
  print-oidc-token:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # this is needed for oidc
      contents: read # this is needed to clone repo
    steps:

    # debug using the action
    - name: Debug OIDC Claims
      uses: github/actions-oidc-debugger@main
      with:
        audience: '${{ github.server_url }}/${{ github.repository_owner }}'
        
    # print oidc token claims manually
    - name: print oidc token claims
      run: |
          IDTOKEN=$(curl -s -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL" -H "Accept: application/json; api-version=2.0" -H "Content-Type: application/json"  | jq -r '.value')
          jwtd() {
            if [[ -x $(command -v jq) ]]; then
                jq -R 'split(".") | .[1] | @base64d | fromjson' <<< "${1}" > jwt_claims.json
                cat jwt_claims.json
                echo ${{ env.ACTIONS_ID_TOKEN_REQUEST_URL}} 
            fi
          }
          jwtd $IDTOKEN
```
{: file='.github/workflows/debug-oidc-demo.yml'}
{% endraw %}

By default, the `sub` of the OIDC token that GitHub Actions generates just looks something like this:

```json
"sub": "repo:joshjohanning-org/standard-oidc-claim-demo:ref:refs/heads/main"
```
{: .nolineno}

Notice that there isn't anything special in there; just the repository that is running the workflow and the ref (or if you were doing a deployment, the deployment environment would show here).

We want to customize this where that our cloud provider (Azure in my example) can authenticate with our approved reusable workflow.

> For AWS, the [docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) say: "Note: Support for custom claims for OIDC is unavailable in AWS." This is saying you can't create any custom claims ([discussed here](https://github.com/aws-actions/configure-aws-credentials/issues/306)), but you can still customize the subject (`sub`) claim as I show later in this post. AWS's [docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub) and [section in the `aws/configure-aws-credentials` action](https://github.com/aws-actions/configure-aws-credentials?tab=readme-ov-file#claims-and-scoping-permissions) has more information on this.
{: .prompt-info }

## Customizing the Subject Claim in GitHub

We can [customize the subject claim using the API](https://docs.github.com/en/rest/actions/oidc?apiVersion=2022-11-28#set-the-customization-template-for-an-oidc-subject-claim-for-a-repository), but more easily, we can use [@tspascoal](https://github.com/tspascoal)'s [gh-oidc-sub](https://github.com/tspascoal/gh-oidc-sub) `gh` CLI extension:

1. Install the `gh` CLI extension:
    ```bash
    gh extensions install tspascoal/gh-oidc-sub
    ```
    {: .nolineno}
2. Let's verify the existing claims (if it is customized or using default):
    ```bash
    gh oidc-sub get --repo joshjohanning-org/oidc-claims-demo
    ```
    {: .nolineno}
3. If nothing has been changed yet at the repo or org level, it should look like this:
    ```json
    {
      "use_default": true
    }
    ```
    {: .nolineno}
    > Note that if it isn't using the default, you can set it back to the default by running:
    > 
    > ```bash
    > gh oidc-sub usedefault --repo joshjohanning-org/oidc-claims-demo
    > ```
    > {: .nolineno}
    {: .prompt-tip }
4. Then, we can run the following command to customize the subject claim to include the `job_workflow_ref`:
    ```bash
    gh oidc-sub set --repo joshjohanning-org/oidc-claims-demo --subs "job_workflow_ref"
    ```
    {: .nolineno}
5. This just returns `{}`, but let's run the `get` command again to verify that it was set:
    ```bash
    gh oidc-sub get --repo joshjohanning-org/oidc-claims-demo
    ```
    {: .nolineno}
6. Now, the output should look like this:
    ```json
    {
      "use_default": false,
      "include_claim_keys": [
        "job_workflow_ref"
      ]
    }
    ```
    {: .nolineno}
7. If we run the step to print out the OIDC token claims as discussed in the [section above](#the-oidc-subject-claim), we will see:
    ```json
    "sub": "job_workflow_ref:joshjohanning-org/oidc-claims-demo/.github/workflows/azure-oidc-demo.yml@refs/heads/main"
    ```
    {: .nolineno}
8. With the subject claim customized, we can require all interactions with Azure use this reusable workflow 🎉

## Using the Subject in Azure

Now that we have the subject claim customized on the GitHub repository, we can use it with the federated credential on the Azure side.

1. In AAD (Entra ID), navigate to the app registration that you want to use to authenticate to Azure
2. Under "Certificates & secrets", add a new "Federated credential"
3. You can select "GitHub" as the federated credential scenario, but it's easier to just use "Other issuer"
4. For the issuer, use: `https://token.actions.githubusercontent.com`
5. For the subject identifier, use something like:
    ```
    job_workflow_ref:joshjohanning-org/reusable-workflows/.github/workflows/azure-oidc-sample.yml@refs/tags/v1
    ```
    {: .nolineno}
    > You will have to decide if you want to use a tag or a branch for the ref, and in Azure, you can't use wildcards (in AWS you can!). I prefer tags for consistency, but you can use a branch if you simply want your users to refer to `@main` to always have the latest. If referencing a branch, use: `refs/heads/main`
    {: .prompt-info }
6. It should look something like this:
    ![Federated credential in Azure using job_workflow_ref](azure-oidc-light.png){: .shadow }{: .light }
    ![Federated credential in Azure using job_workflow_ref](azure-oidc-dark.png){: .shadow }{: .dark }
    _Federated credential in Azure using job_workflow_ref_

> Note the maximum number of federated credentials per app registration from the [Azure docs](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity?pivots=identity-wif-mi-methods-azp#important-considerations-and-restrictions):
> > A maximum of 20 federated identity credentials can be added to an application or user-assigned managed identity.
{: .prompt-info }

## Configuring the Reusable Workflow

So far we have updated the subject claim in GitHub and configured the federated credential in Azure. Now, we will create a reusable workflow in GitHub and test out if we can 1) successfully authenticate using the approved `@v1` tag above, and 2) if it fails when it should when using another tag or any other reusable workflow.

Here's my calling workflow (i.e.: the workflow in my "app" repo):

```yml
name: Azure OIDC Demo

on:
  push:
    branches: main
  pull_request:
    branches: main
  workflow_dispatch:

jobs:
  azure:
    uses: joshjohanning-org/reusable-workflows/.github/workflows/azure-oidc-sample.yml@v1 # v1 is 'approved' workflow
    with:
      keyvault: josh-key-vault-test
```
{: file='.github/workflows/azure-oidc-demo.yml'}

> For [security purposes](https://github.blog/changelog/2023-06-15-github-actions-securing-openid-connect-oidc-token-permissions-in-reusable-workflows/), if you need to fetch an OIDC token generated within a reusable (called) workflow that is outside your enterprise/organization, the `id-token: write` needs to be explicitly set at the caller workflow level or in the specific job that calls the reusable workflow.
{: .prompt-tip }

And here's the called workflow (i.e.: the workflow in my "reusable workflows" repo):

{% raw %}
```yml
name: azure-oidc-sample 

on:
  workflow_call:
      keyvault:
        description: name of the keyvault
        type: string
        default: josh-key-vault-test 

jobs:
  login:
    runs-on: ${{ inputs.runs-on }}
    permissions:
      id-token: write # this is needed for oidc
      contents: read # this is needed to clone repo

    steps:
    - uses: actions/checkout@v4
    # logging in with OIDC
    - name: 'Az CLI login'
      uses: azure/login@v1
      with:
        client-id: d951ac80-75f2-446a-aca6-cd53a68611f0
        tenant-id: e9846558-c4f0-4312-a89e-ebebe80779a1
        subscription-id: 2e9bfb26-ca29-44f5-8920-72c1b0b37188
        
    - name: print azure subscription info
      run: |
        az account show
        az account show | jq ".id"
        
    - name: get all az keyvault secrets
      run: |
        for secret_name in $(az keyvault secret list --vault-name ${{ inputs.keyvault }} --query "[].{name:name}" --output tsv); do
          secret_value=$(az keyvault secret show --vault-name "${{ inputs.keyvault }}" --name $secret_name --query value -o tsv)
          echo "::add-mask::$secret_value"
          echo "$secret_name=$secret_value" >> $GITHUB_ENV
        done
        
    - name: testing secrets
      run: |
        echo "echoing as secret: ${{ secrets.my-secret }}" # doesn't work
        echo "echoing as env: ${{ env.my-secret }}" # works
```
{: file='.github/workflows/azure-oidc-sample.yml'}
{% endraw %}

When running the workflow, we can see that it successfully logs in and fetches the secrets from the keyvault:

![Using OIDC in GitHub Actions to authenticate to Azure and retrieve secrets from a Key Vault](github-oidc-success-light.png){: .shadow }{: .light }
![Using OIDC in GitHub Actions to authenticate to Azure and retrieve secrets from a Key Vault](github-oidc-success-dark.png){: .shadow }{: .dark }
_Using OIDC in GitHub Actions to authenticate to Azure and retrieve secrets from a Key Vault_

If we try to be sneaky and use a different reusable workflow, a different tag/branch, or no reusable workflow at all, it will fail. 

Here's an example where I tried to use a different reusable workflow and it fails:

```yml
jobs:
  azure:
    uses: joshjohanning-org/reusable-workflows/.github/workflows/azure-oidc-sample-not-approved.yml@oidc-sample-not-approved # v1 is 'approved' workflow
    with:
      keyvault: josh-key-vault-test
```
{: file='.github/workflows/azure-oidc-demo.yml'}

![Failing to use OIDC to authenticate to Azure because I'm not using an approved reusable workflow](github-oidc-failure-light.png){: .shadow }{: .light }
![Failing to use OIDC to authenticate to Azure because I'm not using an approved reusable workflow](github-oidc-failure-dark.png){: .shadow }{: .dark }
_Failing to use OIDC to authenticate to Azure because I'm not using an approved reusable workflow_

## Summary

Often, I see that [teams want to abstract and isolate their reusable workflows](https://github.com/orgs/community/discussions/17554) completely from the teams calling them. The team building the reusable workflows don't want to require secrets to be stored in the *calling* repository for the sake of both reducing complexity and increasing security. There is the `secrets: inherit` keyword that can be used to pass in all secrets and satisfy the complexity complain, but it doesn't satisfy the security concern. Any repo-level or organization-level secret that exists in the repository can be accessed by anyone with `write` permissions to the repo by creating a new workflow. There is a [roadmap item](https://github.com/github/roadmap/issues/636) to address these concerns, but there is no timeline for it yet.

However, using OIDC to authenticate to Azure and retrieve secrets from a Key Vault is a great way to solve this problem, especially if you're already using an external key store like Azure Key Vault to manage your secrets. Where it really gets magically is when we combine OIDC and reusable workflows to create a secure and consistent reusable workflow that can be used across the organization without having to make secrets accessible to any other workflow. ✨
