# Copilot instructions

## Authoring / editing blog posts

### Formatting

- Add `{: .filepath}` after filenames to get syntax highlighting
- If it makes sense, after a code block or YML snippet, add a file name on the next line: `{: file='/name/of/file.md'}`
  - For sample usage/examples, this doesn't make sense
- Always describe the code block
- Add the `{: .nolineno}` under certain code blocks:
  - That are 1 line
  - Usage examples - not code that needs line-by-line explanation (unless it's a really complex example)
  - Command variations - showing different ways to run the same script
- Use `{: .prompt-info }`, `{: .prompt-tip }`, `{: .prompt-warning }`, and `{: .prompt-danger }` for callouts
- Wrap YML snippets with the `{% raw %}` and `{% endraw %}` tags to avoid Liquid parsing issues
- Do not leave any trailing whitespace at the end of lines

### Images

- When including images in a post, always include both light and dark variants
- Always use a caption under the post
- Always include alt text

Here's an example:

```md
![Markdown badges in a GitHub organization's README](markdown-badges-light.png){: .shadow }{: .light }
![Markdown badges in a GitHub organization's README](markdown-badges-dark.png){: .shadow }{: .dark }
_Markdown badges in a GitHub organization's README_
```

### Style 

- I typically always have an `## Overview` and `## Summary` section

### Frontmatter

For new posts, always use these frontmatter fields (and in this order):

```yml
---
title: 'GitHub Apps: Configuring the Git Email for Commits'
author: Josh Johanning
date: 2024-11-22 12:00:00 -0600
description: A guide on how to set up the proper Git email address for commits made by your GitHub App to ensure proper commit attribution
categories: [GitHub, Apps]
tags: [GitHub, GitHub Actions, GitHub Apps, GitHub Issues, Git]
media_subpath: /assets/screenshots/2024-11-22-github-apps-commit-email
image:
  path: github-app-commit-light.png
  width: 100%
  height: 100%
  alt: A commit from a GitHub app in a GitHub repository with the commit being attributed to the app
---
```

## Documentation

This is the source of truth for other syntax highlighting, formatting, and authoring examples:

- [Text and Typography](https://chirpy.cotes.page/posts/text-and-typography/)
- [Writing a New Post](https://chirpy.cotes.page/posts/write-a-new-post/)
