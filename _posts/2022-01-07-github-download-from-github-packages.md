---
title: 'Programmatically Download a Package Binary from GitHub Packages'
author: Josh Johanning
date: 2022-01-07 14:30:00 -0600
description: Programmatically download a package binary (such as NuGet, Maven) from GitHub Packages
categories: [GitHub, Packages]
tags: [GitHub, GitHub Packages, Maven, NuGet, npm]
media_subpath: /assets/screenshots/2022-01-07-github-download-from-github-packages
image:
  path: github-packages.png
  width: 100%
  height: 100%
  alt: Assets for a package in GitHub Packages
---

## Overview

We had a team that wanted to push to GitHub packages, which is relatively easily enough to do and is well [documented](https://docs.github.com/en/packages/working-with-a-github-packages-registry). However, they had a subsequent job that was building a Docker image where that dependency (the `.jar`{: .filepath} or `.war`{: .filepath} file) was needed.

There are a couple of different ways you could think about this.

1. Maybe the `docker build` step should occur in the same job as the `mvn build` step so that it has access to the same binary outputs
2. Perhaps instead of GitHub Packages we create a Release on the repository - we can use an [Action](https://github.com/softprops/action-gh-release) to do this and an [API](https://docs.github.com/en/rest/reference/releases#get-a-release-asset) to download the release
3. If we really just want to download the package binary from GitHub Packages...that should be simple enough, right?

## Just use the Packages API, right?

The [API for GitHUb Packages](https://docs.github.com/en/rest/reference/packages) says:

> With the GitHub Packages API, you can **manage** packages for your GitHub repositories and organizations.

Keyword: **manage**, such as listing or deleting packages. It doesn't really imply *downloading* or *retrieiving package assets* like the [Release API](https://docs.github.com/en/rest/reference/releases#get-a-release-asset) has.

Okay, but can't we just go to the package in the UI and copy the download link?

Nope -- check the URL of one of the files in my repo:

> https://github-registry-files.githubusercontent.com/445574648/92585100-6fe8-11ec-8a00-38630c14852f?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20220107%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220107T193611Z&X-Amz-Expires=300&X-Amz-Signature=96f4809aebb229ea01b80832c12e546810837194203927c39a31f2c875b177fd&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=445574648&response-content-disposition=**filename%3Dherokupoc-1.0.0-202201071835.jar**&response-content-type=application%2Foctet-stream

Pretty nasty huh? It looks to be a timed download URL.

After spending a few hours on this, there were a few ways I found to do this with various levels of monstrocities committed in finding. I'll start with the best / easiest and work my way down. 

## Mysteriously hidden URLs to CURL

In hindsight, it's so simple (at least for Maven), yet it's not documented anywhere! I was trying to use the `mvn dependency:get/copy` cli and kept getting stuck on a `401 unauthorized` error message. In the logs, I saw the URL to the `.jar`{: .filepath} file I was trying to download and decided to paste that into my browser. I received a username/password basic auth prompt, and I simply pasted in my PAT as a password and I was able to download that file.

Extrapulating to `curl`, this was how to replicate this in the command line:

{% raw %}
```bash
curl 'https://maven.pkg.github.com/<org>/<repo>/com/<group>/<artifact>/<version>/<file-name>.jar' \
  -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" -L -O
```

And because my biggest pet peave is when someone has this awesome blog post but then hides/obfuscates all the good stuff, here's my actual CURL command I used to download a file:

```bash
curl 'https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package/com/sherlock/herokupoc/1.0.0-202201071559/herokupoc-1.0.0-202201071559.jar' \
  -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" -L -O
```

The `-L` is important here as this tells `curl` to follow redirects. Without it, you'll get a '301 Moved Permanently' because it's trying to use use the expanded URL as mentioned above. If you added the `-v` option to the command, you would see a similar long URL that our `curl` follows the redirect to.

The `-O` downloads the file locally with the same name as in the URL.

### Maven URL

As noted, the Maven URL

Format:

```
https://maven.pkg.github.com/<org>/<repo>/com/<group>/<artifact>/<version>/<file-name>.jar
```

Example:

```
curl -H "Authorization: token ghp_xyz" -L -O \
  https://maven.pkg.github.com/joshjohanning-org/download/com/sherlock/herokupoc/1.0.0-202202122241/herokupoc-1.0.0-202202122241.jar
```

### NuGet URL

Format:

```
https://nuget.pkg.github.com/<org>/download/<package-name>/<version>/<package-name>.<version>.nupkg
```

Example:

```
curl -H "Authorization: token ghp_xyz" -L -O \
  https://nuget.pkg.github.com/joshjohanning-org/download/Wolfringo.Hosting/1.1.1/Wolfringo.Hosting.1.1.1.nupkg
```

### npm URL

`npm` must work a bit differently; the download URL is different:

```bash
# get package versions
version="0.0.3"
token="ghp_xyz"
org="joshjohanning-org"
package_name="npm-package-example"

# get url
url=$(curl -H "Authorization: token $token" -Ls https://npm.pkg.github.com/@$org/$package_name | jq --arg version $version -r '.versions[$version].dist.tarball')
# download 
curl -H "Authorization: token $token" -L -o $package_name-$version.tgz $url
```

## Other options

In my quest to find out how to download a Maven package from GitHub packages, I stumbled upon a few other options that I'll list here for posterity.

### mvn install and mv

I was able to get something like this to work - see my GitHub Action job below:

```yml
  download-job:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: maven-settings-xml-action
      uses: whelk-io/maven-settings-xml-action@v20
      with:
        repositories: '[{ "id": "fix_world", "url": "https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package" }]'
        servers: '[{ "id": "fix_world", "username": "joshjohanning", "password": "${{ secrets.GITHUB_TOKEN }}" }]'

    - name: View settings.xml
      run: cat ~/.m2/settings.xml 
      
    - name: Install with Maven
      run: mvn install -s ~/.m2/settings.xml
      
    # wildcard find and mv command to current directory
    - name: mv
      run: find /home/runner/.m2 -name "*herokupoc-1.*.jar" -exec mv {} . \;
```

### mvn cli - sort of

This is what we were originally trying to do, so thought I would throw it in here to spur other ideas for other languages. We were trying to use `mvn dependenct:get` to download the `.jar`{: .filepath} file, but were ultimately uncessful for one reason or another.

This is the job we ended with:

```yml
  mvncli:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: maven-settings-xml-action
      uses: whelk-io/maven-settings-xml-action@v20
      with:
        repositories: '[{ "id": "fix_world", "url": "https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package" }]'
        servers: '[{ "id": "fix_world", "username": "joshjohanning", "password": "${{ secrets.GITHUB_TOKEN }}" }]'
    - name: download
      run: | 
        mvn dependency:get \
          -DgroupId=com.sherlock \
          -DartifactId=herokupoc \
           -Dversion=1.0.0-202201071835 \
          -Dpackaging=jar \
          -Dclassifier=sources \
          -DremoteRepositories=central::default::https://repo.maven.apache.org/maven2,fix_world::::https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package
```

But gives us an authentication error: 

> Error:  Failed to execute goal org.apache.maven.plugins:maven-dependency-plugin:3.1.1:get (default-cli) on project herokupoc: Couldn't download artifact: org.eclipse.aether.resolution.DependencyResolutionException: Could not transfer artifact com.sherlock:herokupoc:jar:sources:1.0.0-202201071835 from/to fix_world (https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package): authentication failed for **https://maven.pkg.github.com/joshjohanning-org/sherlock-heroku-poc-mvn-package/com/sherlock/herokupoc/1.0.0-202201071835/herokupoc-1.0.0-202201071835-sources.jar**, status: 401 Unauthorized -> [Help 1]

But this was still useful though as this what led me to just try to [`curl` that `.jar`{: .filepath} file URL](#mysteriously-hidden-urls-to-curl) successfully 😀.

### GraphQL

This [post used to include](https://github.com/joshjohanning/joshjohanning.github.io/blob/168740292d487f3dc841b54f630f995420f4b924/_posts/2022-01-07-github-download-from-github-packages.md?plain=1#L71-L244) a GraphQL reference for downloading a file from GitHub Packages, but the GraphQL endpoint for GitHub Packages has been [deprecated on GitHub.com](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/) and [GitHub Enterprise Server 3.7.0+](https://docs.github.com/en/enterprise-server@3.7/admin/release-notes#3.7.0-deprecations) and no longer works.

## Wrap-up

Hopefully this either helped you download a file from GitHub Packages, gives you ideas for other languages, or maybe my struggles convince you to *just use a [Release in GitHub](https://docs.github.com/en/repositories/releasing-projects-on-github)*.

I should mention that if you're running this in a GitHub Action workflow from a repository other than the repository publishing the package, you may have to [update the package's permissions](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility#ensuring-workflow-access-to-your-package) for the repository to have access to the package in (not available in Maven [yet](https://github.com/github/roadmap/issues/578)).

Let me know what I've missed or if you have any other ideas! 📦

{% endraw %}
