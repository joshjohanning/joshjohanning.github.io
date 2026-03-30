---
title: 'How I Maintain My Open Source GitHub Actions'
author: Josh Johanning
date: 2026-03-24 22:00:00 -0500
description: A walkthrough of the tools, workflows, and practices I use to maintain a growing collection of open source JavaScript GitHub Actions
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, Open Source, Dependabot, Supply Chain Security, GitHub Copilot, Custom Agents, Repository Management, Configuration as Code]
media_subpath: /assets/screenshots/2026-03-24-maintaining-oss-github-actions
image:
  path: repo-settings-sync-live-light.png
  width: 100%
  height: 100%
  alt: Bulk repository settings sync results showing PRs created across all managed repositories
---

## Overview

I maintain a handful of open source JavaScript GitHub Actions. As the number of repos grew, I kept running into the same friction: Dependabot PRs piling up across repos, settings and configuration drifting, and the tedious ceremony of tagging and releasing each action. So I built (and adopted) a set of tools to keep everything in sync and make the release process as painless as possible.

This post walks through the workflow I have settled on - from repository configuration and dependency management to publishing and supply chain security.

## The Repository Template

Every action I create starts from the same place: [`joshjohanning/nodejs-actions-starter-template`](https://github.com/joshjohanning/nodejs-actions-starter-template). It is a GitHub [template repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template) with the full action layout pre-configured:

- ESLint and Prettier for linting and formatting
- Jest for testing with coverage badge generation
- CI workflow that runs lint, test, and build validation on every PR
- A publish workflow powered by [`joshjohanning/publish-github-action`](https://github.com/joshjohanning/publish-github-action) ([more on this below](#publishing-releases))
- Dependabot configuration
- A `TEMPLATE_CHECKLIST_DELETE_ME.md`{: .filepath} with a customization guide

When I click "[Use this template](https://github.com/new?template_name=nodejs-actions-starter-template&template_owner=joshjohanning)," I get a new repo that already has CI, publishing, and linting ready to go. My goal has been to keep all of my action repos configured as closely as possible - same dev dependencies, similar `package.json`{: .filepath} scripts, same testing structure - so that switching between repos feels seamless.

## Keeping Repos in Sync

The template only helps at creation time. Repos drift. A new ESLint rule gets added to one repo but not the others, a Dependabot config is updated in one place, or you decide to enable immutable releases across the board. That is where [`joshjohanning/bulk-github-repo-settings-sync-action`](https://github.com/joshjohanning/bulk-github-repo-settings-sync-action) comes in (I wrote a [dedicated post](/posts/bulk-github-repo-settings-sync-action/) on this action if you want the deep dive).

I run this from a dedicated configuration repo, [`joshjohanning/sync-github-repo-settings`](https://github.com/joshjohanning/sync-github-repo-settings), which contains:

- A [`repos.yml`](https://github.com/joshjohanning/sync-github-repo-settings/blob/main/repos.yml) listing every repo I manage and any per-repo overrides
- Configuration files for Dependabot, rulesets, copilot instructions, gitignore, package.json scripts/engines, and workflow files
- A GitHub Actions workflow that runs the sync action on push to `main`

When the workflow runs, it applies settings and opens PRs where file content has changed. Here is a sample of what gets synced:

- **Repository settings**: squash-only merges, auto-delete branches, immutable releases, code scanning, secret scanning, Dependabot alerts
- **`dependabot.yml`**: one config per ecosystem (e.g., one for npm-based actions, one for my composite action stragglers)
- **`.gitignore`**: a shared base with a marker for repo-specific entries that get preserved
- **Rulesets**: a CI-required-status-checks ruleset applied to `main`
- **Copilot instructions**: a shared `.github/copilot-instructions.md` so Copilot understands the repo conventions
- **`package.json`**: syncing `scripts` and `engines` fields to keep npm commands and Node.js version requirements consistent
- **Topics**: ensuring every action repo has the same set of GitHub topics
- **Workflow files**: shared CI and publish workflows

The action uses the GitHub API for commits, so PRs created by the sync are signed and verified. In `repos.yml`, each repo can override any global setting - for example, one repo might need a different Dependabot config or different topics.

An [example of a PR created by the sync](https://github.com/joshjohanning/nodejs-actions-starter-template/pull/45) shows the bot syncing a CI workflow file across repos automatically.

{% raw %}
> The action supports a `dry-run` mode. I set `dry-run: ${{ github.event_name == 'pull_request' }}` [in my workflow](https://github.com/joshjohanning/sync-github-repo-settings/blob/6352ce5a8643abecd29f3de3a44fd30097306464/.github/workflows/sync-github-repo-settings.yml#L43) so that PRs against the config repo preview what would change without applying anything, and pushes to `main` apply the changes for real.
{: .prompt-tip }
{% endraw %}

Here is the dry-run output from a PR against the config repo, followed by the applied run after merging to `main`:

![Dry-run results showing what would be synced across repos](repo-settings-sync-dry-run-light.png){: .shadow }{: .light }
![Dry-run results showing what would be synced across repos](repo-settings-sync-dry-run-dark.png){: .shadow }{: .dark }
_Dry-run mode previews changes without applying them_

![Applied run results showing PRs created across repos](repo-settings-sync-live-light.png){: .shadow }{: .light }
![Applied run results showing PRs created across repos](repo-settings-sync-live-dark.png){: .shadow }{: .dark }
_Applied run creates PRs in each repo where files differ_

## Managing Dependabot PRs

With Dependabot enabled across all my repos, I frequently end up with the same dependency bump PR open in all 10 repos at once. Clicking through each one in the UI is not practical.

I use a script from my [`github-misc-scripts`](https://github.com/joshjohanning/github-misc-scripts) repo: [`merge-pull-requests-by-title.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/merge-pull-requests-by-title.sh). It finds and merges matching PRs across repos. You can target repos by owner and filter by topic (multiple `--topic` flags act as an AND filter):

```bash
./merge-pull-requests-by-title.sh --owner joshjohanning --topic javascript "chore(deps): bump undici from 6.23.0 to 6.24.1" squash
```
{: .nolineno}

For dev dependency updates, that is usually enough - merge and move on. Add `--no-prompt` to skip the per-PR confirmation and merge everything automatically. But for **production dependency updates**, my CI requires a version bump ([more on this in the next section](#version-checking-in-ci)). So, I recently [added a `--bump-patch-version` flag](https://github.com/joshjohanning/github-misc-scripts/pull/158) to the script. It clones each matching PR branch, runs `npm version patch`, commits, and pushes - then I can either merge manually or use `--enable-auto-merge` to let CI pass and merge automatically:

```bash
# Bump patch version on all matching PR branches
./merge-pull-requests-by-title.sh --owner joshjohanning --topic javascript "chore(deps): bump undici*" --bump-patch-version

# Once CI passes, merge with auto-merge (no confirmation prompt)
./merge-pull-requests-by-title.sh --owner joshjohanning --topic javascript "chore(deps): bump undici*" --bump-patch-version --enable-auto-merge --no-prompt
```
{: .nolineno}

> That `--bump-patch-version` feature? I used [Copilot coding agent](https://docs.github.com/en/copilot/using-github-copilot/using-copilot-coding-agent-to-work-on-tasks/about-assigning-tasks-to-copilot) to build it. While in bed so I couldn't forget, I pulled up GitHub mobile and assigned the task in an issue. Copilot opened a PR, and the next morning I reviewed, iterated, and merged it by end of day.
{: .prompt-info }

## Version Checking in CI

Forgetting to bump the version in `package.json`{: .filepath} before merging a PR is an easy mistake that causes publishing issues downstream. That is why I built [`joshjohanning/npm-version-check-action`](https://github.com/joshjohanning/npm-version-check-action) and use it in every action's CI workflow.

It runs on pull requests and validates:

1. **Did the version change?** If source code or production dependencies changed, the version in `package.json`{: .filepath} must be incremented
2. **Is it a valid increment?** The new version must be higher than the latest git tag
3. **Are `package.json`{: .filepath} and `package-lock.json`{: .filepath} in sync?** Catches mismatches from rebases or manual edits
4. **Did the runtime change?** If `action.yml`{: .filepath} changes its `runs.using` value (e.g., `node20` to `node24`), a **major** version bump is required

The action is smart about what triggers a version check. It uses file-level change detection and dependency tree analysis:

- **Code changes** (`.js`, `.ts` files): always require a version bump
- **Production dependency changes**: require a version bump
- **Dev dependency changes**: skipped by default (configurable with `include-dev-dependencies: true`)
- **Metadata-only changes** in `package.json`{: .filepath} (description, scripts): no version bump needed

This means Dependabot PRs that only update dev dependencies pass CI without needing a version bump, which is exactly the behavior I want.

![Version check failing because the package version was not incremented](npm-version-check-light.png){: .shadow }{: .light }
![Version check failing because the package version was not incremented](npm-version-check-dark.png){: .shadow }{: .dark }
_The version check catches when you forget to bump the version_

## Publishing Releases

Once a PR is merged to `main`, I need to compile the JavaScript with `ncc`, push the compiled output to a release tag, and update the major version tag. I use [`joshjohanning/publish-github-action`](https://github.com/joshjohanning/publish-github-action) for this.

The workflow is straightforward:

{% raw %}
```yml
name: Publish GitHub Action
on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v6
      - uses: joshjohanning/publish-github-action@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          npm_package_command: npm run package
```
{% endraw %}
{: file='.github/workflows/publish.yml'}

The action:

1. Installs production dependencies
2. Runs `npm run package` (which calls [`ncc`](https://github.com/vercel/ncc), a bundler that compiles Node.js modules into a single file) to bundle everything into `dist/`
3. Pushes the compiled output to a release tag (e.g., `v1.2.3`)
4. Updates the major version tag (e.g., `v1`) to point to the new release
5. Creates a GitHub release with [auto-generated release notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes), so merged PR titles and authors are included automatically (e.g., Dependabot bumps, bot syncs, and feature PRs all show up without any manual effort). [Here's an example](https://github.com/joshjohanning/nodejs-actions-starter-template/releases/tag/v2.0.1)

I do **not** push the compiled `dist/` folder back to the default branch. Some maintainers do this, but I find it messy - it creates noisy diffs and merge conflicts. Instead, the compiled code only lives on the release tags. People shouldn't be referencing `@main` in their workflows anyway - they should be pinning to a specific version tag (e.g., `@v1`, `@v1.2.3`, or a commit SHA) to ensure stability.

> You'll notice my examples use major version tags like `@v3` rather than commit SHAs. Pinning to a SHA or a known immutable release tag is the safest option, but major version tags are more convenient and readable. It comes down to your risk tolerance.
{: .prompt-info }

The action also supports `create_release_as_draft: true`, which is useful when you have [immutable releases](#immutable-releases) enabled (since you cannot modify a release after publishing). This lets you review the release before it goes live.

> The publish action uses the GitHub API for commits when possible, so the commits on the release tag are verified (signed by GitHub).
{: .prompt-tip }

## Supply Chain Security

Security is a big part of how I maintain these actions. Here is what I do:

### No `pull_request_target`

None of my actions use the `pull_request_target` event. This event runs in the context of the base branch and has access to secrets, making it a common vector for supply chain attacks on public repos. I use `pull_request` exclusively.

### Minimal Permissions

The only credential my workflows use is `GITHUB_TOKEN` (via `secrets.GITHUB_TOKEN`), and I always scope permissions down explicitly. No PATs, no broad `repo` scope tokens sitting in secrets.

### Immutable Releases

I enable [immutable releases](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases) on every action repository. This means once a release is published, it cannot be modified or deleted - preventing an attacker (or even a compromised account) from force-pushing the tag after users have pinned to that version.

The `bulk-github-repo-settings-sync-action` enables this across all repos automatically via the `immutable-releases: true` setting.

I also built [`joshjohanning/ensure-immutable-actions`](https://github.com/joshjohanning/ensure-immutable-actions) to help verify this. It scans your workflow files and reports whether the third-party actions you reference are using immutable releases. You can run it in CI to fail the build if any mutable actions are detected:

{% raw %}
```yml
- name: Ensure immutable actions
  uses: joshjohanning/ensure-immutable-actions@v2
```
{% endraw %}
{: .nolineno}

It generates a report organized by workflow, showing which actions are first-party, which have immutable releases, and which are mutable.

> Immutable releases are currently the best mechanism to prove a tag won't be moved or deleted after publishing. However, they don't guarantee that *future* releases will also be immutable - a repo owner can disable the setting at any time. So while you can verify a specific version is immutable today, you can't enforce that the next release will be.
{: .prompt-warning }

## Upgrading the Node.js Runtime

GitHub Actions recently [announced the deprecation of the Node.js 20 runtime](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) for actions runners. All JavaScript actions need to migrate to `node24`.

For my actions, this means updating `runs.using` in `action.yml`{: .filepath} from `node20` to `node24`, bumping the major version (since `npm-version-check-action` enforces this), and updating CI workflows and documentation.

I created a [Copilot agent for this upgrade process](https://github.com/github/awesome-copilot/blob/main/agents/github-actions-node-upgrade.agent.md) and [contributed it](https://github.com/github/awesome-copilot/pull/1118) to the [`github/awesome-copilot`](https://github.com/github/awesome-copilot) repository. The agent handles the full lifecycle:

- Updating `action.yml`
- Running `npm version major`
- Updating CI workflow Node.js versions
- Updating documentation references
- Scanning for Node.js API incompatibilities
- Generating commit messages and PR descriptions

I used this agent and [Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/about-copilot-cli) to upgrade all of my actions from `node20` to `node24`.

## The Full Picture

Here is how it all fits together:

1. **Create a new action** from the [`nodejs-actions-starter-template`](https://github.com/joshjohanning/nodejs-actions-starter-template)
2. **Add it to `repos.yml`** in the [`sync-github-repo-settings`](https://github.com/joshjohanning/sync-github-repo-settings) repo to start receiving synced settings, Dependabot config, rulesets, copilot instructions, etc.
3. **Develop** with CI feedback from `npm-version-check-action` ensuring versions are bumped correctly
4. **Merge to `main`** and `publish-github-action` handles tagging, compiling, and releasing
5. **Dependabot PRs** roll in; use `merge-pull-requests-by-title.sh` to handle them in bulk (with `--bump-patch-version` for prod deps)
6. **Supply chain security** is enforced via immutable releases, `ensure-immutable-actions`, no `pull_request_target`, and scoped-down `GITHUB_TOKEN` permissions
7. **Runtime upgrades** and other heavy lifts are handled using Copilot CLI and/or custom agents for consistent, repeatable migrations

## Summary

Maintaining open source actions at scale is less about writing the action code and more about the infrastructure around it. The combination of a template repo, a centralized settings sync, bulk Dependabot PR management, automated version checking, and a one-step publish workflow means I can spend my time on features rather than maintenance toil.

If you maintain multiple GitHub Actions (or really any set of repos that should stay in sync), I'd encourage you to implement something similar. The upfront investment pays off quickly once you have more than a couple of repos to manage.

Here are the relevant repos and tools:

| Tool | Purpose |
|------|---------|
| [`nodejs-actions-starter-template`](https://github.com/joshjohanning/nodejs-actions-starter-template) | Template repo for new JavaScript actions |
| [`bulk-github-repo-settings-sync-action`](https://github.com/joshjohanning/bulk-github-repo-settings-sync-action) | Sync settings, files, and configs across repos |
| [`sync-github-repo-settings`](https://github.com/joshjohanning/sync-github-repo-settings) | My configuration repo that drives the sync |
| [`npm-version-check-action`](https://github.com/joshjohanning/npm-version-check-action) | CI check for version bumps in PRs |
| [`publish-github-action`](https://github.com/joshjohanning/publish-github-action) | Compile and release JavaScript actions |
| [`ensure-immutable-actions`](https://github.com/joshjohanning/ensure-immutable-actions) | Verify third-party actions use immutable releases |
| [`merge-pull-requests-by-title.sh`](https://github.com/joshjohanning/github-misc-scripts/blob/main/gh-cli/merge-pull-requests-by-title.sh) | Bulk merge Dependabot PRs across repos |

If you have suggestions for improving my workflows or tools I should check out, I'd love to hear about them - please leave a comment! ✨
