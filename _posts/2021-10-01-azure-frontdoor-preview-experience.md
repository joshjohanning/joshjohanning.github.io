---
title: 'Azure Front Door Standard/Premium Tips and Tricks'
author: Josh Johanning
date: 2021-10-01 16:30:00 -0500
description: I share my experience, lessons-learned, and tips and tricks for working with the new Azure Front Door Standard/Premium SKUs
categories: [Azure, Front Door]
tags: [Azure, Azure Front Door]
image:
  path: /assets/screenshots/2021-10-01-azure-frontdoor-preview-experience/front-door-overview-expanded.png
  width: 500   # in pixels
  height: 530   # in pixels
  alt: Front Door Overview
---

## Update

Since writing this article, Azure Front Door Standard/Premium has been released to GA (on March 29, 2022). You can read more about it here: [Introducing the new Azure Front Door: Reimagined for modern apps and content](https://azure.microsoft.com/en-us/blog/introducing-the-new-azure-front-door-reimagined-for-modern-apps-and-content/).

Likewise, [Azure Front Door Terraform resources](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_endpoint) have since become available to help manage:

- [**AzureRM v3.25.0**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.25.0) (Sept 29, 2022): Additions of [azurerm_cdn_frontdoor_route](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_route), [azurerm_cdn_frontdoor_custom_domain](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_custom_domain), and[azurerm_cdn_frontdoor_route_disable_link_to_default_domain](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_route_disable_link_to_default_domain)
- [**AzureRM v3.21.0**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.21.0) (Sept 2, 2022): Additions of [azurerm_cdn_frontdoor_rule](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_rule) and [azurerm_cdn_frontdoor_secret](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_secret)
- [**AzureRM v3.19.9**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.19.0) (Aug 18, 2022): Additions of [azurerm_cdn_frontdoor_firewall_policy](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_firewall_policy) and [azurerm_cdn_frontdoor_security_policy](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_security_policy)
- [**AzureRM v3.15.0**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.15.0) (July 21, 2022): Additions of [azurerm_cdn_frontdoor_origin](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_origin) and [azurerm_cdn_frontdoor_origin_group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_origin_group)
- [**AzureRM v3.10.0**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.10.0) (June 10, 2022): Addition of [azurerm_cdn_frontdoor_rule_set](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_rule_set)
- [**AzureRM v3.9.0**](https://github.com/hashicorp/terraform-provider-azurerm/releases/tag/v3.9.0) (June 2, 2022): Additions of [cdn_frontdoor_profile](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_profile) (with options of `Standard_AzureFrontDoor` or `Premium_AzureFrontDoor`) and [azurerm_cdn_frontdoor_endpoint](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cdn_frontdoor_endpoint)

The bulk of this article should still be relevant, but if anything is strikingly wrong or has changed, leave a comment to let me know!

## Overview

I want to talk about Azure Front Door - not the old Azure Front Door - the new Azure Front Door, the new [PREVIEW Front Door with the Standard/Premium SKUs](https://docs.microsoft.com/en-us/azure/frontdoor/standard-premium/overview). But Josh, wait, this is a DevOps blog, why are you talking about Azure Front Door? Well, I had the ~~pleasure~~ experience of working with Front Door (Preview) on my most recent project, and thought I would be doing the world a disservice by not sharing a little bit of the ~~frustration~~ knowledge I have gained while working with it. Whether no one else is using this service or no one is talking about it, we struggled to find many resources online for how to do certain things in the Front Door (Preview), so this is where this article comes into play.

I am not planning on writing an entire how-to article, this is just intended to serve as a resource that hopefully the SEO Gods can help make someone else's life easier. If you are unfamiliar with Azure Front Door (Preview), or want some official background and guidance from Microsoft, see the [What is Azure Front Door Standard/Premium (Preview)?](https://docs.microsoft.com/en-us/azure/frontdoor/standard-premium/overview) page. Also, this is a [great resource](https://www.kenmuse.com/blog/comparing-azure-front-door-to-other-services/) from Ken Muse on when you should use Azure Front Door vs. Application Gateway, Load Balancer, or Traffic Manager.

Note: For the purposes of this article, I am going to abbreviate Azure Front Door as AFD, and when I say AFD, I mean the new Preview Azure Front Door; I will not be referring to the classic / old Front door here.

(PS: Guinness Book of Records, see above for my submission on most times "Front Door" has been used in a single sentence)

## Things That Work Well

- Private Endpoints
    - Azure Front Door does a great job of routing to services such as App Services, Function Apps, Storage Accounts, and Private Link Services that are protected via Private Endpoints
    - Since these are 'magic' aka managed Private Endpoints, the Private Endpoint doesn't live in your subscription and you don't have access to it. Therefore, there doesn't seem to be a way to get the Origin Group / Origin deployment to automatically approve these, so you have to remember to go to the target resource and approve the Private Endpoint manually. This is similar to how Azure Data Factory's Private Endpoints work
    - Private Endpoints only work with the Premium SKU
- Certificates!
    - Azure Front Door does a great job of automatically managing certificates - including expirations - the default setting is to let Azure Front Door handle all of this for you with 0 configuration
    - You still have the option to bring your own certificate by creating an Azure Front Door Secret linked to a Certificate in an Azure Key Vault - Azure Front Door even shows expiration of that certificate on the Domain page
    - If you are using your own certificate, it needs to be in PFX / PKCS 12 format (not PEM)
- Web Application Firewall (WAF)
    - Anecdotally, I only have experience with the Premium SKU of the Azure Front Door Preview service, but [creating a WAF tied to Microsoft-provided default rule set](https://docs.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-create-portal?toc=/azure/frontdoor/standard-premium/toc.json#default-rule-set-drs) is relatively simple
    - The Standard SKU does not let you use a WAF
- Letting teams share a single Front Door resource
    - Teams can manage their own Endpoints in a single Azure Front Door resource without worry of mucking it up too much for other teams
    - This splits the [current $165/monthly cost](https://azure.microsoft.com/en-us/pricing/details/frontdoor/) for the Premium SKU
- Linking Origin to just about any hostname
    - Works great for serving static websites hosted in an Azure Storage Account - we created a static website container `$web` and set our Origin to `Origin Type: Custom` and `Host Name: mystaticsite.z21.web.core.windows.net`
    - We got the idea from [article as a basis for this static website setup](https://web.archive.org/web/20230329203146/https://docs.rackspace.com/blog/azure-front-door-storage-static-website/) (okay this is referencing the Classic Azure Front Door, but the custom host name [screenshot](https://stackoverflow.com/a/74105613/4270353) was what did the trick)
    - If using Private Endpoints, you still have to have Azure Front Door create a private endpoint to the storage account. We set the 'target sub resource' (aka `groupId` in the ARM/API) to `web`.
    - We did something similar with our Kubernetes linkage, creating a Private Link Service bound to the AKS managed subnet and Internal Load Balancer of our nginx service. In this pattern, the `Origin Type: Custom` and `Host Name` was set to the IP of the Internal Load Balancer and we used `null()` for our `Origin Host Header` and let nginx do header routing based on the URL that is being sent from Front Door to the Cluster. We set the 'target sub resource' (aka `groupId` in the ARM/API) to `null()`. See below [example](#arm-null-property-and-terraform)
     - Works as expected with App Services and Function Apps; just set `Origin Type: App Services`

## Things That Don't Work Well

- Private Endpoints
    - For about 3-4 weeks in July/early August, Azure Front Door's Private Endpoints just completely died and Microsoft support was super slow in getting any attention to this. I get that it's a Preview feature and things happen, but the resolution time was a little disappointing. We haven't had any issues since they "reverted" the change that broke this, though
    - You have to manually approve the Private Endpoint on your target resource
- URL Rewriting
    - We were trying to use a single Azure Front Door Endpoint to host all of our App Service APIs as Origins; sort of like a poor man's APIM (around 18x cheaper if you're not using all of the premium features of a Premium APIM)
    - We wanted `myfd.z01.azurefd.net/foo` to redirect to `foo.azurewebsites.net`
    - Maybe we were just doing it wrong, but we struggled to do native URL Rewriting in Azure Front Door - it's incredibly ~~possible~~ likely that we are just misinterpreting how it's supposed to work
    - See my [Stack Overflow](https://stackoverflow.com/questions/68564910/url-rewrite-in-azure-front-door-preview-standard-premium) post of what we were trying to do, and someone's [suggestion](https://stackoverflow.com/a/68914412/4270353) on how to resolve (I have not had a chance to test yet)
    - We instead did [URL Rewriting](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/url-rewriting?view=aspnetcore-5.0) by using middleware at the app level. It just requires a making a [small modification in the startup.cs file](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/url-rewriting?view=aspnetcore-5.0#url-redirect) (and if you're serving a Swagger page, there too!). See: [examples below](#url-rewrite)
    - For Function Apps, there is no way to rewrite the incoming URL. However, you can edit the [host.json file and customize the base path](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-output#hostjson-settings) by modifying the `routePrefix` property. See: [example below](#function-apps---hostjson)
- Update Times
    - Okay minor gripe, but it takes anywhere from [5-20 minutes](https://docs.microsoft.com/en-us/azure/frontdoor/standard-premium/faq#how-long-does-it-take-to-deploy-an-azure-front-door-does-my-front-door-still-work-when-being-updated) for a change you make to Front Door to propagate down to you
    - Sometimes loading in a different browser / using a proxy can help alleviate cache/dns issues
- [No native Terraform Resource (yet)](https://github.com/hashicorp/terraform-provider-azurerm/issues/11983)
    - We are using the [azurerm_template_deployment](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/template_deployment) resource to deploy an ARM template within Terraform
- Editing via the UI
    - You have to click the 'Edit' button on the Endpoint, then click into the Origin Group/Route to make changes - if you just click on the Origin Group/Route without clicking 'Edit' first, you will just be in a read only mode.
- WebSockets
    - [Azure Front Door does not support WebSockets](https://github.com/MicrosoftDocs/architecture-center/issues/1891#issuecomment-918733057) - use Long Polling instead for SignalR use cases

## Random notes

- HTTPS Redirect in .NET Web Apps
    - It is considered best practice to redirect http to https. This can be done in the code by adding the following to the `startup.cs`{: .filepath} file: `app.UseHttpsRedirection();`
    - The problem with this method when using Azure Front Door with Private Endpoints is that this causes the app to redirect to the host (azurewebsites.net) instead of the incoming host URL (custom domain on Azure Front Door)
    - To work around this, you should remove this line of code altogether from the application and let Front Door redirect traffic to HTTPS (a setting on the Route)
    - If you're working with an Angular app, make sure to remove any HTTPS redirect from the `web.config`{: .filepath}
- Deleting Endpoints / Domains / Front Door
    - If you want to delete a domain, you will need to clean up all of the associations (i.e.: the Route)
    - You can delete the entire Endpoint the domain is associated to as well
    - If there is/was a WAF associated to that endpoint, though, you need to manually go into the WAF resource and remove the association to the domain manually. This isn't made clear by the UI error (`Failed to delete the custom domain(s)`) or the CLI error (`(BadRequest) Property 'AfdDomainEntityKey.AfdDomainName' cannot be set to 'mysubdomain.mydomain.com'.`)
    - If you go to re-create the Endpoint with the same name, note that it will fail for the first time with a `Conflict: That resource name isn't available`  error message!!! You simply have to attempt to create the endpoint another time and then it will go through properly. The error will look something like this: `error": {\r\n "code": "Conflict",\r\n "message": "That resource name isn't available."`
    - If you delete your entire Front Door, you still need to delete the WAF manually, and you will still see a `conflict` error message the first time you try to re-create a deleted endpoint
- `Our services aren’t available right now` error
    - Make sure you have approved the Private Endpoint on the target resource
    - Alternatively, go and edit the Origin Group > Origin, uncheck the Private Endpoint box, save the Origin, Save the Origin Group, wait 10-30 seconds for it to apply, edit the Origin Group > Origin, check the Private Endpoint box and select the right resource, save the Origin, save the Origin Group, and go and re-approve the Private Endpoint on the target resource
- `<h2>Our services aren't available right now</h2><p>We're working to restore all services as soon as possible.` error
    - Your endpoint is still provisioning
    - Or, your Route is misconfigured
    - The HTML will be not be rendered on the page for this error - if it does render it means Front Door is routing correctly it's likely a problem with a private endpoint (see above)
- `Page not found` blue page error
    - Wait 5-20 minutes for the Endpoint to provision
- Sometimes CORS errors disguise themselves as a misconfigured Endpoint / Private Endpoint
- Be familiar with [grabbing a Bearer token to interact directly with the Azure REST APIs](https://social.technet.microsoft.com/wiki/contents/articles/51140.azure-rest-management-api-the-quickest-way-to-get-your-bearer-token.aspx)
    - As an example, you can plug that Bearer token into Postman and use this `GET` request to list the details about a Origin in an Origin Group: 
    ```terminal
    GET https://management.azure.com/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/my-afd-rg/providers/Microsoft.Cdn/profiles/my-afd/originGroups/myorigingroup/origins?api-version=2020-09-01&Full
    ```
    - This can also be helpful when deciphering what changes / values you need to use in an ARM template


## Summary

Hopefully I haven't scared you away; Azure Front Door (Preview) has a lot of great features and shows a lot promise! And when it works, it works real well. Half of the frustration that we had with this service was that no one else had written about it, so we were kind of making it up as we go along. We have a solid foundation now, and once there is a native Terraform module, making changes for us will be even easier.

Happy hacking!

*See the [Appendix below](#appendix-examples) for miscellaneous [logging](#log-analytics--diagnostics-query), [URL rewriting](#url-rewrite), [AFD CLI](#afd-cli-commands), and [ARM template examples](#arm-null-property-and-terraform)*

## Appendix: Examples

### Log Analytics / Diagnostics Query

This assumes you have a [Diagnostics Settings created](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings?tabs=CMD) that captures `FrontDoorAccessLog` logs and sends to a Log Analytics resource.

See the below for an example Log Analytics / Diagnostics query to find non-200 HTTP Status Codes

```
AzureDiagnostics
 | where httpStatusCode_s != 200 
    and TimeGenerated > ago(500m)
```

### URL Rewrite

See the below sections for examples on how to do URL Rewriting in .NET (.NET Core) App Services and Function Apps

#### .NET - startup.cs

```csharp
using Microsoft.AspNetCore.Rewrite;  // add this using

public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory)
{
    // url rewrite
    var options = new RewriteOptions().AddRewrite(@"^myapi/(.*)", "$1", 
                  skipRemainingRules: true);
    app.UseRewriter(options);
    
    ... // remainder of code below
}
```

> Note: This makes the URL for local development something like `http://localhost:5001/myapi/...`

> Note: If you need `/path` and `/path/` to work, then the regex for the rewrite should be `@"^myapi[/]?(.*)"`

#### .NET - Swagger

If you are using Swagger for an API, you should also update the prefix for the Swagger endpoint:

```csharp

// Enable middleware to serve generated Swagger as a JSON endpoint.
app.UseSwagger();

app.UseSwaggerUI(c =>
{
   // Specify swagger JSON endpoint
   var prefix = apiPath == "" ? "" : $"/{apiPath}";
   c.SwaggerEndpoint($"{prefix}/swagger/{apiVersion}/swagger.json", apiDefinitionTitle);
   c.DocExpansion(DocExpansion.None);
   // specifying the Swagger-ui endpoint.
   c.RoutePrefix = "swagger-ui";
   c.DefaultModelsExpandDepth(-1);
});
```

#### Function Apps - host.json

`host.json`{: .filepath}:
```json
{
  "version": "2.0",  
  "extensions": {
    "http": {
      "routePrefix": "function1"
    }
  }
```
{: file='host.json'}

### AFD CLI Commands

#### Delete Origin Group

```terminal
az afd origin-group delete --profile-name my-afd --resource-group my-afd-rg --origin-group-name myorigingroup --yes
```

#### Delete Custom Domain

```terminal
az afd custom-domain delete --profile-name my-afd --resource-group my-afd-rg --custom-domain-name mysubdomain.mydomain.net
```

#### Delete Route

```terminal
az afd route delete --profile-name my-afd --resource-group my-afd-rg --endpoint-name my-endpoint --route default
```

#### Delete Endpoint

```terminal
az afd endpoint delete --profile-name my-afd --resource-group my-afd-rg --endpoint-name my-endpoint
```

#### Show Endpoint

```terminal
az afd endpoint show --profile-name my-afd --resource-group my-afd-rg --endpoint-name my-endpoint
```

#### Adding Custom Domain with Certificate

Note that when `--custom-domain-name` asks for a name, it's just the friendly name of the domain as it appears in AFD

```bash
az afd custom-domain create -my-afd-rg --custom-domain-name foobar --profile-my-afd --host-name '*.mysubdomain.mydomain.com' --minimum-tls-version TLS12 --certificate-type CustomerCertificate --secret my-wildcard-cert-pfx --debug
```

#### Purging Cache

```bash
az afd endpoint purge --ids "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/my-afd-rg/providers/Microsoft.Cdn/profiles/my-afd/afdendpoints/my-endpoint" --content-paths "/*"
```

### ARM Null() property and Terraform

We couldn't figure out a way to pass in `null()` from Terraform to our ARM template using the [azurerm_template_deployment](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/template_deployment) resource, and passing in `""` failed in mysterious ways or ended up just hanging the deployment. Essentially, we wrote some logic to convert the `""` passed into the template parameter to `null()` for us. 

Note lines 20, 25, and 69 for the relevant logic:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "profileName": {
            "type": "string"
        },
        "originGroupsName":{
            "type": "string"
        },
        "hostName":{
            "type": "string"
        },
        "hostHeader":{
            "type": "string"
        },
        "privateLinkResourceId":{
            "type": "string"
        },
        "privateLinkResourceType":{
            "type": "string"
        }
    },
    "variables": {
        "is-private-link-service-type": "[equals('', parameters('privateLinkResourceType'))]",
        "is-null-host-header": "[equals('', parameters('hostHeader'))]"
    },
    "resources": [
        {
            "type": "Microsoft.Cdn/profiles/originGroups",
            "apiVersion": "2020-09-01",
            "name": "[concat(parameters('profileName'), '/', parameters('originGroupsName'))]",
            "properties": {
                "loadBalancingSettings": {
                    "sampleSize": 4,
                    "successfulSamplesRequired": 3,
                    "additionalLatencyInMilliseconds": 50
                },
                "healthProbeSettings": {
                    "probePath": "/",
                    "probeRequestType": "HEAD",
                    "probeProtocol": "Http",
                    "probeIntervalInSeconds": 100
                },
                "sessionAffinityState": "Disabled"
            }
        },
        
        {
            "type": "Microsoft.Cdn/profiles/originGroups/origins",
            "apiVersion": "2020-09-01",
            "name": "[concat(parameters('profileName'), '/', parameters('originGroupsName'), '/default')]",
            "dependsOn": [
                "[resourceId('Microsoft.Cdn/profiles/originGroups', parameters('profileName'), parameters('originGroupsName'))]"
            ],
            "properties": {
                "hostName": "[parameters('hostName')]",
                "originHostHeader": "[if(variables('is-null-host-header'), null(), parameters('hostHeader'))]",
                "httpPort": 80,
                "httpsPort": 443,
                "priority": 1,
                "weight": 1000,
                "enabledState": "Enabled",
                "sharedPrivateLinkResource": {
                    "privateLinkLocation": "[resourceGroup().location]",
                    "privateLink": {
                        "id": "[parameters('privateLinkResourceId')]"
                    },
                    "groupId": "[if(variables('is-private-link-service-type'), null(), parameters('privateLinkResourceType'))]",
                    "requestMessage": "Private link service from AFD"
                }
            }
        }
    ]
}
```
