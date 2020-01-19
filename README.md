# GitHub Actions for Azure Spring Cloud Sample with Key Vault

## Why Use Key Vault

Key Vault is like a strongbox to store keys. Enterprise users take seriously to the credentials, so they would like not to store the keys to the scope they do not control, such as CI/CD environments.

Then is it a game to get a key with another key? Yes, but it makes sense. Because the key to get credentials in key vault shoud be very limited for resource scope, that is to say, the key to get credentials can only access to the key vault scope, but not the entire Azure scope. That's more like you lock a master key that can open all doors in a building with a key can only open the strongbox.

## Generate Credential to Access to Key Vault - The key to open the strongbox

Execute command below on you local machine:

```bash
az ad sp create-for-rbac --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.KeyVault/vaults/<KEY_VAULT> --sdk-auth
```

Notics the scope following `--scopes` parameter, that the limit of the key to access to the resource, it can only access to the strongbox.

You should get something like

```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
    "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
    "resourceManagerEndpointUrl": "https://management.azure.com/",
    "activeDirectoryGraphResourceId": "https://graph.windows.net/",
    "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
    "galleryEndpointUrl": "https://gallery.azure.com/",
    "managementEndpointUrl": "https://management.core.windows.net/"
}
```

Then save it to GitHub as secrets. If you do not know how to save it to GitHub, see https://github.com/Sneezry/spring-actions#set-azure-credential-and-enable-github-actions.

## Add Access Policies for the Credential

Only create a credential to access to the Key Vault is not enough. The credential created above can only get general information of the Key Vault, but not the contents it stores.

To get secrets stored in the Key Vault, you need add access policies for the credential.

Go to the Key Vault dashboard in Azure Portal, click **Access control** menu, then open Role assignments tab. Select **Apps** for type, **This resource** for scope, and you should see the credential you created in previous step:

![](https://image.sneezry.com/hy9wblgg0uu2.png)

Copy the credential name, `azure-cli-2020-01-19-04-39-02` for example. Open **Access policies** menu, click `+Add Access Policy` link, select **Secret Management** for teplate, then select principal. Paste the credential name in principal select input box:

![](https://image.sneezry.com/s24q12hh9xjh.png)

After add access policy, click save button to save changes.

## Generate Big Scope Azure Crendential - The master key to open all doors in the building

Similar with the first step, we change the scope to bigger to generate the master key:

```bash
az ad sp create-for-rbac --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID> --sdk-auth
```

Again, you get something like

```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
    "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
    "resourceManagerEndpointUrl": "https://management.azure.com/",
    "activeDirectoryGraphResourceId": "https://graph.windows.net/",
    "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
    "galleryEndpointUrl": "https://gallery.azure.com/",
    "managementEndpointUrl": "https://management.core.windows.net/"
}
```

Copy the entire JSON string carefully, go back to Key Vault dashboard, open **Secrets** menu, then click **Generate/Import** button. Input the secret name, such as `AZURE-CRENDENTIALS-FOR-SPRING`. Paste the crendential JSON string to **Value** input box. You may notice the vaule input box is an one-line text feild, rather then a multi-line text area. That's fine, just paste the JSON string there, it works.

![](https://image.sneezry.com/iuvskcdnl9je.png)

## Combine They All in GitHub Actions:

```yaml
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}                              # Strongbox key you generated in the first step
    - uses: Azure/get-keyvault-secrets@v1.0
      with:
        keyvault: "zlhe-test"
        secrets: "AZURE-CREDENTIALS-FOR-SPRING"                              # Master key to open all doors in the building
      id: keyvaultaction
    - uses: azure/login@v1
      with:
        creds: ${{ steps.keyvaultaction.outputs.AZURE-CREDENTIALS-FOR-SPRING }}
    - name: Azure CLI script
      uses: azure/CLI@v1
      with:
        azcliversion: 2.0.75
        inlineScript: |
          az extension add --name spring-cloud                               # Spring CLI commands from here
          az spring-cloud list
```
