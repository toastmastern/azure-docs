﻿---
ms.assetid: 
title: Azure Key Vault Storage Account Keys
description: Storage account keys provide a seemless integration between Azure Key Vault and key based access to Azure Storage Account.
ms.topic: article
services: key-vault
ms.service: key-vault
author: BrucePerlerMS
ms.author: bruceper
manager: mbaldwin
ms.date: 07/25/2017
---
# Azure Key Vault Storage Account Keys

Before Azure Key Vault Storage Account Keys, developers had to manage their own Azure Storage Account (ASA) keys and rotate them manually or through an external automation. Now, Key Vault Storage Account Keys are implemented as [Key Vault secrets](https://docs.microsoft.com/rest/api/keyvault/about-keys--secrets-and-certificates#BKMK_WorkingWithSecrets) for authenticating with an Azure Storage Account. 

The ASA key feature manages secret rotation for you and removes the need for your direct contact with an ASA key by offering Shared Access Signatures (SAS) as a method. 

For more general information on Azure Storage Accounts, see [About Azure storage accounts](https://docs.microsoft.com/azure/storage/storage-create-storage-account).

## Supporting interfaces

The Azure Storage Account keys feature is initially available through the REST, .NET/C# and PowerShell interfaces. For more information, see [Key Vault Reference](https://docs.microsoft.com/azure/key-vault/).

## Storage account keys behavior

### What Key Vault manages

Key Vault performs several internal management functions on your behalf when you use Storage Account Keys.


1. Azure Key Vault manages keys of an Azure Storage Account (SAS). 
    - Internally, Azure Key Vault can list (sync) keys with an Azure Storage Account.  
    - Azure Key Vault regenerates (rotates) the keys periodically. 
    - Key values are never returned in response to caller. 
    - Azure Key Vault manages keys of both Storage Accounts and Classic Storage Accounts. 
2. Azure Key Vault allows you, the vault/object owner, to create SAS (account or service SAS) definitions. 
    - The SAS value, created using SAS definition, is returned as a secret via the REST URI path. For more information, see [Azure Key Vault storage account operations](https://docs.microsoft.com/rest/api/keyvault/storage-account-key-operations).

### Naming guidance

Storage account names must be between 3 and 24 characters in length and may contain numbers and lowercase letters only.  
 
A SAS definition name must be 1-102 characters in length containing only 0-9, a-z, A-Z.

## Developer experience 

### Before Azure Key Vault Storage Keys 

Developers used to need to do the following practices with a storage account key to get access to Azure storage. 
 
 ```
//create storage account using connection string containing account name 
// and the storage key 

var storageAccount = CloudStorageAccount.Parse(CloudConfigurationManager.GetSetting("StorageConnectionString"));
var blobClient = storageAccount.CreateCloudBlobClient();
 ```
 
### After Azure Key Vault Storage Keys 

```
//Use PowerShell command to get Secret URI 

Set-AzureKeyVaultManagedStorageSasDefinition -Service Blob -ResourceType Container,Service -VaultName yourKV  
-AccountName msak01 -Name blobsas1 -Protocol HttpsOnly -ValidityPeriod ([System.Timespan]::FromDays(1)) -Permission Read,List

//Get SAS token from Key Vault

var secret = await kv.GetSecretAsync("SecretUri");

// Create new storage credentials using the SAS token. 

var accountSasCredential = new StorageCredentials(secret.Value); 

// Use credentials and the Blob storage endpoint to create a new Blob service client. 

var accountWithSas = new CloudStorageAccount(accountSasCredential, new Uri ("https://myaccount.blob.core.windows.net/"), null, null, null); 

var blobClientWithSas = accountWithSas.CreateCloudBlobClient(); 
 
// If SAS token is about to expire then Get sasToken again from Key Vault and update it.

accountSasCredential.UpdateSASToken(sasToken);

  ```
 
 ### Developer best practices 

- Only allow Key Vault to manage your ASA keys. Do not attempt to manage them yourself, you will interfere with Key Vault's processes. 
- Do not allow ASA keys to be managed by more than one Key Vault object. 
- If you need to manually regenerate your ASA keys, we recommend that you regenerate them via Key Vault. 

## Getting started

### Setup for role-based access control (RBAC) permissions

Key Vault needs permissions to *list* and *regenerate* keys for a storage account. Set up these permissions using the following steps:

- Get ObjectId of Key Vault: 

    `Get-AzureRmADServicePrincipal -SearchString "AzureKeyVault"`

- Assign Storage Key Operator role to Azure Key Vault Identity: 

    `New-AzureRmRoleAssignment -ObjectId <objectId of AzureKeyVault from previous command> -RoleDefinitionName 'Storage Account Key Operator Service Role' -Scope '<azure resource id of storage account>'`

    >[!NOTE]
    > For a classic account type, set the role parameter to *"Classic Storage Account Key Operator Service Role"*.

### Storage account onboarding 

Example: As a Key Vault object owner you add a storage account object to your Azure Key Vault to onboard a storage account.

During onboarding, Key Vault needs to verify that the identity of the onboarding account has permissions to *list* and to *regenerate* storage keys. In order to verify these permissions, Key Vault gets an OBO (On Behalf Of) token from the authentication service, audience set to Azure Resource Manager, and makes a *list* key call to the Azure Storage service. If the *list* call fails, the Key Vault object creation fails with a HTTP status code of *Forbidden*. The keys listed in this fashion are cached with your key vault entity storage. 

Key Vault must verify that the identity has *regenerate* permissions before it can take ownership of regenerating your keys. To verify that the identity, via OBO token, as well as the Key Vault first party identity has these permissions:

- Key Vault lists RBAC permissions on the storage account resource.
- Key Vault validates the response via regular expression matching of actions and non-actions. 

Find some supporting examples at [Key Vault - Managed Storage Accont Keys Samples](https://github.com/Azure/azure-sdk-for-net/blob/psSdkJson6/src/SDKs/KeyVault/dataPlane/Microsoft.Azure.KeyVault.Samples/samples/HelloKeyVault/Program.cs#L167).

If the identity does not have *regenerate* permissions or if Key Vault's first party identity doesn’t have *list* or *regenerate* permission, then the onboarding request fails returning an appropriate error code and message. 

The OBO token will only work when you use first-party, native client applications of either PowerShell or CLI.

## Other applications

- SAS tokens, constructed using Key Vault storage account keys, provide even more controlled access to an Azure storage account. For more information, see [Using shared access signatures](https://docs.microsoft.com/azure/storage/storage-dotnet-shared-access-signature-part-1).

## See also

- [About keys, secrets, and certificates](https://docs.microsoft.com/rest/api/keyvault/)
- [Key Vault Team Blog](https://blogs.technet.microsoft.com/kv/)
