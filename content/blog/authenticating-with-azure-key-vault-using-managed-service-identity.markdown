---
layout: post
title: "Authenticating with Azure Key Vault Using Managed Service Identity"
comments: true
categories: 
- Azure Key Vault
tags: 
date: 2017-10-18
completedDate: 2017-10-18 19:10:36 +1100
keywords: 
description: Using Managed Service Identity to authenticate a client application to connect with Azure Key Vault
primaryImage: keyvault_managed_service_identity.png
---

***How do you secure the access keys to the Key Vault itself?***.

If you use ClientId/Secret to authenticate with a key vault, then you are likely to end up having these in the web.config file ([there still are ways around](http://www.rahulpnath.com/blog/keeping-sensitive-configuration-data-out-of-source-control/)) which is what we initially set out to avoid, by using Azure Key Vault. The recommended approach till now was to use certificate-based authentication so that you need to have only the thumbprint id of the certificate in the web.config and you can deploy the certificate along with the application. If you are not familiar with either way of authenticating with Key Vault, then check out this [article](http://www.rahulpnath.com/blog/authenticating-a-client-application-with-azure-key-vault/). With the Secret or certificate-based authentication, we also run into the problem of credentials expiring which in turn can lead to application downtime.

Managed Service Identity (MSI) solves this problem by allowing an Azure App Service, Azure Virtual Machines or Azure Functions to connect to Key Vault (and a few other services) without any explicit credentials in the code.

> *[Managed Service Identity](https://docs.microsoft.com/en-us/azure/active-directory/msi-overview) (MSI) makes solving this problem simpler by giving Azure services an automatically managed identity in Azure Active Directory (Azure AD). You can use this identity to authenticate to any service that supports Azure AD authentication, including Key Vault, without having any credentials in your code.*

MSI can be enabled through the Azure Portal. E.g., to enable MSI for App Service, the portal has an option as shown below.

<img src="/images/keyvault_managed_service_identity.png" class="center" alt="Enable Managed Service Identity for Azure App Service" />

Once enabled we can add an access policy in the key vault to give permissions to the Azure App service. Search by the app service name and assign the required access policies.

For an application to access the key vault, we need to use *AzureServiceTokenProvider* from [Microsoft.Azure.Services.AppAuthentication](https://www.nuget.org/packages/Micros) NuGet package. Instead of using the ClientCredential or ClientAssertionCertificate to acquire the token, we will use AzureServiceTokenProvider to acquire the token for us.

``` csharp
var azureServiceTokenProvider = new AzureServiceTokenProvider();
var keyVaultClient = new KeyVaultClient(new KeyVaultClient.AuthenticationCallback(azureServiceTokenProvider.KeyVaultTokenCallback));
```
*The [AzureServiceTokenProvider](https://azure.microsoft.com/en-us/resources/samples/app-service-msi-keyvault-dotnet/) class tries the following methods to get an access token:-*

1. *Managed Service Identity (MSI) - for scenarios where the code is deployed to Azure, and the Azure resource supports MSI.*
2. *Azure CLI (for local development) - Azure CLI version 2.0.12 and above supports the get-access-token option. AzureServiceTokenProvider uses this option to get an access token for local development.* 
3. *Active Directory Integrated Authentication (for local development). To use integrated Windows authentication, your domain’s Active Directory must be federated with Azure Active Directory. Your application must be running on a domain-joined machine under a user’s domain credentials.*

### Local Development

For the AzureServiceTokenProvider to work locally we need to install the [Azure CLI]( https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) and also setup an environment variable - *AzureServicesAuthConnectionString*. Depending on whether you want to use ClientId/Secret or ClientId/Certificate-based authentication the value for the environment variable changes.

``` text
AzureServicesAuthConnectionString to RunAs=App;AppId=AppId;TenantId=TenantId;AppKey=Secret.
Or
RunAs=App;AppId=AppId;TenantId=TenantId;CertificateThumbprint=Thumbprint;CertificateStoreLocation=CurrentUser
```

<img src="/images/kkeyvault_msi_tenantId.png" class="center" alt="Get Tenant Id and AppId" />

As shown above, you can get the TenantId and AppId from the App Registrations page in the Azure portal. Clicking on the Endpoints button reveals a list of URL's which has the TenantId GUID. The AppId is displayed against each of the AD application. Once you set the environment variable, the application will be able to connect to Key Vault without any additional configuration entries in web/app config.

Azure Managed Service Identity makes it easier to connect to Key Vault and removes the need of having any sensitive information in the application configuration file. It also helps remove the overhead of renewing the certificate/secrets used to connect to the Vault. One less thing to worry about the application!