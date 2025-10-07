---
title: 'GitHub Discussions Migration Utility'
author: Josh Johanning
date: 2025-10-07 12:00:00 -0500
description: A utility to migrate GitHub Discussions between repositories, including categories, labels, comments, and replies
categories: [GitHub, Migrations]
tags: [GitHub, Scripts, GitHub Discussions, Migrations, Node.js]
media_subpath: /assets/screenshots/2025-10-07-github-discussions-migration-utility
image:
  path: github-discussions-light.png
  width: 100%
  height: 100%
  alt: GitHub Discussions in a repository
---

## Overview

GitHub Discussions are a great way to facilitate conversations within your repository or organization, but what happens when you need to move them? Whether you're migrating between GitHub instances (server to cloud, cloud to cloud, etc.), moving discussions to a different organization, or consolidating repositories across organizations, moving discussions manually can be a tedious process. Today, the GitHub Enterprise Importer (GEI) [does not support](https://docs.github.com/en/enterprise-cloud@latest/migrations/using-github-enterprise-importer/migrating-between-github-products/about-migrations-between-github-products#data-that-is-not-migrated-1) migrating discussions, so I built a [Node.js utility](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-discussions/migrate-discussions.js) ([README](https://github.com/joshjohanning/github-misc-scripts/blob/main/scripts/migrate-discussions/README.md)) to help with this task.

The utility handles discussions, comments, replies, categories, labels, and even preserves metadata like reactions and poll results. It includes intelligent rate limiting and resume capabilities to handle large migrations reliably. I should note, "reliably" does not mean quickly - GitHub has strict rate limits on content-creating operations to prevent abuse, so patience is key when migrating large numbers of discussions. (See [rate limiting details](#rate-limiting-details) below for more information.)

> See my [GitHub Migration Tools Collection](/posts/github-migration-tools/) post.
{: .prompt-info }

> If you're moving discussions within the same organization, use GitHub's native [transfer discussion](https://docs.github.com/en/discussions/managing-discussions-for-your-community/managing-discussions#transferring-a-discussion) feature instead. This utility is best for cross-organization or cross-instance migrations, or when you need to automate transfers (since there's no GraphQL endpoint for the native transfer feature).
{: .prompt-tip }

## Running the Script

### Prerequisites

1. A clone of my [github-misc-scripts](https://github.com/joshjohanning/github-misc-scripts) repo
2. Node.js installed
3. Both source and target repositories must have GitHub Discussions enabled
4. GitHub tokens with appropriate permissions:
   - Source token: `repo` scope with read access to discussions
   - Target token: `repo` scope with write access to discussions
   - GitHub App tokens recommended for better rate limits (but note they expire after 1 hour - see [Future Improvements](#future-improvements))
5. Dependencies installed via `npm i`

You can generate a GitHub App token using the [`gh-token`](https://github.com/Link-/gh-token) extension:

```bash
export SOURCE_TOKEN=$(gh token generate --app-id YOUR_APP_ID --installation-id YOUR_INSTALLATION_ID --key /path/to/private-key.pem --token-only)
export TARGET_TOKEN=$(gh token generate --app-id YOUR_APP_ID --installation-id YOUR_INSTALLATION_ID --key /path/to/private-key.pem --token-only)
```
{: .nolineno}

### Usage

You can call the script via:

```bash
cd github-misc-scripts
export SOURCE_TOKEN=ghp_abc
export TARGET_TOKEN=ghp_xyz
cd ./scripts/migrate-discussions
npm i
node ./migrate-discussions.js source-org source-repo target-org target-repo
```
{: .nolineno}

To resume from a specific discussion number (useful if interrupted):

```bash
node ./migrate-discussions.js source-org source-repo target-org target-repo --start-from 50
```
{: .nolineno}

### Example

An example of this in practice:

```bash
export SOURCE_TOKEN=ghp_abc
export TARGET_TOKEN=ghp_xyz
cd ./scripts/migrate-discussions
npm i
node ./migrate-discussions.js joshjohanning-org discussions-source joshjohanning-org discussions-target
```
{: .nolineno}

For GitHub Enterprise Server instances, set the API URL environment variables:

```bash
export SOURCE_API_URL=https://github.mycompany.com/api/v3
export TARGET_API_URL=https://api.github.com
export SOURCE_TOKEN=ghp_abc
export TARGET_TOKEN=ghp_xyz
node ./migrate-discussions.js source-org source-repo target-org target-repo
```
{: .nolineno}

## Features

The script migrates discussions with comprehensive support for:

### Content Migration

- **Discussion categories** - Automatically creates missing categories in the target repository (or uses "General" as fallback)
- **Labels** - Creates labels in the target repository if they don't exist
- **Comments and replies** - Copies all comments and threaded replies with proper attribution
- **Poll results** - Copies poll results as static snapshots with tables and optional Mermaid charts
- **Reactions** - Preserves reaction counts on discussions, comments, and replies
- **Discussion states** - Maintains locked and closed status
- **Answered discussions** - Marks answered discussions and preserves the accepted answer
- **Pinned discussions** - Indicates pinned discussions with a visual indicator (GraphQL API doesn't support pinning)

### Rate Limiting and Reliability

- **Automatic rate limit handling** - Uses Octokit's built-in throttling plugin with retry logic
- **GitHub-recommended delays** - Waits 3 seconds between discussions/comments to stay under secondary rate limits
- **Resume capability** - Use `--start-from <number>` to resume from a specific discussion if interrupted
- **Configurable retries** - Retries up to 15 times for both rate-limit and non-rate-limit errors

## Summary Output

After completion, the script displays comprehensive statistics:

```text
[2025-10-02 19:38:44] ============================================================
[2025-10-02 19:38:44] Discussion copy completed!
[2025-10-02 19:38:44] Total discussions found: 10
[2025-10-02 19:38:44] Discussions created: 10
[2025-10-02 19:38:44] Discussions skipped: 0
[2025-10-02 19:38:44] Total comments found: 9
[2025-10-02 19:38:44] Comments copied: 9
[2025-10-02 19:38:44] Primary rate limits hit: 0
[2025-10-02 19:38:44] Secondary rate limits hit: 0
[2025-10-02 19:38:44] WARNING: 
The following categories were missing and need to be created manually:
[2025-10-02 19:38:44] WARNING:   - Blog posts!
[2025-10-02 19:38:44] WARNING: 
[2025-10-02 19:38:44] WARNING: To create categories manually:
[2025-10-02 19:38:44] WARNING: 1. Go to https://github.com/joshjohanning-emu/discussions-test/discussions
[2025-10-02 19:38:44] WARNING: 2. Click 'New discussion'
[2025-10-02 19:38:44] WARNING: 3. Look for category management options
[2025-10-02 19:38:44] WARNING: 4. Create the missing categories with appropriate names and descriptions
[2025-10-02 19:38:44] 
All done! ✨
```

## Limitations & Notes

### What It Doesn't Do

While the utility handles most discussion migration scenarios, there are some limitations:

- **Discussion categories** - Cannot create categories via API; must be created manually in the target repository beforehand (discussions will use "General" as fallback)
- **Pinning discussions** - GitHub API doesn't allow pinning via GraphQL; pinned status is indicated in the discussion body
- **Live polls** - Poll results are copied as static snapshots; users cannot vote on migrated polls
- **Attachments** - Images and files referenced in discussions may not copy over and require manual handling
- **Reactions** - Copied as read-only summaries; users cannot add new reactions to migrated content

### Category Handling

- Discussion categories must exist in the target repository before running the script
- If a category doesn't exist, discussions will be created in the "General" category as a fallback
- Missing categories are tracked and reported at the end of the script

### Rate Limiting Details

GitHub limits content-generating requests to avoid abuse:

- No more than 80 content-generating requests per minute
- No more than 500 content-generating requests per hour

The script automatically handles this by:

- Staying under 1 discussion or comment created every 3 seconds (GitHub's recommendation)
- Automatic retry with wait times from GitHub's `retry-after` headers
- Retrying up to 15 times if rate limits are consistently hit

### Configuration Options

You can edit these constants at the top of the script to customize behavior:

- `INCLUDE_POLL_MERMAID_CHART` - Set to `false` to disable Mermaid pie charts for polls (default: `true`)
- `RATE_LIMIT_SLEEP_SECONDS` - Sleep duration between API calls (default: `0.5` seconds)
- `DISCUSSION_PROCESSING_DELAY_SECONDS` - Delay between processing discussions/comments (default: `3` seconds)
- `MAX_RETRIES` - Maximum retries for both rate-limit and non-rate-limit errors (default: `15`)

### Content Preservation

- The script adds attribution text to preserve original author and timestamp information (the API doesn't allow setting creation date or author metadata - all migrated content will show as created by the token user)
- Poll results are copied as static snapshots - voting is not available in copied discussions
- Reactions are copied as read-only summaries
- Locked discussions will be locked in the target repository
- Closed discussions will be closed in the target repository
- Answered discussions will have the same comment marked as the answer

## Future Improvements

- [ ] Native support for GitHub Apps and ability to fetch new tokens before they expire in an hour

## Summary

This utility makes it possible to migrate GitHub Discussions between repositories, whether you're moving between GitHub instances or just consolidating your discussions. The script handles the heavy lifting of copying discussions, comments, and metadata while respecting GitHub's rate limits.

Drop a comment here or an issue or PR on my [github-misc-scripts repo](https://github.com/joshjohanning/github-misc-scripts/tree/main/scripts/migrate-discussions) if you have any feedback or suggestions! Happy migrating! 🚀
