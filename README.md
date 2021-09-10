# joshjohanning.github.io

## Overview

A DevOps Blog  - Blogging about Azure DevOps and GitHub practices, tips, and my continuous improvement DevOps journey.

[josh-ops.com](https://josh-ops.com)

## Theme Source

Chirpy:
* [GitHub](https://github.com/cotes2020/jekyll-theme-chirpy)
* [Example and tips/best practices](https://chirpy.cotes.info/)
* Upgrading - copy in the new source while taking care to keep existing configuration/customizations in place? 

## Comment System

Utterances:
* [GitHub](https://github.com/utterance/utterances)
* [Example/configuration](https://utteranc.es/)

The configuration lives in `/_layouts/post.html`

I added a section for dynamically selecting whether to use the light or dark utterances comment theme. I loosely documented this in an [issue in the utterances repository](https://github.com/utterance/utterances/issues/549#issuecomment-917091550).

```html
    <!-- When page loads, determine whether to show light mode or dark mode utterances comments -->
    <section>
      <div id="light-mode">
        <script src="https://utteranc.es/client.js"
          repo="joshjohanning/joshjohanning.github.io"
          issue-term="title"
          theme="github-light"
          crossorigin="anonymous"
          async>
        </script>
      </div>

      <div id="dark-mode">
        <script src="https://utteranc.es/client.js"
          repo="joshjohanning/joshjohanning.github.io"
          issue-term="title"
          theme="github-dark"
          crossorigin="anonymous"
          async>
        </script>
      </div>

      <script>
        var uttLight = document.getElementById("light-mode");
        var uttDark = document.getElementById("dark-mode");

        let initTheme = "light";
        if ($("html[mode=dark]").length > 0
          || ($("html[mode]").length == 0
            && window.matchMedia("(prefers-color-scheme: dark)").matches ) ) {
          initTheme = "dark";
        }

        if(initTheme === "dark"){
          uttLight.parentNode.removeChild(uttLight);
        } else {
          uttDark.parentNode.removeChild(uttDark);
        }
      </script>
    </section>
```
