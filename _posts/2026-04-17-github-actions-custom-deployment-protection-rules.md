---
title: 'Enforcing Deployment Promotion with Custom Deployment Protection Rules'
author: Josh Johanning
date: 2026-04-17 15:00:00 -0500
description: Using a Custom Deployment Protection Rule (GitHub App) to enforce environment promotion ordering and ServiceNow change ticket validation - works on any workflow, across any repo, at scale
categories: [GitHub, Actions]
tags: [GitHub, GitHub Actions, GitHub Apps, Deployments]
media_subpath: /assets/screenshots/2026-04-17-github-actions-custom-deployment-protection-rules
image:
  path: deployment-gate-light.png
  width: 100%
  height: 100%
  alt: A deployment protection rule rejecting a production deployment because the artifact was not deployed to a prior environment
---

## Overview

One of the most common requests I hear from enterprise customers is: *"How do I enforce that an artifact goes through Dev → QA → Staging → Production in order?"*

The challenge gets harder when you add constraints:
- **1,500+ repos** where every team writes their own CI/CD workflows
- No plans to build centralized reusable workflows for everyone
- Need to validate a **ServiceNow change ticket** before production
- Deploy to both **on-prem (OpenShift)** and **cloud (Azure)** — so OIDC alone doesn't solve it
- Need **segregation of duties** — a separate team approves production deployments

GitHub [Environments](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/using-environments-for-deployment) with protection rules handle some of this, but the built-in options (required reviewers, wait timers, branch restrictions) can't enforce *"was this SHA deployed to QA before Staging?"* That requires custom logic.

Enter [Custom Deployment Protection Rules](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/creating-custom-deployment-protection-rules) — a GitHub App that automatically intercepts any deployment to a protected environment and applies your custom validation logic. Teams don't need to change their workflows at all. The gate fires at the **environment level**, not the workflow level.

In this post, I'll walk through how to build a Custom Deployment Protection Rule that enforces environment promotion ordering and ServiceNow change ticket validation, and how to roll it out across hundreds of repos.

> Custom Deployment Protection Rules are available in public repositories for all plans. For private/internal repositories, you need GitHub Enterprise.
{: .prompt-info }

## How It Works

The flow is straightforward:

1. A team's workflow targets an environment (e.g., `environment: Production-East`)
2. GitHub sees the Custom Deployment Protection Rule attached to that environment
3. GitHub sends a `deployment_protection_rule` webhook to your app
4. Your app runs custom checks (prior environment deployed? change ticket valid?)
5. Your app responds with `approved` or `rejected`
6. GitHub allows or blocks the deployment accordingly

The key insight is that **any** workflow that uses `environment:` triggers the gate automatically. Teams don't add any enforcement logic to their workflows — it's all handled at the environment level.

```
Team writes ANY workflow → deploys to "Production" environment
                                      ↓
              GitHub automatically calls your gate app
                                      ↓
              Gate checks: ✅ Prior env deployed? ✅ Change ticket valid?
                                      ↓
                          Approve or Reject deployment
```

## What the Gate Checks

The demo gate app ([`deployment-gate-demo`](https://github.com/joshjohanning-org/deployment-gate-demo)) performs two checks:

### 1. Prior Environment Deployment (SHA-Based)

The gate queries the [GitHub Deployments API](https://docs.github.com/en/rest/deployments/deployments) to verify that the **exact SHA** being deployed has a successful deployment in the required prior environment:

{% raw %}
```
GET /repos/{owner}/{repo}/deployments?environment={prior_env}&sha={current_sha}
```
{: .nolineno}
{% endraw %}

This is strict — if SHA `abc1234` was deployed to Dev last week, and you're now deploying SHA `def5678` to QA, the gate rejects. The same SHA must go through each environment in order.

> A **failed** deployment doesn't count. If a deployment to QA exists but has a `failure` status, the gate will reject promotion to Staging with: *"SHA `abc1234` has deployments to 'QA' but none with a success status."*
{: .prompt-warning }

### 2. ServiceNow Change Ticket Validation

For production environments, the gate reads the change ticket from the workflow run's display title (via {% raw %}`run-name:`{% endraw %}) and validates it against ServiceNow's API. If no ticket is provided, or the ticket isn't approved, the deployment is rejected.

## Setting Up the Gate App

### Prerequisites

- A GitHub organization (Enterprise Cloud for private repos)
- Node.js 20+
- A [smee.io](https://smee.io) URL for local development (or a hosted endpoint for production)

### 1. Create the GitHub App

The gate app uses the [GitHub App manifest flow](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/creating-a-github-app-from-a-manifest) for easy setup. Clone the repo and run the setup script:

```bash
git clone https://github.com/joshjohanning-org/deployment-gate-demo.git
cd deployment-gate-demo
./setup.sh
```
{: .nolineno}

This opens your browser to create the GitHub App with the right permissions:

| Permission | Level | Purpose |
|---|---|---|
| Actions | Read-only | Read workflow run inputs |
| Deployments | Read and write | Query deployments + respond to protection rules |
| Metadata | Read-only | Required for all GitHub Apps |

And subscribes to the `deployment_protection_rule` event.

### 2. Configure the Environment Hierarchy

The environment hierarchy is defined in `config.yml`{: .filepath}:

```yaml
environments:
  Dev:
    order: 1
    requires_prior: null
    requires_change_ticket: false

  QA:
    order: 2
    requires_prior: Dev
    requires_change_ticket: false

  Staging:
    order: 3
    requires_prior: QA
    requires_change_ticket: false

  Production-East:
    order: 4
    requires_prior: Staging
    requires_change_ticket: true

  Production-Central:
    order: 4
    requires_prior: Staging
    requires_change_ticket: true
```
{: file='config.yml'}

If an environment name isn't in the config, the gate auto-approves. This means you can roll it out incrementally — only environments you explicitly configure are gated.

### 3. Run the App

```bash
npm install
npm start
```
{: .nolineno}

The app starts, connects to smee.io, and waits for webhook events:

```
╔══════════════════════════════════════════════════════════╗
║       Deployment Gate - Custom Protection Rule          ║
╚══════════════════════════════════════════════════════════╝

  Server:      http://localhost:3000
  Webhook:     http://localhost:3000/webhook
  Mock SNOW:   http://localhost:3000/api/servicenow/ticket/:number
  App ID:      3381065

  Smee proxy:  https://smee.io/your-channel
               → http://localhost:3000/webhook

  Environment hierarchy:
    1. Dev (no prior required)
    2. QA (requires Dev)
    3. Staging (requires QA)
    4. Production-East (requires Staging + change ticket)
    4. Production-Central (requires Staging + change ticket)

  Waiting for webhook events...
```
{: .nolineno}

### 4. Enable the Gate on Environments

For each environment you want to protect, add the Custom Deployment Protection Rule:

1. Go to the repo → **Settings** → **Environments** → select the environment
2. Under **Deployment protection rules**, enable your gate app

Or use the API to do it programmatically:

{% raw %}
```bash
gh api -X POST \
  "repos/{owner}/{repo}/environments/{env}/deployment_protection_rules" \
  -F integration_id=YOUR_APP_ID
```
{% endraw %}
{: .nolineno}

## The Sample Team Workflow

The sample team app ([`deployment-gate-app-demo`](https://github.com/joshjohanning-org/deployment-gate-app-demo)) shows what a typical team's workflow looks like. The important thing to notice is what's **not** in the workflow — there's zero enforcement logic:

{% raw %}
```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      release_tag:
        description: "Release tag to deploy"
        required: true
        type: string
      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - Dev
          - QA
          - Staging
          - Production-East
          - Production-Central
      change_ticket:
        description: "ServiceNow change ticket (required for Production)"
        required: false
        type: string

run-name: "Deploy ${{ inputs.release_tag }} to ${{ inputs.environment }} ${{ inputs.change_ticket }}"

jobs:
  deploy:
    name: "Deploy ${{ inputs.release_tag }} to ${{ inputs.environment }}"
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}  # This is what triggers the gate!
    steps:
      - name: Download release artifact
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh release download "${{ inputs.release_tag }}" \
            --repo "${{ github.repository }}" \
            --pattern "app-build.zip" --dir .

      - name: Deploy
        run: |
          echo "Deploying ${{ inputs.release_tag }} to ${{ inputs.environment }}..."
          unzip -q app-build.zip
          # Your actual deployment commands here
```
{% endraw %}
{: file='.github/workflows/deploy.yml'}

The team picks a release tag, an environment, and optionally a change ticket. The Custom Deployment Protection Rule fires automatically on the environment — the team didn't add any gate logic.

## Gate in Action

### Approval: QA After Dev

When deploying to QA with a prior successful Dev deployment:

```
════════════════════════════════════════════════════════════
Deployment Protection Rule - Request Received
════════════════════════════════════════════════════════════
  Repository:  joshjohanning-org/deployment-gate-app-demo
  Environment: QA
  SHA:         a92a500

  Check 1: Prior environment 'Dev' deployment...
    PASS: Found successful deployment of SHA `a92a500` to 'Dev'.
  Check 2: Change ticket not required — PASS

  Final decision: APPROVED
════════════════════════════════════════════════════════════
```
{: .nolineno}

### Rejection: QA Without Dev

When trying to deploy a new SHA directly to QA without deploying to Dev first:

```
════════════════════════════════════════════════════════════
Deployment Protection Rule - Request Received
════════════════════════════════════════════════════════════
  Repository:  joshjohanning-org/deployment-gate-app-demo
  Environment: QA
  SHA:         a92a500

  Check 1: Prior environment 'Dev' deployment...
    FAIL: No deployment of SHA `a92a500` found in environment 'Dev'.
          This exact commit/ref must be deployed to 'Dev' before
          it can be promoted.
  Check 2: Change ticket not required — PASS

  Final decision: REJECTED
════════════════════════════════════════════════════════════
```
{: .nolineno}

### Rejection: Production Without Change Ticket

```
════════════════════════════════════════════════════════════
Deployment Protection Rule - Request Received
════════════════════════════════════════════════════════════
  Repository:  joshjohanning-org/deployment-gate-app-demo
  Environment: Production-East
  SHA:         33d9c14

  Check 1: Prior environment 'Staging' deployment...
    FAIL: No deployments found for environment 'Staging'.
  Check 2: ServiceNow change ticket validation...
    FAIL: No ServiceNow change ticket provided.

  Final decision: REJECTED
════════════════════════════════════════════════════════════
```
{: .nolineno}

## Rolling Out at Scale

The repo includes a rollout script that programmatically creates environments and attaches the gate across repos:

```bash
# Dry run first
./scripts/rollout.sh --dry-run repo-1 repo-2 repo-3

# From a file (one repo name per line)
./scripts/rollout.sh --file priority-repos.txt

# All repos in the org
./scripts/rollout.sh --all
```
{: .nolineno}

The script creates environments per repo and attaches the gate:

| Environment | Gate | Branch Restriction |
|---|---|---|
| Dev | No | None |
| QA | Yes (requires Dev) | None |
| Staging | Yes (requires QA) | None |
| Production-East | Yes (requires Staging + ticket) | `main` + `v*` tags |
| Production-Central | Yes (requires Staging + ticket) | `main` + `v*` tags |

### Phased Rollout Strategy

1. **Host the gate app** — Azure Web App, VM, container, or any platform that can receive HTTPS webhooks
2. **Install the GitHub App org-wide** — it only activates on environments with the protection rule attached
3. **Start with high-priority repos** — run the rollout script on your most critical repos first
4. **Expand gradually** — add more repos over time, eventually running `--all`
5. **Communicate** — teams don't need to change anything as long as their deploy jobs use `environment:`

## Why This Scales

| Traditional Approach | Custom Deployment Protection Rules |
|---|---|
| Build reusable workflows for every team | Teams use ANY workflow |
| Hope teams include the right checks | Gate fires automatically on environment |
| Enforce via code review of workflows | Enforce via environment protection rules |
| Update 1,500 repos when rules change | Update ONE app when rules change |

## What If Teams Bypass Environments?

If a team's workflow doesn't use `environment:` at all, the gate won't fire. This is a gap, but there are mitigations:

- **Audit automation** — periodically scan repos for workflows that deploy without using environments
- **Runner-level enforcement** — if production deployment requires a specific runner group, and that runner group requires an environment, teams can't skip it
- **OIDC restrictions** — cloud credentials (Azure, AWS) only issued to specific environments
- **Organizational policy** — make it a requirement that deployment workflows use environments

## Interaction with Required Reviewers

If you add both a Custom Deployment Protection Rule **and** required reviewers to an environment, they run **in parallel**. Both must pass before the deployment proceeds. The reviewer can see the gate's approval/rejection comment while making their decision.

## Demo Repos

All the code is available in these repos:

| Repo | Purpose |
|---|---|
| [`deployment-gate-demo`](https://github.com/joshjohanning-org/deployment-gate-demo) | The Custom Deployment Protection Rule app (Express.js + smee.io) |
| [`deployment-gate-app-demo`](https://github.com/joshjohanning-org/deployment-gate-app-demo) | Sample team app with CI/CD workflows |

## Summary

Custom Deployment Protection Rules solve the *"enforce governance without owning the workflows"* problem. You build one gate app, attach it to environments across your org, and every deployment is automatically validated — regardless of which team wrote the workflow, which language they use, or where they deploy to. The gate checks the native GitHub Deployments API (which Actions creates automatically when a job uses `environment:`), so there's zero overhead for development teams.

For organizations with hundreds or thousands of repos where centralized reusable workflows aren't feasible, this is probably the most practical enforcement mechanism available today.
