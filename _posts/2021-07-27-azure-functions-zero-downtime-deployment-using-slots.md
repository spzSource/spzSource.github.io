---
layout: post
title: "Zero-downtime deployment for Azure functions - how-to guide."
date: 2021-07-27 19:50:00 +0300
categories: devops
tags: devops, deployment, continuous, CD, azure, functions, slots, guide
---

I found it useful to highlight the approach of zero-downtime deployment in context of Azure Functions as well as a few "pain points", because the standard documentation does not explain all nuances (there are a lot of nuances by the way :smile:).

## Context

To achieve zero-downtime deployment we need to be able to run two versions of the same application (for instance v1 and v2) in parallel. All remaining in-flight requests to v1 must be handled correctly and traffic between v1 and v2 must be redirected seamlessly. This is what deployment slots provide for us "out of the box".

Just to mention, deployment slots also provide the ability for partial traffic redirection, so are suitable for A/B testing or canary releases as well, but it's out of the scope of this article.

:warning: Please keep in mind that for Consumtion plan it's possible to create only one slot per app, so in that case it's not possible to fully apply A/B testing practices.

## How to create deployment slot

Here is how it can be created using Azure Management Library (`Microsoft.Azure.Management`):

```cs
var function = await Azure.AppServices.FuntionApp
    .GetByResourceGroupAsync("group", "name");

var configuration = GetConfiguration(...);

configuration.Add("FUNCTION_EXTENSION_VERSION", "~3");
configuration.Add("WEBSITE_SWAP_WARMUP_PING_PATH", "/api/healthcheck");
configuration.Add("WEBSITE_SWAP_WARMUP_PING_STATUSES", "200");
configuration.Add("WEBSITE_ADD_SITENAME_BINDING_IN_APPHOST_CONFIG", "1");
configuration.Add("WEBSITE_RUN_FROM_PACKAGE", "1");

var slot = await function
    .DeploymentSlots
    .Define("staging")
    .WithBrandNewConfiguration()
    .WithSystemAssignedManagedServiceIdentity()
    .WithWebApAlwaysOn(false)
    .WithTags(new Dictionary<string, string> { 
        { "Name", "ServiceA" },
        { "Version", "0.9.1" }
    })
    .WithStickyAppSettings(configuration)
    .CreateAsync();
```

There are few things to mention:

1. Instead of using `WithBrandNewConfiguration` and setting up all configuration from scratch it's possible to inherit configuration from another slot (methods `WithConfigurationFromFunctionApp()` `WithConfigurationFromDeploymentSlot()`), but this causes unintended slot swap, bacause `WEBSITE_CONTENTSHARE` (this is where execution binaries are located) setting is not slot specific and copied over during swap, so we end up with two slots pointing to the same location. More details here: [link](https://github.com/MicrosoftDocs/azure-docs/issues/36458)
2. For slot we set `WithWebApAlwaysOn` to false. That will force the app in staging slot to be suspended eventually after swap. 
3. All app specific settings are defined as sticky (`WithStickyAppSettings`). Meaning that the configuration is slot specific.
4. Created slot does not inherit Identity (in cases MSI configured), so it's required to created a new identity for the slot (`WithSystemAssignedManagedServiceIdentity()` method)

Specific settings:
- `FUNCTION_EXTENSION_VERSION` must be set in case `WithBrandNewConfiguration` when defining the slot, because in that case slot doesn't inherit function runtime version.
- `WEBSITE_SWAP_WARMUP_PING_PATH` and `WEBSITE_SWAP_WARMUP_PING_STATUSES` are used for configuring introspection/health-check endpoint, which is used for warnup phase. In my particular case there is a custom health-check endpoint `/api/healtcheck` indicating healthiness of the service.
- `WEBSITE_ADD_SITENAME_BINDING_IN_APPHOST_CONFIG` - prevents unexpected host restart. Here is explanation ([link](https://ruslany.net/2019/06/azure-app-service-deployment-slots-tips-and-tricks/)):

    > By default Function Runtime put the site’s hostnames into the site’s applicationHost.config file “bindings” section. Then when the swap happens the hostnames in the applicationHost.config get out of sync with what the actual site’s hostnames are. That does not affect the app in anyway while it is running, but as soon as some storage event occurs, e.g. storage volume fail over, that discrepancy causes the worker process app domain to recycle. If you use this app setting then instead of the hostnames we will put the sitename into the “bindings” section of the appHost.config file. The sitename does not change during the swap so there will be no such discrepancy after the swap and hence there should not be a restart.
- `WEBSITE_RUN_FROM_PACKAGE` - is used for reducing the risk of file copy locking issues and increasing overall deployment performace. See explanation here: [link](https://docs.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package)
   
## How to swap deployment slots

There are two way of doing that: automatic (auto-swap) and manual. For automatin swap it's required to set `WithAutoSwapSlotName("production")` when creating the slot. I found automatic swap less transparent, so the example below shows manual approach. 

```cs
await slot.DeployZip(...); // custom extension method
await slot.SwapAsync("production");
```

A good news that `SwapAsync` waits until slots fully swapped! So we are safe to run any kind of automatic acceptance tests right after `SwapAsync` completed.
    
## Testing

There are two version of the same service (`0.9.6` and `0.9.7`). Version `0.9.6` is deployed to production slot and all traffict is directed to it. The goal is to deploy `0.9.7` seamleslly without affecting consumers.

Just for test purposes I've created the following script:

```powershell
while (1) {
    $response = Invoke-Webrequest -Uri "https://serviceA.company.com/api/healthcheck" -SkipHttpErrorCheck -Headers @{ "Cache-Control" = "no-cache" }

    if ($request.statuscode -eq "200") {
        Write-Host (Get-Date) ":" "Status code:" $response.statuscode " -> Version: " ($response.Content | ConvertFrom-Json).service.version
    } else {
        Write-Host (Get-Date) ":" "Status code:" $response.statuscode
        [console]::beep(500, 300)
    }
    Start-Sleep -s 0.5
}
```
The script simply calls healt-check endpoint every second and logs result to console output. In addition it produces spooky sound each time when service is unavailable :) 

Lets run the script against the staging and production slots. Here is an output:

![Before deployment]({{ "/assets/deployment-slots/2021-07-27_21h13_29.png" | absolute_url }})

First part of the screenshot (1) contains information about production slot, the second part (2) - is for staging slot. As you may see, `production` slot have `0.9.6`, `staging` slot contains `0.9.5` (previously swapped).

Now let's deploy a new version `0.9.7` to `staging` slot and call swap. Here is what happened:

![After deployment]({{ "/assets/deployment-slots/2021-07-27_21h25_21.png" | absolute_url }})

As you may see `production` slot (1) was successfully swithed to `0.9.7` without downtime after warmup on staging slot. In turn, `staging` slot now contains `0.9.6` version.

Happy codding!



