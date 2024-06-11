---
title: 'Quickly Migrate NuGet Packages to a New Feed'
author: Josh Johanning
date: 2020-12-23 16:45:00 -0600
categories: [Azure DevOps, Artifacts]
tags: [Azure DevOps, NuGet, Scripts, Migrations]
media_subpath: /assets/screenshots/2020-12-20-nuget-pusher-script
image:
  path: azure-artifacts.png
  width: 100%
  height: 100%
  alt: NuGet packages in Azure Artifacts
---

## Summary

This is a *very* simple bash script that can assist you in migrating NuGet packages to a different Artifact feed. It's written with Azure DevOps in mind as a target, but there's no reason why you couldn't use any other artifact feed as a destination.

I used the script after I ran a NuGet restore locally of a Visual Studio solution, found my `.NuGet`{: .filepath} folder with all of the cached packages, placed my script in that folder, and ran it. This saved me a lot of time, migrating 102 packages while I grabbed a cup of coffee ☕️!

> Note that this won't work for migrating NuGet packages to **GitHub Packages** since the `<repository url="..." />` element in the `.nuspec`{: .filepath} file in the `.nupkg`{: .filepath} needs to be updated. See my other posts: 
> - [Migrate NuGet Packages Between GitHub Instances](/posts/github-packages-migrate-nuget-packages/)
> - [Migrate NuGet Packages to GitHub Packages](/posts/github-packages-migrate-nuget-packages-to-github-packages/)
{: .prompt-info }

## The Script

Copy the script below and save it as `nuget-pusher.sh`{: .filepath} in the folder where your `.nupkg`{: .filepath} files are located. Don't forget to `chmod +x nuget-pusher.sh`{: .filepath} to make it executable!

{% raw %}

```bash
#!/bin/bash

# Usage: ./nuget-pusher-script.sh <nuget-feed-name> <nuget-feed-source> <PAT>

if [ -z "$3" ]; then
    echo "Usage: $0 <nuget-feed-name> <nuget-feed-source> <PAT>"
    exit 1
fi

echo "..."

NUGET_FEED_NAME=$1
NUGET_FEED_SOURCE=$2
PAT=$3

# adding to ~/.config/NuGet/NuGet.config
dotnet nuget add source \
  $NUGET_FEED_SOURCE \
  --name $NUGET_FEED_NAME \
  --username "az" \
  --password $PAT \
  --store-password-in-clear-text

results=$(find . -name "*.nupkg")
resultsArray=($results)

for pkg in "${resultsArray[@]}"
do
    echo $pkg
    dotnet nuget push --source \
      $NUGET_FEED_NAME \
      --api-key az \
      $pkg
done

# clean up
dotnet nuget remove source $NUGET_FEED_NAME

echo "..."
```

{% endraw %}

> According to the [docs](https://learn.microsoft.com/en-us/azure/devops/artifacts/nuget/dotnet-exe?view=azure-devops#publish-packages), any string will work for the `--api-key` parameter.
{: .prompt-info }

## Running the Script

The script below can be called via:

```bash
./nuget-pusher-script.sh <nuget-feed-name> <nuget-feed-source> <PAT>
```

An example:

```bash
./nuget-pusher-script.sh \
  azure-devops \
  https://pkgs.dev.azure.com/jjohanning0798/_packaging/my-nuget-feed/nuget/v3/index.json \
  xyz_my_pat
```

## Bonus: Locating .nupkg Packages

How to find the location of `.nupkg`{: .filepath} files:

```bash
find / -name "*.nupkg" 2> /dev/null
```

How to find the location of `.nupkg`{: .filepath} files and copy them all to a directory:

```bash
find / -name "*.nupkg" -exec cp "{}" ./my-directory  \; 2> /dev/null
```

> The `2> /dev/null` is to suppress permission errors.
{: .prompt-info }

## Improvement Ideas

* [ ] The `username` doesn't matter in this script since when using an Azure DevOps PAT, username is irrelevant. If one was pushing to a NuGet feed that required username authentication, I would presume you would add that as an input.
* [ ] One could also add a source folder as an input too instead of relying on current directory
* [x] Also one could use a more elaborate input mechanism to the script...
* [x] Use `dotnet nuget` instead of `nuget` command
* [x] Support other systems (such as `ubuntu`) with `--store-password-in-clear-text` flag
* [x] Clean up temporary NuGet source when done
