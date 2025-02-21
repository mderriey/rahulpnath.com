---
title: "Variable Snapshotting in Azure DevOps When Using Azure Key Vault"
comments: true
date: 2019-07-18
categories:
- Azure DevOps
- Azure Key Vault
---

In the previous post, we saw how to [Enable History for Azure DevOps Variable Groups Using Azure Key Vault](/blog/azure-devops-variable-groups-history/). However, there is one issue with using this approach. Whenever a deployment is triggered, it fetches the latest Secret value from the Key Vault. This behaviour might be desirable or not depending on the nature of the Secret.

In this post, we will see how we can use an alternative approach using the Azure Key Vault Pipeline task to fetch Secrets and at the same time, allow us to snapshot variable values against a release.

### Secrets in Key Vault 

Before we go any further, lets looks look at how Secrets looks like in Key Vault. You can create Secret in Key Vault via various mechanisms including Powershell, cli, portal, etc. From the portal, you can create a new Secret as shown below (from the Secrets section under the Key Vault). A Secret is uniquely identifiable by the name of the Vault, Secret Name, and the version identifier. Without the version identifier, the latest value is used. 

![](/images/azure_keyvault_secrets.jpg)

Since we have only one Secret Version created, this can be identified using the full SecretName/Identifier or just the SecretName as both are the same. 

> Depending on how you want your consuming application to get a Secret value you should choose how to refer a Secret - using the name only (to always get the latest version) or using the SecretName/Identifier to get the specific version.

### Azure Key Vault Pipeline Task

The [Azure Key Vault task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-key-vault?view=azure-devops0 can be used to fetch all or a subset of Secrets from the vault and set them as variables that is available in the subsequent tasks of a pipeline. Using the *[secretsFilter](https://github.com/microsoft/azure-pipelines-tasks/tree/master/Tasks/AzureKeyVaultV1#parameters-of-the-task)* property on the task, it supports either downloading all the Secrets (as of their latest version) or specify a subset of Secret names to download. When specifying the names you choose either of the two formats - *SecretName* or *SecretName/VersionIdentifier*.

When using the *SecretName* the Secret is available against the same name in the subsequent tasks as a Variable. With *SecretName/Identifier* format the Secret is available as a variable with name *SecretName/Identifier* and also with the name *SecretName* (excluding the Identifier part).

 E.g.  If the *secretsFilter* property is set to *ConnectionString/c8b9c1dd4e134118b13568a26f8d9778*, two variables will be created after the Azure Key Vault task step - one with name *ConnectionString/c8b9c1dd4e134118b13568a26f8d9778* and another one with name *ConnectionString*. 

![](/images/keyvault_task_azure_devops.jpg)

The application configuration can use either of the two names, mostly *ConnectionString*. We do not want the application configuration to be tightly coupled with the Secret Identifier in Key Vault.

### Azure DevOps Variable Groups

With the Key Vault integration now moved out of the [Variable Groups](/blog/azure-devops-variable-groups-history/) it can either be blank or have the list of SecretNames. For the Azure Key Vault task the variables names can either be passed in as a parameter value specified in the Variable Groups (as shown in the image above) or we can specify the actual names (along with identifiers) as part of the task itself. I prefer to pass it as a variable from the Variable Groups. It can either be on a variable with comma separated values or multiple variables passed as comma separated values in the task. To minimize editing the task on every release I chose to pass it as a single variable with comma separated values (as shown below).

![](/images/azure_devops_variables.jpg)

Having the Azure Key Vault task as the first task in the pipeline all subsequent tasks will have the variables from Key Vault available to use - including file transforms and variable substitution options. Using the specific version of the Secret locks down the Secret value at that time to a release created. When the secret value is updated, the variable will need to be updated with the new version identifier for it to use the latest version. If not it will still the older version of the Secret Value. This behaviour is now exactly as with the default variables in the DevOps pipeline - by snapshotting the variable values as part of the release. 


*The above [feature](https://github.com/microsoft/azure-pipelines-tasks/issues/10445) is part of the Azure Key Vault Task as of [version 1.0.37](https://developercommunity.visualstudio.com/content/problem/622521/azure-devops-pipeline-task-does-not-show-full-vers.html).*