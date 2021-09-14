---
title: 'Quickly Migrate NuGet Packages to a New Feed En Masse'
author: Josh Johanning
date: 2020-12-23 16:45:00 -0600
categories: [Azure DevOps]
tags: [Azure DevOps, NuGet, Scripts]
---

## Summary

This is a *very* simple bash script that can assist you in migrating NuGet packages to a different Artifact feed. It's written with Azure DevOps in mind as a target, but there's no reason why you couldn't use any other artifact feed as a destination.

I used the script after I ran a NuGet restore locally of a Visual Studio solution, found my .NuGet folder with all of the cached packages, placed my script in that folder, and ran it. This saved me a lot of time, migrating 102 packages while I grabbed a cup of coffee!

{% raw %}

```bash
echo -n "NuGet feed name?"
read nugetfeed
echo -n "NuGet feed source?"
read nugetsource
echo -n "Enter PAT"
read pat
# adding to ~/.config/NuGet/NuGet.config
nuget sources add -Name $nugetfeed -Source $nugetsource -username "az" -password $pat 
results=$(find . -name "*.nupkg")
resultsArray=($results)

for pkg in "${resultsArray[@]}"
do
    echo $pkg
    nuget push -Source $nugetfeed -ApiKey az $pkg
done
```

{% endraw %}

## Improvement Ideas

* The `username` doesn't matter in this script since when using an Azure DevOps PAT, username is irrelevant. If one was pushing to a NuGet feed that required username authentication, I would presume you would add that as an input.
* One could also add a target folder as an input too.
* Also one could use a more elaborate input mechanism to the script...
