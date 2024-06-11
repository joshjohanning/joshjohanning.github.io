---
title: 'Integrate GitHub Enterprise Server with Slack'
author: Josh Johanning
date: 2023-12-18 19:30:00 -0600
description: Integrate GitHub Enterprise Server to receive notifications in Slack without opening up the firewall
categories: [GitHub, Integrations]
tags: [GitHub, Enterprise Server, Slack]
media_subpath: /assets/screenshots/2023-12-18-github-enterprise-server-slack
image:
  path: ghes-slack-integration.png
  width: 100%
  height: 100%
  alt: GitHub Enterprise Server integration with Slack
---

## Overview

I recently had a customer ask if it was possible to natively integrate GitHub Enterprise Server with Slack. Right now, they are using custom actions in GitHub Actions and scripts in Jenkins to push CI/CD notifications to Slack.

GitHub has a [Slack integration](https://github.com/integrations/slack), but the docs are slightly ambiguous to as whether it works with GitHub Enterprise Server without having to open up your network to allow inbound access from all of [Slack's URLs](https://github.slack.com/help/urls). The docs state:

> Bidirectional connectivity between GHES and Slack: Our GHES integration is not just a notification service. It will also enable you to perform actions directly from chat. **So, the only prerequisite you need to ensure your GHES instance is accessible from Slack.**

Slack has a [socket mode](https://api.slack.com/apis/connections/socket), which uses WebSockets initiated from the GitHub Enterprise Server instance to communicate with Slack. This is similar to how self-hosted runners work (well, slightly different, [self-hosted runners use long-polling](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#communication-between-self-hosted-runners-and-github)), which allow external communication without opening up inbound traffic in the firewall. 

But currently, the README has a seemingly ominous note regarding Slack socket mode:

> **Slack Socket mode**
>
> Proxies not currently supported.

I believe this to mean *Slack socket mode is supported*, but *if you're using a proxy, then it will not work*. 

Several other people have had similar [questions](https://github.com/integrations/slack/issues/1702) as to how and if this integration will work in GitHub with Slack socket mode enabled. The answer is yes, and I'll walk you through!


## Prerequisites

1. GitHub Enterprise Server 3.8+
2. Access to the GitHub Enterprise Server management console
3. Admin access in Slack to generate an **[App Configuration Token](https://api.slack.com/apps)** and **install an app into a workspace**
  - Note that you have to install the Slack app to the workspace *from a link provided in the management console*, but more on that in a bit
4. [Organization owner](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners) access to the GitHub org(s) that want to integrate with Slack to be able to install the GitHub Slack app

## Integration Steps

1. Navigate to the GitHub Enterprise Server management console, e.g. `https://github.example.com:8443/setup/settings`
2. Scroll down to the "Chat Integration" section and click "Enable GitHub Chat integration"; it should default to Slack, but if not, select the Slack radio button
  ![Enable GitHub Chat integration](ghes-slack-integration-step-02.png){: .shadow }
  _Enable GitHub Chat integration_
3. Navigate to the [Slack API portal](https://api.slack.com/apps)
4. Click on the "Generate Token" button - this will generate a token that GitHub will use to *create an app for you* under your account
  ![Generate Token](ghes-slack-integration-step-04.png){: .shadow }
  _Click the generate token button to allow GitHub to register an app for you_
5. Confirm the Slack workspace where the app will be created
  ![Selecting the Slack workspace for the app](ghes-slack-integration-step-05.png){: .shadow }
  _Selecting the Slack workspace for the app_
6. Click the "Copy" button in the "Access Token" column for the newly created token
  ![Copy the Access Token](ghes-slack-integration-step-06.png){: .shadow }
  _Copy the newly generated access token_
7. Paste in the access token, check the box for "Configure to use socket mode", and click "Generate App"
  ![Pasting in the access token, selecting socket mode, and generating the app](ghes-slack-integration-step-07.png){: .shadow }
  _Pasting in the access token, selecting socket mode, and generating the app_
8. After a few moments, you should a message saying "Slack app generated successfully"
  ![Slack app generated successfully](ghes-slack-integration-step-08.png){: .shadow }
  _Slack app generated successfully_
9. You will see a "Slack App ID" was generated - click the ID to follow the link to the Slack API portal
  ![Slack App ID](ghes-slack-integration-step-09.png){: .shadow }
  _A Slack APP ID was generated - follow the link_
10. Scroll down to "App-Level Tokens" and click "Generate Token and Scopes"
  ![Generate Token and Scopes](ghes-slack-integration-step-10.png){: .shadow }
  _Generate Token and Scopes_
11. Give the token a name and add the `connections:write` and `authorizations:read` scopes and click "Generate"
  ![Creating a token with connections and authorizations access](ghes-slack-integration-step-11-1.png){: .shadow }
  _Creating a token with `connections:write` and `authorizations:read` access_
12. Copy the token
  ![Copy the token](ghes-slack-integration-step-12-1.png){: .shadow }
  _Copy the token_
13. Paste the token in the "Slack app level token" field and click "Save"
  ![Paste the Slack app level token and save](ghes-slack-integration-step-13.png){: .shadow }
  _Paste the Slack app level token and click save_
14. You should see a message indicating the Slack app level token was saved successfully
  ![Slack app settings have been saved](ghes-slack-integration-step-14.png){: .shadow }
  _Slack app settings have been saved_
15. Click on the server's "Save Settings" button to save the changes - this shouldn't cause any downtime but it will take some time (15-30 minutes) for the changes to take effect
  ![Save settings](ghes-slack-integration-step-15.png){: .shadow }
  _Save settings and wait for the server to finish configuring_
16. Once the server is finished configuring, navigate back to the management console
17. Scroll down to the "Chat Integration" section again; you should see "Chat integration is now available to be installed in the workspace" - follow the link!
  - The URL: `https://github.example.com/_slack/`
  ![From the management console, install the Slack app](ghes-slack-integration-step-17.png){: .shadow }
  _From the management console, install the Slack app to the workspace_
18. Click on the "Add to Slack" button
  ![Add the Slack App](ghes-slack-integration-step-18.png){: .shadow }
  _Add the Slack app_
19. Install the app into the Slack workspace
  ![Install the Slack App](ghes-slack-integration-step-19.png){: .shadow }
  _Allow the Slack app to be installed to the Slack workspace_
20. After it's installed, it will redirect you back to Slack
  ![Redirected back to Slack](ghes-slack-integration-step-20.png){: .shadow }
  _After installing the app, you will be redirected back to Slack_
21. The redirect will take you to the `GHE` bot in Slack and will ask you to link your account to begin using
  ![Connect your GitHub account to Slack by interacting with the `GHE` bot](ghes-slack-integration-step-21.png){: .shadow }
  _Connect your GitHub account to Slack by interacting with the `GHE` bot_
22. Click the button to connect your GitHub account to Slack 
  ![Authorize your GitHub Account with Slack to generate a code and paste it back into Slack](ghes-slack-integration-step-22.png){: .shadow }
  _Authorize your GitHub Account with Slack by generating a code and pasting it back into Slack_
23. Copy the verification code that you will paste back into Slack
  ![Code for connecting your GitHub account to Slack](ghes-slack-integration-step-23.png){: .shadow }
  _Code for connecting your GitHub account to Slack_
24. Click the "Enter Token" button and paste in the code
  ![Pasting in the verification code to complete the authentication](ghes-slack-integration-step-24.png){: .shadow }
  _Pasting in the verification code to complete the authentication_
25. Success! Your GitHub account is now linked to Slack
  ![GitHub Enterprise user account is now connected to Slack](ghes-slack-integration-step-25.png){: .shadow }
  _GitHub Enterprise user account is now connected to Slack_
26. In a Slack channel, type and send `/ghe subscribe owner/repo` to subscribe to a repo's notifications - you'll notice that we are now prompted to install the Slack GitHub App to the repository now
  ![Subscribing to a repo to a Slack channel](ghes-slack-integration-step-26.png){: .shadow }
  _Subscribing to a repo to a Slack channel_
27. Select the organization to install the Slack GitHub App to
  ![Installing the Slack GitHub App to a GitHub organization](ghes-slack-integration-step-27.png){: .shadow }
  _Installing the Slack GitHub App to a GitHub organization_
28. Determine if you want to grant the Slack GitHub App access to all repositories or only select repositories
  ![Grant access to all repositories or select repositories for the Slack GitHub App](ghes-slack-integration-step-28.png){: .shadow }
  _Grant access to all repositories or select repositories for the Slack GitHub App_
29. Assuming you have permissions (organization owner), the app should now be installed
30. Head back to the Slack channel and send `/ghe subscribe owner/repo` again - you should now see a message indicating that the channel is now subscribed to the repository
  ![GitHub notifications in Slack](ghes-slack-integration-step-30.png){: .shadow }
  _GitHub notifications in Slack_
31. To only receive notifications for a [specific feature](https://github.com/integrations/slack/?tab=readme-ov-file#customize-your-notifications), use `/ghe subscribe owner/repo <feature>`
32. You can also DM the `GHE` bot to subscribe to repositories directly to your DMs, as well as using commands such as `/ghe open owner/repo` or `/ghe close [issue link]` to open or close issues

That's it! 🎉

## Summary

Now, we can integrate GitHub natively with Slack to be able to receive rich notifications just like if we were using GitHub.com or GitHub Enterprise Cloud. The GitHub Enterprise Server instance that I was using for this post had firewall rules that prevented the outside world from accessing my instance, and I was able to prove that you don't need to create any special firewall rules to be able to use the Slack integration. 

all possible because of [Socket Mode](https://api.slack.com/apis/connections/socket) in Slack, which creates a WebSocket connection initiated by your GitHub Enterprise Server instance to Slack's servers to pass information bidirectionally. This is much better than building your own Actions or configuring your own webhooks!

Enjoy the notifications! 📣 💬
