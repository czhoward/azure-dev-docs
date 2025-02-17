---
title: Securely ingest Azure Blob Storage data using Azure Key Vault with a Python Azure Function
description: Learn how to retrieve a secret from Azure Key Vault to access Azure Storage Blob, using a serverless Python Azure Function.
services: python, azure-functions, azure-key-vault, azure-storage-accounts
ms.custom: devx-track-python, devx-track-azurepowershell, devx-track-azurecli
ms.devlang: python
ms.topic: how-to
ms.prod: azure-python
author: jess-johnson-msft
ms.author: jejohn
ms.date: 08/19/2021
---
# Ingest data from Azure Blob Storage using a Python Azure Function and Azure Key Vault

In this article, you'll learn how to retrieve a secret from Azure Key Vault to access Azure Storage Blob, using a serverless Python Function.

![Relational Data Ingestion - Securely Extract Data diagram.](./media/quickstart-securely-retrieve-blob-data/qs_akv_asb_fun-INGESTION-Simplified2.png)

## Prerequisites

* An active Azure subscription - [create one for free](https://azure.microsoft.com/free/)
* [Azure CLI](/cli/azure/install-azure-cli) or [PowerShell 7](/powershell/scripting/install/installing-powershell-core-on-windows)
* The [Azure Functions Core Tools](/azure/azure-functions/functions-run-local) version 3.x.
* [Visual Studio Code](https://code.visualstudio.com/) on one of the [supported platforms](https://code.visualstudio.com/docs/supporting/requirements#_platforms).
* The [PowerShell extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell).
* The [Azure Functions extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions) for Visual Studio Code.
* [Python 3.6+](./configure-local-development-environment.md) is required and the following packages:
  * azure-storage-blob `pip install azure-storage-blob`
  * azure-identity `pip install azure-identity`
  * azure-keyvault-keys `pip install azure-keyvault-secrets`

## 1. Configure your environment
This article assumes the following Azure Resources have **already been provisioned**:

* Azure Active Directory (Azure AD), sign-up or learn more about [Azure AD](/azure/active-directory/fundamentals/sign-up-organization)
* Azure Resource Group, to create a new Resource Group you can use the [Azure portal](/azure/azure-resource-manager/management/manage-resource-groups-portal), [Azure PowerShell](/powershell/module/az.resources/new-azresourcegroup), or [Azure CLI](/cli/azure/group)
* Azure Storage Account, to create a new Storage Account you can use the [Azure portal](/azure/storage/common/storage-quickstart-create-account?tabs=azure-portal), [Azure PowerShell](/azure/storage/common/storage-quickstart-create-account?tabs=azure-powershell), or [Azure CLI](/azure/storage/common/storage-quickstart-create-account?tabs=azure-cli)
* Azure Key Vault, to create a new Key Vault you can use the [Azure portal](/azure/key-vault/keys/quick-create-portal), [PowerShell](/azure/key-vault/keys/quick-create-powershell), or [Azure CLI](/azure/key-vault/keys/quick-create-cli)
* HTTP Trigger or Blob Trigger Azure Function, to create a new Function you can use the [Visual Studio Code](/azure/azure-functions/create-first-function-vs-code-python), [Azure PowerShell](/azure/azure-functions/create-first-function-vs-code-powershell), or [Azure CLI](/azure/azure-functions/create-first-function-cli-python)

The following names and IDs are needed later use in this article:

* Azure Function App ID
* Azure Storage Account Name
* Azure Storage Account ID
* Azure Key Vault Name
* Azure Active Directory User

## 2. Upload a CSV to a blob container

To ingest relational data using a serverless Python Azure Function, upload a data (blob) to an Azure Storage container. If you already have your data (blob) uploaded, you can skip to the next section.

Create a file named '*financial_sample.csv*' locally that contains this data.

|Segment|Country|Product|Units Sold|Manufacturing Price|Sale Price|Gross Sales|Date|
|----|----|----|----|----|----|----|----|
|Government|Canada|Carretera|1618.5|$3.00|$20.00|$32,370.00|1/1/2014|
|Government|Germany|Carretera|1321|$3.00|$20.00|$26,420.00|1/1/2014|
|Midmarket|France|Carretera|2178|$3.00|$15.00|$32,670.00|6/1/2014|
|Midmarket|Germany|Carretera|888|$3.00|$15.00|$13,320.00|6/1/2014|
|Midmarket|Mexico|Carretera|2470|$3.00|$15.00|$37,050.00|6/1/2014|

Copy the below data into the file:

```csv
Segment,Country,Product,Units Sold,Manufacturing Price,Sale Price,Gross Sales,Date
Government,Canada,Carretera,1618.5,$3.00,$20.00,"$32,370.00",1/1/2014
Government,Germany,Carretera,1321,$3.00,$20.00,"$26,420.00",1/1/2014
Midmarket,France,Carretera,2178,$3.00,$15.00,"$32,670.00",6/1/2014
Midmarket,Germany,Carretera,888,$3.00,$15.00,"$13,320.00",6/1/2014
Midmarket,Mexico,Carretera,2470,$3.00,$15.00,"$37,050.00",6/1/2014
```

Upload your data (blob) to your storage container.

### [PowerShell](#tab/azure-powershell)

Run [Set-AzStorageBlobContent](/powershell/module/az.storage/set-AzStorageblobcontent) to upload data to an Azure Blob Storage container.

```powershell
Set-AzStorageBlobContent -File "<file-path>" -Container "<container-name>" -Blob "financial_sample.csv" -Context "<storage-account-context>" 
```

### [Azure CLI](#tab/azure-cli)

Run [az storage blob upload](/cli/azure/storage/blob#az_storage_blob_upload) to upload data to an Azure Blob Storage container.

```azurecli
az storage blob upload --account-name "<storage-account>" --container-name "<container>" --name "financial_sample" --file "financial_sample.csv" --auth-mode login
```

* * *

## 3. Set a secret to the blob access key

Create a 'secret' in Azure Key Vault to store the Storage Account access key.

### [PowerShell](#tab/azure-powershell)

Run [Set-AzKeyVaultSecret](/powershell/module/az.keyvault/set-azkeyvaultsecret) to create a secret in Azure Key Vault.

``` powershell
Set-AzKeyVaultSecret -VaultName "<keyvault-name>" -Name "BlobAccessKey" -SecretValue "<secret-value>"
```

### [Azure CLI](#tab/azure-cli)

Run [az keyvault secret set](/cli/azure/keyvault/secret) to create a secret in Azure Key Vault.

``` azurecli
az keyvault secret set --vault-name "<keyvault-name>" --name "BlobAccessKey" --value "<secret-value>"
```

* * *

>[!IMPORTANT]
>A common approach for storing sensitive information is to move the data from the application code into a 'config.json' file. However, this practice still stores the sensitive information in plain text. We recommend instead using [Azure Key Vault](https://azure.microsoft.com/services/key-vault/). Azure Key Vault is a secure centralized cloud solution for storing and managing sensitive information, such as passwords, certificates, and keys.

## 4. Configure access between storage and key vault

Next, you need to authorize Azure Key Vault to access your Azure Storage Account to manage your Storage Account access key. Learn more about the [built-in Azure roles](/azure/role-based-access-control/built-in-roles) and [Access Policies](/azure/key-vault/general/security-features#privileged-access).

### [PowerShell](#tab/azure-powershell)

Run [New-AzRoleAssignment](/powershell/module/az.resources/new-azroleassignment), [Set-AzKeyVaultAccessPolicy](/powershell/module/az.keyvault/set-azkeyvaultaccesspolicy), [Add-AzKeyVaultManagedStorageAccount](/powershell/module/az.keyvault/add-azkeyvaultmanagedstorageaccount) to authorize Key Vault to access your Storage Account.

```powershell
# Set the Storage Account keys regeneration period to 30 days
$regenPeriod = [System.Timespan]::FromDays(30)

# Assign Azure role "Storage Account Key Operator Service Role" to Key Vault, limiting the access scope to your Storage Account
New-AzRoleAssignment -ApplicationId "<your-function-app-id>" -RoleDefinitionName "Storage Account Key Operator Service Role" -Scope "<storage-account-id>"

# Give your user account permission to managed Storage Accounts
Set-AzKeyVaultAccessPolicy -VaultName "<keyvault-name>" -UserPrincipalName "<user@domain.com>" -PermissionsToSecrets get,set,delete

# Add your Storage Account to your Key Vault's managed Storage Accounts
Add-AzKeyVaultManagedStorageAccount -VaultName "<keyvault-name>" -AccountName "<storage-account-name>" -AccountResourceId "<storage-account-id>" -ActiveKeyName "<key1>" -RegenerationPeriod $regenPeriod
```

### [Azure CLI](#tab/azure-cli)

Run [az role assignment create](/cli/azure/role/assignment), [az keyvault set-policy](/azure/key-vault/general/assign-access-policy-cli), [az keyvault storage add](/cli/azure/keyvault/storage) to authorize Key Vault to access your Storage Account.

```azurecli
# Assign Azure role "Storage Account Key Operator Service Role" to Key Vault, limiting the access scope to your Storage Account
az role assignment create --role "Storage Account Key Operator Service Role" --assignee 'https://vault.azure.net' --scope "/subscriptions/<subscription-id>/resourceGroups/<storage-account-resource-group-name>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>"

# Give your user account permission to managed Storage Accounts
az keyvault set-policy --name "<keyvault-name>" --upn user@domain.com --storage-permissions get,set,delete

# Add your Storage Account to your Key Vault's managed Storage Accounts
az keyvault storage add --vault-name "<keyvault-name>" -n "<storage-account-name>" --active-key-name key1 --auto-regenerate-key --regeneration-period P90D --resource-id "/subscriptions/<subscription-id>/resourceGroups/<storage-account-resource-group-name>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>"
```

* * *

## 5. Retrieve Key Vault secret in an Azure Function

The storage access key is now securely stored in a centralized Key Vault. Now the secret value (blob access key) can be retrieved within the Azure Function.

Storing secrets in Azure Key Vault, rather than storing the sensitive data in plain text, improves the security of your sensitive information.

>[!IMPORTANT]
>Be sure to add the Environment Variable values to both the 'local.setting.json' file for local development, and the [Azure Function appsettings configuration](/azure/azure-functions/functions-how-to-use-azure-function-app-settings).

``` Python
import logging
import os
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

# Set variables to storage account information.
storage_account_name = "<your-storage-account-name>"
blob_container_name = "<your-blob-container-name>"
storage_account_url = f"https://{storage_account_name}.blob.core.windows.net/"
blob_csv_filename = "financial_sample.csv"

try:
  # Set variables from appsettings configurations/Environment Variables.
  key_vault_name = os.environ["AKV_NAME"]
  blob_secret_name = os.environ["AKVS_BLOB_ACCESS_KEY"]
  key_vault_Uri = f"https://{key_vault_name}.vault.azure.net"

  # Authenticate and securely retrieve Key Vault secret for access key value.
  az_credential = DefaultAzureCredential()
  client = SecretClient(vault_url=key_vault_Uri, credential= az_credential)
  access_key_secret = client.get_secret(blob_secret_name)
  
except Exception as e:
  logging.info(e)

.....

```

>[!NOTE]
>In this article, the logged in user is used to authenticate to Key Vault, which is the preferred method for local development. For applications deployed to Azure, managed identity should be assigned to App Service or Virtual Machine, for more information, see [Managed Identity Overview](/azure/active-directory/managed-identities-azure-resources/overview).

## 6. Get data from Azure Storage with serverless Function

Extract, Transform, and Load (ETL) is a common approach used in data processing solutions. In this approach, data is extracted from one or more source systems, then transformed in a 'staging' area. Finally, the processed data is loaded into a data store to be consumed by analytic tools.

Append the below code to your existing code to begin the ETL process. This function will securely *extract* raw data from blob storage into your serverless Azure Function.

``` python
.....
from io import StringIO
import pandas as pd
from azure.storage.blob import BlobClient

blob_client = BlobClient( account_url=storage_account_url, 
                          container_name=blob_container_name, 
                          blob_name=blob_csv_filename, 
                          credential=access_key_secret.value )

# Download blob file as StorageStreamDownloader object (stored in memory)
downloaded_blob = blob_client.download_blob()

# Load blob data into a Pandas DataFrame
df = pd.read_csv(StringIO(downloaded_blob.content_as_text()))

# Check the blob file data
df
.....
```

```console
    Segment     Country  Product    Units Sold  Manufacturing Price  Sale Price   Gross Sales   Date
0   Government  Canada   Carretera  1618.5      $3.00                $20.00       "$32,370.00"  1/1/2014
1   Government  Germany  Carretera  1321        $3.00                $20.00       "$26,420.00"  1/1/2014
2   Midmarket   France   Carretera  2178        $3.00                $15.00       "$32,670.00"  6/1/2014
3   Midmarket   Germany  Carretera  888         $3.00                $15.00       "$13,320.00"  6/1/2014
4   Midmarket   Mexico   Carretera  2470        $3.00                $15.00       "$37,050.00"  6/1/2014
```

## 7. Clean up resources

When no longer needed, remove the resource group, and all related resources:

### [PowerShell](#tab/azure-powershell)

Run [Remove-AzResourceGroup](/powershell/module/az.resources/remove-azresourcegroup) to delete the Azure Resource Group.

``` powershell
Set-AzResourceGroup -Name "MyResourceGroup"
```

### [Azure CLI](#tab/azure-cli)

Run [az group delete](/cli/azure/group) to delete the Azure Resource Group.

```azurecli
az group delete --name myresourcegroup
```

* * *

## Next steps

Continue your Python on Azure data journey with the below articles:

* [Tasks to prepare data for enhanced machine learning](/azure/architecture/data-science-process/prepare-data)