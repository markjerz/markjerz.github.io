---
title: Fixing an Azure App Service Certificate after moving resources to a new subscription
---

# Introduction

For the most part, moving resources to a new Azure subscription is a fairly simple task. You would expect it that though, as it is a management plane operation and so all your compute and storage stays in the same place (and continues to work).

App Service Certificates also move along with the other resources and continue to work against your app services. App Service Certificates are stored in key vaults and this actually requires some manual intervention. When created, the key vault is tied to the default Azure Active Directory tenant for the subscription that it is in. All of the access policies on the key vault are similarly tied to that tenant. Moving the key vault from one tenant to another leads to the key vault being innaccesible to users in the new tenant. As per [this Azure document](https://docs.microsoft.com/en-us/azure/key-vault/general/subscription-move-fix) you need to reset the tenant on the key vault to match that of the new tenant.

As part of this operation you also remove all of the access policies on the existing key vault.

# Certificate Renewal

Azure automatically renews, and re-applies, App Service Certificates on your App Services. However, when you've gone through a subscription move, the re-binding doesn't appear to work correctly and the certificate has to be imported manually.

The first thing to do is to re-configure the key vault for the certificate. Simply access the App Service Certificate configuration and ensure that Step 1 is configured with the correct key vault.

![DSL Editor](/img/app-service-certificate-config.jpg "DSL Editor")

Once you've done that you should be able to import the App Service Certificate via the App Services, by clicking:

![DSL Editor](/img/app-service-certificate-import.jpg "DSL Editor")

Doing this, you pick the existing certificate, Azure validates that it can be imported and then you click ok. All seems good, except in my case where I received the following error:

> Failed to add App Service certificate to the app, Check error for more details. Error Details: The parameter KeyVaultId & KeyVaultSecretName has an invalid value.

If you take a look at the http request that the portal sends you will see the values for the KeyVaultId and the KeyVaultSecretName parameters. In my instance they were the correct values, matching the new subscription and the correct key vault.

I'm not sure what the issue is - I presume that the old tenant/subscription is not updated on one of the resources somewhere. The (inelegant) solution to the problem is to simply remove all of the bindings to the certificate, delete the Microsoft.Web/certificates for that App Service Certificate and then re-import and bind the updated certificate in to your App Services.