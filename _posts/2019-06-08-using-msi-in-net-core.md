---
layout: post
title: "Azure managed identities: specificities for local development under .Net Core"
date: 2019-06-08 22:55:00 +0300
categories: azure
tags: netcore, netcoreapp, dotnet, .net, msi, azure, managed, service, identities, resources.
---

Managed identities for Azure resources provides automatic managment for identities in Azure AD in order to authenticate to any resources without having any credentials in the code. I guess a reader is already familiar with managed identities. Just in case here is a [link](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) to appropariate page to start with.

As stated [here](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication), for local authentication the following approaches could be used:

- authenticate using Visual Studio account;
- authenticate using Azure CLI;
- authenticate using Current User account (Integrated Windows Authentication - IWA).

The last approach looks pretty promising, as no any tools required to be installed. But sad fact is that this approach does not work under .NET Code. I've already created ticket in appropriate [github repository](https://github.com/MicrosoftDocs/azure-docs/issues/32439) in order to change documentation as appropriate.

In case integrated windows authentication scenario under .NET Core token acquiring operation fails with the following exception:

```
System.Private.CoreLib: Exception while executing function: TestFunction. Microsoft.Azure.Services.AppAuthentication: Parameters: Connection String: RunAs=CurrentUser, Resource: https://vault.azure.net, Authority: https://login.windows.net/32b5ceed-3d0b-47a1-8c05-78f3c4b70792. Exception Message: Tried to get token using Active Directory Integrated Authentication. Access token could not be acquired. unknown_user: Could not identify logged in user.
```

If we go deeper through the code we can notice that `AzureServiceTokenProvider` uses `ADAL` library which has limited support for IWA (Integrated Windows Authentication) for .NET Core, as under this runtime it is not possible to determine the username (UPN) of the currently logged in user. Please see the following [link](https://github.com/AzureAD/azure-activedirectory-library-for-dotnet/blob/cf3fb60854e5d55edaec880d1d409652f7fca20a/core/src/Platforms/netcore/NetCorePlatformProxy.cs#L53).

So you need to be aware of such limitations under .NET Core. Hope documentation will be updated soon. Votes for the [issue](https://github.com/MicrosoftDocs/azure-docs/issues/32439) are welcomed ;)

Stay tuned and happy coding!