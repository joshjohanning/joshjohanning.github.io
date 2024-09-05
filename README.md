# josh-ops.com

## Overview

A DevOps Blog - Blogging about GitHub and Azure DevOps practices, tips, scripts, and my continuous improvement DevOps journey.

[**josh-ops.com →**](https://josh-ops.com)

[![Build and Deploy](https://github.com/joshjohanning/joshjohanning.github.io/actions/workflows/pages-deploy.yml/badge.svg?branch=main)](https://github.com/joshjohanning/joshjohanning.github.io/actions/workflows/pages-deploy.yml)

## Theme Source

Chirpy:

* [GitHub repo](https://github.com/cotes2020/jekyll-theme-chirpy)
* [Example and tips/best practices](https://chirpy.cotes.page/)
* [Upgrading](#upgrading-the-theme) (using `git cherry-pick` to pull changes from upstream)

## Comment System

- [Utterances](https://utteranc.es/) (configured [directly in Chirpy](https://github.com/joshjohanning/joshjohanning.github.io/blob/a54c9633e6cab32fd30dc69afc9ffd74857cbd8a/_config.yml#L84-L92)) which uses GitHub issues for post comments

## Deviations from Chirpy

### Adding Speaking tab

- Added a [speaking tab](https://josh-ops.com/speaking/) to capture my speaking engagements
- [Used an icon](https://github.com/joshjohanning/joshjohanning.github.io/blob/ab7bb6e3842189adf1dccc909e1e77b86b625d0a/_tabs/speaking.md?plain=1#L3) from [fontawesome](https://fontawesome.com/v4/icons/) for the link in the sidebar

### Light Mode Sidebar Background Color

- For my implementation of Chirpy v4.3.0 to v6.1.0, I [reverted](https://github.com/joshjohanning/joshjohanning.github.io/pull/8) the light mode sidebar background color to the pre-v4.3.0 color (blue/purple)
- When I updated from [Chirpy v6.1.0 to v6.3.0](https://github.com/joshjohanning/joshjohanning.github.io/pull/30), I decided to use the latest upstream values for the light mode sidebar background color (light gray)

#### Changelog

- See: [#8](https://github.com/joshjohanning/joshjohanning.github.io/pull/8) where I reverted to the pre-v4.3.0 color (blue/purple)
- In [#27](https://github.com/joshjohanning/joshjohanning.github.io/pull/27), I updated the `sidebar-active-color` property to the latest upstream value
- In [#30](https://github.com/joshjohanning/joshjohanning.github.io/pull/30), I reverted to the latest upstream values for light mode, which included a change to the `sidebar-bg` and `sidebar-muted-color` properties to bring in the light gray sidebar background color

### Preview Images

- Chirpy [v5.4.0](https://github.com/cotes2020/jekyll-theme-chirpy/commit/4b6ccbcbccce27b9fcb035812efefe4eb69301cf) introduced a strict `40 / 21` (`1:91:1`) aspect ratio requirement for post's preview images such that they would be cropped if you used a different aspect ratio
- In prior versions, I removed this code so that the post's preview images would still render with their original size
- In June 2023, I updated most of the preview images with the new aspect ratio and [brought back](https://github.com/joshjohanning/joshjohanning.github.io/commit/1920dc7d98cbe11a6882ae0ec067fabccd64426b) preview images to the home page, but I still left out the `40 / 21;` line from the `post.scss` file to account for the images that weren't updated
- In November 2023, I updated to Chirpy v6.2.3 and the preview image logic was moved to `commons.scss`; removed the `40 / 21;` line for the non-updated images

#### Changelog

- Upstream commit introducing change: [4b6ccbc](https://github.com/cotes2020/jekyll-theme-chirpy/commit/4b6ccbcbccce27b9fcb035812efefe4eb69301cf) (Chirpy [v5.4.0](https://github.com/cotes2020/jekyll-theme-chirpy/releases/tag/v5.4.0))
- My changes so that preview image still shimmers before loading, but no image cropping: [b282712^..bb1dc1f](https://github.com/joshjohanning/joshjohanning.github.io/compare/b282712087028da95e292e3159d20cdf63d59feb^..bb1dc1f1bdbba4ee7d62858d834e0ca19f7745db)
  - Really only need to get rid of `aspect-ratio: 40 / 21;` line
- June 20, 2023: [Updated](https://github.com/joshjohanning/joshjohanning.github.io/commit/af83c7019c5783f70d5e725991097a7217a6658a) most of the post images to reflect the `1.91:1` aspect ratio since that's the ratio the [home page expects](https://github.com/joshjohanning/joshjohanning.github.io/commit/1920dc7d98cbe11a6882ae0ec067fabccd64426b) for the post preview images
  - I still left out the `40 / 21;` line in the `post.scss` file for the images I didn't update to show the original image size on the post page
- November 1, 2023: In Chirpy [v6.2.3](https://github.com/joshjohanning/joshjohanning.github.io/pull/30), the preview image logic was moved to `commons.scss`; removed the `40 / 21;` line for the non-updated images

## Upgrading the Theme

Since we aren't using the theme gem (so we can do customizations), we have to do it the old-fashioned way:

1. Ensure chirpy is set as a remote: `git remote add chirpy https://github.com/cotes2020/jekyll-theme-chirpy.git`
2. Ensure you have the latest upstream commit: `git fetch chirpy`
3. Compare the upstream [releases](https://github.com/cotes2020/jekyll-theme-chirpy/releases) and [commits](https://github.com/cotes2020/jekyll-theme-chirpy/commits/master) to find the first and last release/commit in the range you want to update
    - Recommendation is to use release tag milestones instead of loose commits that aren't part of a release yet
    - You can use this [link](https://github.com/cotes2020/jekyll-theme-chirpy/compare/a887f1d^..602e984) to compare the changes between two commits in GitHub (same for [releases](https://github.com/cotes2020/jekyll-theme-chirpy/compare/v5.6.0..v5.6.1))
4. Start the `git cherry-pick`:
    - To cherry-pick between a range of release tags (more common): `git cherry-pick "v5.6.0..v5.6.1" -m 1`
    - To cherry-pick a single commit (not as common): `git cherry-pick a887f1d -m 1`
    - If getting GPG errors, modify the local git config: `git config commit.gpgsign false`, but modify it back to `true` after you are done cherry-picking and rebasing (before amending commit)
5. Review merge conflicts - use a combination of `git cherry-pick --skip` (for when readme/starter posts are updated) and `cherry-pick --continue` (to continue after you resolve real merge conflicts)
6. Starting in Chirpy v5.6.0, run: `npm run build && git add assets/js/dist _sass/dist -f && git commit -m "update js assets"` ([docs](https://github.com/cotes2020/jekyll-theme-chirpy/wiki/Upgrade-Guide#upgrade-the-fork))
   - You can also run a command that's referenced in the `init.sh` to remove this from `.gitignore`: `sed -i '' "/.*\/dist$/d" .gitignore`
7. Rebase the number of commits you just brought in (you should see icon in VS Code): `git rebase -i HEAD~16`
    - Leave the top commit as `pick` but change the rest to `squash`
    - Update the commit message as appropriate
8. Pay close attention to the terminal output as to which new files are being created and if they should be deleted (new files show up as `create mode 100644 file.ext`)
    - For example, we wouldn't want to commit a GitHub workflow or issue template that wasn't needed here
    - If there are new files that we don't want to track, delete the files, commit, and run another rebase `git rebase -i HEAD~2`
    - This command can help with tracking new files in the most recent commit: `git diff-tree --compact-summary -r HEAD --diff-filter=A`
9.  Ensure commit signing is enabled: `git config commit.gpgsign true`
10. Update author and commit time: `git commit --amend --author "Josh Johanning <joshjohanning@github.com>" --date=now -S`
11. [Test changes locally before pushing](#building--testing-locally)

## Building / Testing Locally

```sh
bundle install
npm i && npm run build
bundle exec jekyll s
```

### Additional build notes

#### On macOS

Check ruby version: `ruby -v` (if ruby 2.6.10p210, then you need to upgrade to 3.0.0+):

1. Install Ruby via Homebrew: `brew install ruby` (can also use [`rvm`](https://rvm.io/rvm/install))
2. Make sure the new Ruby is in your path: `export PATH="/opt/homebrew/opt/ruby/bin:$PATH"' >> ~/.zshrc`
3. Check ruby version: `ruby -v` (should be 3.0.0+)
4. Build and serve the site as normal

#### On Codespaces

If seeing a `racc 1.6.2` permission error, run:

```sh
sudo chown -R codespace /usr/local/rvm/gems/ruby-3.1.4/extensions/x86_64-linux/3.1.0
bundle install
```
