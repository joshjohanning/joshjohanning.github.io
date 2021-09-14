---
title: 'Azure DevOps: Delete Custom Fields on Process Template'
author: Josh Johanning
date: 2021-08-10 22:30:00 -0600
description: Delete custom fields on a process template in Azure DevOps using the REST API
categories: [Azure DevOps, Work Items]
tags: [Azure DevOps, Work Items]
---

## Overview

I had the annoying misfortune today of running into an issue in Azure DevOps when customizing a process template. I added a field to a work item but I created the field as the wrong type. Once the custom field is created, there is not way to delete the field through the UI. Even with the REST API, it was a little tricky to find.

## Deleting the Field

First: You'll need to be a Project Collection Administrator in order to run this API.

This is a [link to the API](https://docs.microsoft.com/en-us/rest/api/azure/devops/wit/fields/delete?view=azure-devops-rest-6.0) we are going to use.

```
DELETE https://dev.azure.com/{organization}/{project}/_apis/wit/fields/{fieldNameOrRefName}?api-version=6.0

```

In Postman, let's create a new request with that URL string. I would have thought we would have passed in the Work Item `Process Template Name` instead of the `Project`, but I suppose it knows based on the project what process template it is using. 

Here is an example for my Azure DevOps organization / team project deleting the `NewTestField`.

```
https://dev.azure.com/jjohanning0798/TestTeamProject/_apis/wit/fields/NewTestField?api-version=6.0
```


Additionally, we should create a [Personal Access Token (PAT)](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page) with full permissions.

Under the Postman authentication's tab, we can leave the username blank and enter the PAT for the password. Use Basic Authentication.
![Postman authentication](/assets/screenshots/2021-08-10-azdo-delete-custom-field/postman-auth.png){: .shadow }
_Setting up Basic authorization in Postman with a PAT_

Send the request.

If it was successful, you will see a `204 No Content` message near the
![Postman authentication](/assets/screenshots/2021-08-10-azdo-delete-custom-field/postman-response.png){: .shadow }
_Successful request in Postman_

The field should no longer appear on our work item, and we can re-create the field with the right name and type. 

If you don't have the proper permissions (ie: Project Collection Administrator), you'll receive the following message:

```
"message": "Access Denied: 08dd71ec-5369-619d-bc32-495207cd99b7 needs the following permission(s) on the resource Delete field from organization to perform this action: Delete field from organization",
```

Enjoy! 
