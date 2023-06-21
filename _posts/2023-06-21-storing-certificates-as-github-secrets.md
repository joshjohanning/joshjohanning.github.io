---
title: 'Using GitHub Actions Secrets to store Certificates/Keys'
author: Josh Johanning
date: 2023-06-21 16:00:00 -0500
description: Storing a certificate/private key as a GitHub Actions secret
categories: [GitHub, GitHub Actions]
tags: [GitHub, GitHub Actions]
---

## Overview

In Azure DevOps, if you wanted to store a certificate, you would use a "[Secure Files](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops)" feature. GitHub doesn't have the same native functionality, but you can still store the value of a certificate as a secret to be used in your GitHub Actions workflows. Let's see some approaches.

## Storing the Value of a Key/Certificate

When storing an unencrypted key/certificate, you can simply grab the contents of the `.pem`{: .filepath } and store it as a secret in GitHub. Then, you can write the value of the secret to a file and use in your GitHub Actions workflows.

Sample steps:

1. Display/copy the contents of the `.pem`{: .filepath } file: `cat private-key.pem`
2. Add the value of the `.pem`{: .filepath } file as an [Action secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) in GitHub
3. In the workflow, add a step to write the value of the secret to a file:

{% raw %}

```yml
- name: save secret to file
  run: | 
    echo $PRIVATE_KEY > private-key.pem
  env:
    PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
```

Easy, right?

> Note: This works well for secret values that can be copied/pasted as plaintext. This does not work well for binary files, such as `.p12`{: .filepath } files.
{: .prompt-info }

## Storing the Value of a File

For encrypted or binary files, such as `.p12`{: .filepath } certificates, you can `base64` the entire file and store the `base64` value as a secret in GitHub. Then, you can decode the value of the secret to a file and use in your GitHub Actions workflows.

Sample steps:

1. Use the `base64` command to encode the file: `base64 ./my-certificate.p12`
  - On macOS, you may have to use: `base64 -i ./my-certificate.p12`
  - There is also a PowerShell option: `[System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes("path/to/file"))`
2. Add the `base64` value as an [Action secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) in GitHub
3. In the workflow, add a step to write the value of the secret to a file:

```yml
- name: save secret to file
  run: | 
    echo -n $MY_CERTIFICATE | base64 -d -o ./my-certificate.p12
  env:
    MY_CERTIFICATE: ${{ secrets.MY_CERTIFICATE }}
```

This is pretty much the exact same thing with the added step of decoding the `base64` value to a file.

> Note: This option will work regardless of the type of file you're storing.
{: .prompt-info }

{% endraw %}

## Summary

Whether you're storing the private key of a GitHub App or storing the signing and distribution certificates for an iOS build, you can use GitHub Actions secrets to store the value of a certificate/private key and use it in your workflows. 🚀
