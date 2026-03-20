---
description: 'Upgrade the Jekyll Chirpy theme to a new version using the cherry-pick method'
agent: 'agent'
argument-hint: 'Target version (e.g. v7.5.0)'
---

# Chirpy Theme Upgrade

Upgrade the Jekyll Chirpy theme to a new version using the cherry-pick method.

## Context

- This is a Jekyll blog using the [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy) without the theme gem (customizations are made directly)
- The upstream remote is named `chirpy` and points to `https://github.com/cotes2020/jekyll-theme-chirpy.git`
- Upgrades are done by cherry-picking commits between release tags from the upstream repo
- Full upgrade instructions are in the repo's [README.md](../../README.md) under "Upgrading the Theme"

## Instructions

1. **Determine versions**: Check the current version in `jekyll-theme-chirpy.gemspec` (the `spec.version` field). The target version should be provided by the user or found at the upstream [releases page](https://github.com/cotes2020/jekyll-theme-chirpy/releases).

2. **Fetch upstream**: `git fetch chirpy --tags`

3. **Create branch**: `git checkout -b chirpy-v{TARGET_VERSION}` (e.g., `chirpy-v7.5.0`)

4. **Cherry-pick the range**: `git cherry-pick "v{CURRENT}..v{TARGET_VERSION}" -m 1`

5. **Handle merge conflicts** — these are the expected patterns:
   - **Files deleted in our repo but modified upstream** (e.g., `docs/CHANGELOG.md`, `.github/dependabot.yml`, upstream workflow files like `ci.yml`/`cd.yml`/`codeql.yml`/`commitlint.yml`/`lint-js.yml`/`lint-scss.yml`/`pr-filter.yml`, starter posts like `_posts/2019-08-*`): Use `git rm <file>` to remove them, then `git cherry-pick --continue --no-edit`. If the commit becomes empty, use `git cherry-pick --skip`.
   - **`_config.yml` conflicts**: Keep our customized values (name, email, URLs, etc.) but accept new upstream fields/features. Watch for the social section — always keep `name: Josh Johanning` and `email: jjohanning@uwalumni.com`.
   - **`Gemfile` conflicts**: Accept the upstream structural changes but evaluate whether to keep any custom gems we've added. Check if the upstream CI passes without them before removing.

6. **Build JS assets** (required since Chirpy v5.6.0):
   ```sh
   npm run build && git add assets/js/dist _sass/vendors -f && git commit -m "update js assets"
   ```

7. **Squash all commits into one** via interactive rebase:
   ```sh
   GIT_SEQUENCE_EDITOR="sed -i '' '2,\$s/^pick/squash/'" \
   GIT_EDITOR="printf 'updating to chirpy v{TARGET_VERSION}' >" \
   git rebase -i HEAD~{N}
   ```
   Where `{N}` is the total number of commits (cherry-picked + 1 for the asset build commit).

8. **Check for unwanted new files**: `git diff-tree --compact-summary -r HEAD --diff-filter=A`
   - Delete any new GitHub workflows, issue templates, or starter content that shouldn't be in our repo
   - If cleanup is needed, commit the deletions and do another squash rebase (`git rebase -i HEAD~2`)

9. **Amend the final commit** with proper author and timestamp:
   ```sh
   git commit --amend --author "Josh Johanning <joshjohanning@github.com>" --date=now -S
   ```

10. **Test locally**:

   ```sh
   bundle install
   npm i && npm run build
   bundle exec jekyll build
   bundle exec jekyll serve
   ```

   Verify the site builds and serves at `http://127.0.0.1:4000/` with HTTP 200.

11. **DO NOT create a PR automatically** — leave the branch ready for manual review and PR creation.

## Known Deviations from Upstream

These customizations must be preserved during upgrades:

- **Preview image aspect ratio**: In `_sass/base/_base.scss`, the line `aspect-ratio: 40 / 21;` is commented out to prevent cropping of some preview images. Do NOT uncomment it.
- **Speaking tab**: A custom speaking tab exists in `_tabs/speaking.md` — don't remove it.
- **Custom social links**: Our `_config.yml` has customized social links, comment system (Utterances), and site verification settings.
- **Deploy workflow**: Our deploy workflow lives at `.github/workflows/pages-deploy.yml`. Upstream changes from `starter/pages-deploy.yml` should be accepted as long as git maps the file correctly (see [Starter Workflow Mapping](#️-starter-workflow-mapping) section below).

## PR Format

When creating a PR manually, follow this format:
- **Branch name**: `chirpy-v{TARGET_VERSION}`
- **PR title**: `updating to chirpy v{TARGET_VERSION}`
- **PR body**: `- updating from chirpy v{CURRENT} to [v{TARGET_VERSION}](https://github.com/cotes2020/jekyll-theme-chirpy/releases/tag/v{TARGET_VERSION})`
- **Merge type**: Merge commit (not squash or rebase)

## Troubleshooting

- If `bundle exec jekyll build` fails after upgrade, check for breaking changes in the [upstream release notes](https://github.com/cotes2020/jekyll-theme-chirpy/releases)
- If `npm run build` fails, try `npm i` first to update node dependencies
- If GPG signing causes issues during cherry-pick, disable temporarily: `git config commit.gpgsign false` — but remember to re-enable: `git config commit.gpgsign true`
- The "Merge branch 'master' into production" commits from upstream often duplicate all the conflicts you already resolved — just resolve them the same way again

## ⚠️ Starter Workflow Mapping

Our deploy workflow lives at `.github/workflows/pages-deploy.yml`, but upstream's version lives at `.github/workflows/starter/pages-deploy.yml` (different path). During cherry-pick, git's content detection usually matches the patch to our file automatically because the contents are similar enough. However:

- **Always verify** that action version bumps from the upstream starter workflow actually landed in our `pages-deploy.yml`
- Compare the upstream diff for `starter/pages-deploy.yml` against our `pages-deploy.yml` after the cherry-pick
- If the contents diverge too much, git may silently skip the patch — in that case, manually apply the changes
- You can check the upstream starter diff with: `git diff v{CURRENT}..v{TARGET} -- .github/workflows/starter/pages-deploy.yml`
