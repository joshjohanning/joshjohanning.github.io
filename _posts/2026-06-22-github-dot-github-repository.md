---
title: 'Everything you can do with .github and .github-private repositories'
author: Josh Johanning
date: 2026-06-22 15:00:00 -0500
description: A centralized reference for .github and .github-private repository features, including required files and visibility on GitHub.com, GHEC, EMU, GHEC with data residency, and GHES
categories: [GitHub, Organizations]
tags: [GitHub, GitHub Actions, GitHub Copilot, GitHub Issues, EMU]
media_subpath: /assets/screenshots/2026-06-22-github-dot-github-repository
image:
  path: enterprise-copilot-config-light.png
  width: 100%
  height: 100%
  alt: Enterprise Copilot configuration UI showing .github-private as the source for custom agents
---

## Overview

GitHub gives organizations two specially-named repositories - `.github` and `.github-private` - that act as the home for several org-wide defaults and features, including community profiles, default issue templates, starter workflows, and org-wide Copilot custom agents.

I wrote this because customers have asked if there is one centralized GitHub Docs page that lists every feature powered by these repositories. There are good docs for each individual feature, but not a single reference that answers, "_What can I put in `.github` or `.github-private`, where does it go, and what visibility does the repo need?_"

Each feature has its own **required visibility**, and those requirements differ between regular GitHub.com / GHEC, EMU (Enterprise Managed Users), GHEC with data residency, and GHES. If the repo has the wrong visibility, the feature usually just doesn't apply.

This post collects the features that live in `.github` or `.github-private`, the files involved, and the repo visibility each one needs.

## The Two Repositories at a Glance

The repos are typically used like this:

- **`.github`** (typically public or internal) → community health files, public org profile README, GitHub Actions workflow templates, Copilot custom agents, and Copilot settings/plugin standards
- **`.github-private`** (typically private, sometimes internal) → member-only org profile README, Copilot custom agents, and enterprise-oriented Copilot settings/plugin standards

For Copilot features, both repos can be valid depending on the feature and scope. For enterprise-level custom agent sharing across organizations, the docs still point to a designated `.github-private` repository.

### Feature and visibility reference

| Feature | Repo | GitHub.com / GHEC | GHEC + EMU / GHEC-DR | GHES |
| --- | --- | --- | --- | --- |
| [Community health files (most)](#feature-1-default-community-health-files) | `.github` | Public or Internal | Internal | Public or Internal |
| [Org-wide issue / PR templates](#feature-1-default-community-health-files) | `.github` | **Public only** | Not available | **Public only** |
| [Public org profile README](#feature-2-public-organization-profile-readme) | `.github` | **Public** | Not available | **Public** |
| [Member-only org profile README](#feature-3-member-only-organization-profile-readme) | `.github-private` | **Private only** | **Private only** | **Private only** |
| [Workflow templates](#feature-4-github-actions-workflow-templates-starter-workflows) | `.github` | Public / Internal / Private | **Internal only** (private doesn't serve templates) | Public / Internal / Private (≥ 3.21) |
| [Copilot custom agents - org scope](#feature-5-copilot-custom-agents-org--enterprise-scope) | `.github` or `.github-private` | Depends on selected repo | Internal or Private | Depends on selected repo |
| [Copilot custom agents - enterprise scope](#feature-5-copilot-custom-agents-org--enterprise-scope) | Designated `.github-private` repo | Internal or Private | Internal or Private | Internal or Private |
| [Copilot plugin marketplace and enterprise standards](#feature-6-copilot-plugin-marketplace-and-enterprise-standards) | `.github` or `.github-private` | Depends on selected repo | Internal or Private | Depends on selected repo |

For Copilot rows that say "depends on selected repo," use the visibility supported by that repo on your platform: `.github` follows `.github` visibility behavior, while `.github-private` supports internal or private for these Copilot use cases. For related files that do not work as org-wide defaults, see [Copilot files that are not org-wide defaults](#what-about-copilot-instructionsmd-agentsmd-skills-hooks-and-prompts).

## Feature 1: Default Community Health Files

GitHub's [default community health files](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file) feature lets files in the org's `.github` repo serve as **org-wide defaults** for any repo that doesn't have its own copy. This is the original purpose of the `.github` repo.

### Supported files

| File / Path | Purpose |
| --- | --- |
| `CODE_OF_CONDUCT.md` | Community code of conduct |
| `CONTRIBUTING.md` | Contribution guidelines |
| `GOVERNANCE.md` | Project governance |
| `SECURITY.md` | Security vulnerability reporting |
| `SUPPORT.md` | Support resources |
| `FUNDING.yml` | Sponsor button configuration |
| `.github/ISSUE_TEMPLATE/` (+ `config.yml`) | Issue templates and issue forms |
| `.github/PULL_REQUEST_TEMPLATE/` or `pull_request_template.md` | PR templates |
| `.github/DISCUSSION_TEMPLATE/*.yml` | Discussion category forms |

**Not supported as org defaults:** `LICENSE`, `CODEOWNERS`, `dependabot.yml`. Those are per-repo only.

### File location precedence (inside the `.github` repo)

GitHub searches in this order:

1. `.github/` folder
2. Root of the repo
3. `docs/` folder

The exception is **issue templates**, which must live in `.github/ISSUE_TEMPLATE/`.

### Visibility requirements

| Platform | `.github` repo visibility | Notes |
| --- | --- | --- |
| GitHub.com (free/pro/team) | **Public** | Only option |
| GHEC (non-EMU) | **Public** or **Internal** | Issue/PR templates require **public** |
| GHEC + EMU | **Internal** | Issue/PR templates **don't work** at all (EMUs can't have public repos) |
| GHES | **Public** or **Internal** | Issue/PR templates require **public** |

> **Issue and PR template visibility:** Org-wide **issue templates and PR templates always require a public `.github` repository**, even on GHEC and GHES. An internal `.github` repo will serve most other health files, but issue/PR templates silently won't apply. Since EMU orgs currently can't create public repos, **EMU organizations cannot use org-wide issue/PR templates**.
{: .prompt-warning }

> **Issue template labels:** If a default issue template assigns a label, that label must exist in **both** the `.github` repo and the target repo where the template is used, or the template silently won't apply it.
{: .prompt-info }

## Feature 2: Public Organization Profile README

A [public organization profile README](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/customizing-your-organizations-profile) is shown on your org profile page to people who are **not members** of the organization.

### Setup

- Create a **public** `.github` repo
- Add `profile/README.md` to it
- Supports full GitHub Flavored Markdown, emoji, images, GIFs

### Visibility requirements

| Platform | `.github` repo visibility |
| --- | --- |
| GitHub.com / GHEC (non-EMU) | **Public** |
| GHEC + EMU | Not available (no public repos allowed) |
| GHES | **Public** |

The docs explicitly call this out: *"Public organization profiles are not available with EMUs."*

## Feature 3: Member-Only Organization Profile README

A [member-only organization profile README](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/customizing-your-organizations-profile) is shown only to **members** of the org, on the member view of the profile page.

### Setup

- Create a **private** repo named `.github-private` (note: this is a **different repo** from `.github`)
- Add `profile/README.md` to it

### Visibility requirements

| Platform | `.github-private` repo visibility |
| --- | --- |
| All platforms | **Private** (confirmed - internal does not work) |

I actually tested this myself: if you flip `.github-private` from private to internal, the org member-only README disappears. For this feature, the repo must be private.

## Feature 4: GitHub Actions Workflow Templates (Starter Workflows)

[Workflow templates](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates) (starter workflows) let you publish reusable workflow templates that members of the org see when creating a new workflow in any repo.

### Setup

Stored in `.github/workflow-templates/` of the org's `.github` repo. Each template needs two files:

- `your-template.yml` - the workflow itself
- `your-template.properties.json` - metadata (name, description, icon, categories)

### Visibility matrix

Templates "cascade down" - a more open `.github` repo can serve templates to more restrictive target repos:

| `.github` repo visibility | Templates available to |
| --- | --- |
| **Public** | All repos (public, internal, private) |
| **Internal** | Internal and private repos only |
| **Private** | Private repos only |

### Platform-specific behavior

| Platform | Notes |
| --- | --- |
| GitHub.com / GHEC (non-EMU) | Any visibility works (public, internal, private) |
| GHEC + EMU / GHE.com (data residency) | **Internal only** (see below) |
| GHES ≥ 3.21 | Any visibility works |
| GHES < 3.21 | **Public only** |

Starter workflows from non-public `.github` repos was a fairly recent change - see the [September 2025 changelog](https://github.blog/changelog/2025-09-18-actions-yaml-anchors-and-non-public-workflow-templates/#use-workflow-templates-from-non-public-github-repositories) for the announcement.

> **EMU / GHE.com workflow templates:** On EMU and GHE.com (data residency), a **private** `.github` repo does *not* serve workflow templates - even to other private repos in the org. The templates don't appear in the "new workflow" picker. You need the `.github` repo to be **internal**. Since EMU/GHE.com can't have public repos, **internal is the only working option I found**. This contradicts the documented cascade rules, which say a private `.github` should at least serve private repos, but that is not the behavior I observed on EMU/GHE.com.
{: .prompt-warning }

> The non-public workflow templates change applies **only to GitHub Actions workflow templates**. The changelog is explicit: *"These changes only apply to GitHub Actions - other products that use `.github` repositories (e.g., GitHub Issues) will continue to only use public `.github` repositories."* So issue/PR templates are still public-only.
{: .prompt-warning }

## Feature 5: Copilot Custom Agents (Org / Enterprise Scope)

[Custom agents](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/invoke-custom-agents) for Copilot (CLI and the Copilot coding agent) can be defined at user, repository, org, or enterprise scope. The org/enterprise-scoped versions live in either `.github` or `.github-private`, but **the rules differ between org-shared and enterprise-shared agents**.

### Setup

Per the [October 2025 changelog](https://github.blog/changelog/2025-10-28-custom-agents-for-github-copilot/):

> *"By adding a configuration file under `.github/agents` in any repository or in the `agents` folder in the `{org}/.github` or `{org}/.github-private` repository, you can define agent personas..."*

So you have two valid locations: `agents/AGENT-NAME.md` in **`.github`** or in **`.github-private`** - but where you put them matters depending on whether you want org-scoped or enterprise-scoped sharing.

### Agent file scoping reference

| Level | Location | Scope |
| --- | --- | --- |
| User | `~/.copilot/agents/` | All projects for that user |
| Repository | `.github/agents/AGENT-NAME.md` | Current repo only |
| Organization | `agents/AGENT-NAME.md` in the org's `.github` **or** `.github-private` | All repos in that org |
| **Enterprise** | `agents/AGENT-NAME.md` in **`.github-private`** of the org pointed to by enterprise config | **All orgs in the enterprise** |

### Naming conflicts and precedence

Custom agents are resolved by **filename**. If the same agent filename exists at multiple scopes, the more specific scope wins:

```text
user-level > repository-level > organization-level > enterprise-level
```
{: .nolineno }

For example, if all four scopes define `test-writer.md`, Copilot uses the user-level `test-writer.md`. If there is no user-level copy, the repository-level one wins, then organization-level, then enterprise-level.

This is about duplicate **agent filenames**, not general instruction collisions. If different instruction sources all provide competing guidance, that can still be messy, but it is not the same deterministic custom-agent precedence rule.

> **Context note:** Making an org or enterprise agent available does not mean the full contents of every shared agent are injected into every prompt. In practice, shared agents show up as available agent definitions/metadata, and the selected or invoked agent controls the behavior.
{: .prompt-info }

### Org-scoped vs enterprise-scoped - confirmed by testing

Both `.github` and `.github-private` work for org-scoped agents (agents visible inside the org that owns the repo). I confirmed this on an EMU org: agents defined in both repos showed up in the same org.

For **enterprise-scoped agents** (where an enterprise admin points the enterprise Copilot config at a single org, and agents propagate to *other* orgs in the enterprise), the enterprise docs and UI point to `.github-private` as the configuration source:

> *"Configuration is sourced from `{org}/.github-private`."*

![Enterprise Copilot Agents settings showing the configuration sourced from the org's .github-private repository](enterprise-copilot-config-light.png){: .shadow }{: .light }
![Enterprise Copilot Agents settings showing the configuration sourced from the org's .github-private repository](enterprise-copilot-config-dark.png){: .shadow }{: .dark }
_The enterprise Copilot Agents settings page shows `.github-private` as the configuration source_

I tested this end-to-end: with an enterprise config pointing at an org that had agents in both `.github` and `.github-private`, only the `.github-private` agents (`org-standards` and `test-writer` in my case) showed up when viewing from a different org in the enterprise. Agents in `.github` of the same source org did *not* propagate enterprise-wide - they remained org-scoped.

> **Enterprise-scoped agents:** If you want an agent to be visible to *other orgs* in your enterprise via the enterprise Copilot configuration, use the designated `.github-private` repository. Agents in `.github` still work for org-scoped sharing.
{: .prompt-warning }

This also lines up with the docs. The official [prepare for custom agents](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/prepare-for-custom-agents) and [enterprise custom agents](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-agents/prepare-for-custom-agents) pages now document both `.github` and `.github-private` for org-scoped agents after a [recent docs clarification](https://github.com/github/docs/commit/21a6a142d04455daaba3ce419f3f5ee80c30ee82). For enterprise propagation, the documented enterprise configuration source is still `.github-private`.

### Visibility requirements

Visibility depends on which repo you use:

| Repo | Visibility behavior |
| --- | --- |
| `.github` | Use the visibility supported for `.github` on your platform |
| `.github-private` | **Internal** is recommended because it automatically grants read access to organization or enterprise members; **private** also works |

For enterprise-scoped agents, use the designated `.github-private` repository from the enterprise configuration.

## Feature 6: Copilot plugin marketplace and enterprise standards

[Enterprise-level Copilot plugin standards](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-agents/configure-enterprise-plugin-standards) are specifically about **Copilot Plugins** today. A plugin can package things like agents, skills, hooks, prompts, MCP servers, and related metadata so users can install and update them consistently.

You do **not** need a marketplace to use Copilot Plugins. A plugin can be installed directly from a repo or other supported source. The enterprise plugin standards configuration is more about governance: giving users a curated source, improving discoverability, and making versioning/updates more manageable across clients.

The enterprise docs use `.github-private`, but the same Copilot settings can also work from `.github`. In practice, `.github-private` is the placement you are more likely to use for enterprise-wide configuration because it is member-only by default.

### Setup

- Repo: **`.github`** or **`.github-private`**
- Path: `.github/copilot/settings.json`

### Visibility requirements

Visibility depends on which repo you use:

| Repo | Visibility behavior |
| --- | --- |
| `.github` | Use the visibility supported for `.github` on your platform |
| `.github-private` | **Internal** is recommended for enterprise-wide use; **private** also works |

## What about `copilot-instructions.md`, `AGENTS.md`, skills, hooks, and prompts?

A common question - do any of these per-repo Copilot files work as **org-wide defaults** from `.github` or `.github-private`? Short answer: **no**. Org-level Copilot instructions do exist, but you define them in organization settings, not in either of these repositories.

| File / Folder | Scope | Notes |
| --- | --- | --- |
| `.github/copilot-instructions.md` | Per-repo | Lives in each individual repo |
| `.github/instructions/**/*.instructions.md` | Per-repo (path-specific) | Modular instructions in each repo |
| `~/.copilot/copilot-instructions.md` | Personal/global | User's own machine |
| `AGENTS.md` | Per-repo | Git root or cwd - repo-scoped instructions |
| `.github/skills/` | Per-repo | Copilot skill definitions in the current repo |
| `.github/hooks/` | Per-repo | Copilot hook definitions in the current repo |
| `.github/prompts/` | Per-repo | Reusable prompt files in the current repo |
| `.github/workflows/copilot-setup-steps.yml` | Per-repo | Cloud agent environment setup |
| `.github/mcp.json` | Per-repo | MCP server configuration |

You can store skills, hooks, or prompts in `.github-private` if you want a central repo for source control, but that does **not** make them automatically available to Copilot coding agent across every repo. For CCA, install what you need in the target repo or during setup (for example, with `.github/workflows/copilot-setup-steps.yml`). For supported desktop/CLI clients, enterprise plugin standards can help push/install plugins so users get the right skills or tools locally, but that is a plugin/client policy flow - not an automatic `.github-private` repo lookup by CCA.

For **[org-level Copilot instructions](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-organization-instructions)**, use the Org Settings UI → Copilot → Custom Instructions. There is no file-based equivalent in `.github` or `.github-private`. The docs currently describe org custom instructions as supported for Copilot Chat on GitHub.com, Copilot code review on GitHub.com, and Copilot coding agent on GitHub.com. In practice, we've also seen VS Code pick up org instructions for repos in that org, even though the docs do not clearly say that yet - so treat client support as something to test rather than assume.

> **Org custom instructions gotcha:**
> - They apply based on the organization that owns the repository you are working in, not necessarily the organization or enterprise that assigned your Copilot license.
> - They are not file-based instructions, so they may not appear in logs or UI that are specifically showing **instruction file discovery** (`copilot-instructions.md`, `.instructions.md`, `AGENTS.md`, etc.).
> - If you are testing whether org instructions apply, verify behavior directly or inspect the final assembled context/debug output when available; don't rely only on file-discovery logs.
{: .prompt-warning }

In contrast, the Copilot features that *do* use `.github` or `.github-private` directly are [custom agents](#feature-5-copilot-custom-agents-org--enterprise-scope) and [Copilot plugin marketplace / enterprise standards](#feature-6-copilot-plugin-marketplace-and-enterprise-standards).

## EMU / GHEC-DR limitations

As of this writing, if you're on EMU or GHEC with data residency (GHE.com), these features are **not available** because they require a public repo:

- Public org profile README
- Org-wide issue templates
- Org-wide PR templates
- Org-wide discussion templates

One additional workflow template behavior to be aware of:

- **Workflow templates need the `.github` repo to be internal** - on EMU/GHEC-DR, a private `.github` repo did not serve workflow templates, even to private repos. This differs from the documented cascade rules, which suggest private should work for private repos.

Everything else (community health files like `CONTRIBUTING.md`/`SECURITY.md`/`SUPPORT.md`, workflow templates from an internal `.github`, Copilot custom agents, member-only README) still works.

## Summary

The `.github` and `.github-private` repos are used by several org-wide GitHub features, but the required repo, path, and visibility depend on the feature. The two practical takeaways:

1. **`.github` vs `.github-private` is mostly a clean split:** community health files, public README, and workflow templates live in `.github`. Member-only README lives in `.github-private`. Copilot **custom agents** and plugin standards can use either repo, but enterprise-wide configurations commonly use `.github-private`.
2. **Visibility rules are feature-specific:** issue/PR templates still require **public**, the member-only README requires **private**, and Copilot features can use internal visibility when hosted from `.github-private`.

If you're on EMU, the public-only requirements for issue/PR templates and the public org README are current product limitations. For issue and PR templates, the workaround is to manage them per-repo.
