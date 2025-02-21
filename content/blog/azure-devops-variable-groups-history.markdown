---
title: "Enable History for Azure DevOps Variable Groups Using Azure Key Vault"
date: 2019-06-12
comments: true
categories: 
- Azure DevOps
- Azure Key Vault
---
***Using Key Vault via Variable Group does not snapshot configuration values from the vault. When a deployment is triggered the latest value from the Key Vault is used.***

A couple of weeks back, I accidentally deleted an Azure DevOps Variable Group at work, and it sucked out half of my day. I was lucky that it was the development environment. Since I had access to the environment, I remoted to the corresponding machines (using the Kudu console) and retrieved the values from the appropriate config files. Even with all access in place, it still took me a couple of hours to fix it. This is not a great place to be.

{{< tweet 1130369563135160320>}}

In this post, let us see how we can link secrets from Key Vault to DevOps Variable Groups and how it helps us from accidental deletes and similar mistakes. Check out my video on [Getting started with Key Vault](https://www.youtube.com/watch?v=51Qmk3TQJ44) and other [related articles](https://www.rahulpnath.com/blog/category/azure-key-vault/) if you are new to Key Vault. 

> [Azure Key Vault](https://azure.microsoft.com/en-au/services/key-vault/) enables safeguard cryptographic keys and secrets used by the application. It increases security and control over keys and passwords.

### Link Variable Group to Key Vault

Azure DevOps supports [linking Secrets from an Azure Key Vault to a Variable Group](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml#link-secrets-from-an-azure-key-vault). When creating a new variable group toggle on the '*Link secrets from an Azure Key Vault as variables*' option. You can now link the Azure subscription and the associated Key Vault to retrieve the secrets. Clicking the Authorize button next to the Key Vault sets the required permissions on the Key Vault for the Azure Service Connection (that what connects your DevOps account with the Azure subscription).

![Azure DevOps Variable Groups and Azure Key Vault](/images/devops_variable_groups_key_vault.jpg)

The Add button pops up a dialog as shown below. It allows you to select the Secrets that needs to be available as part of the Variable Group. 

![Azure DevOps Variable Groups link secrets from Vault](/images/devops_variable_groups_key_vault_secrets.jpg)

Create an Azure Key Vault for each environment to manage Secrets for that environment. As per the current [pricing](https://azure.microsoft.com/en-au/pricing/details/key-vault/), creating a Key Vault does not have any cost associated. Cost is based on operations to Key Vault - around *USD $0.03/10,000 transactions* (at the time of writing.)

### Version History for Variable Changes

Key Vault supports versioning and creates a new version of an object (key/secret/certificate) each time it is updated. This helps keep track of previous values. You can set expiry/activation dates on the secret if applicable. Further, by having [Expiry Notification for Azure Key Vault](https://rahulpnath.com/blog/expiry-notification-for-azure-key-vault-keys-and-secrets/) set up, you can stay on top of rotating your secrets/certificates on time. The Variable Group refers only to the Secret names in the Key Vault. The secret names are the same as what is in the application configuration file. Every time a release is deployed, it reads the latest value of the Secret from the associated Key Vault and uses that for the deployment. This is different from when defining the variables directly as part of the group, where the variables are snapshot at the time of release.

> For every deployment, the latest version of the Secret is read from the associated Key Vault. 

Make sure this behavior is acceptable with your deployment scenario before moving to use Key Vault via Variable Groups. However, there is a different plugin that you can use to achieve variable snapshotting even with Key Vault, which I will cover in a separate post.

### Handling Accidental Deletes

In case anyone accidentally deletes a variable group in Azure DevOps, it is as simple as cloning one of your other environments and renaming to be Dev Variable group. Mostly it's the same set of variables across all environments. The actual secret value is not required anymore, as that is managed in Key Vault.

**For argument sake, what if I accidentally delete the Key Vault itself?**

Good news is Key Vault does have a [Recovery Option](https://blogs.technet.microsoft.com/kv/2017/05/10/azure-key-vault-recovery-options/). Assuming you create the Key Vault with the recovery options set (which you obviously will now), using the *EnableSoftDelete* parameter from Powershell, you can recover back from any delete action on the vault/key/secret.

Hope this helps save half a day (or even more) of someone (maybe me again) who accidentally deletes a variable group!