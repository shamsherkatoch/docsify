# Azure keys and secrets rotation
Rotating keys and secrets is a critical security practice to ensure the integrity and confidentiality of your applications. Here’s a step-by-step guide on how to build a solution for keys and secret rotation using Azure Key Vault and Azure DevOps:

## Prerequisites
1. **Azure Subscription**: Ensure you have an active Azure subscription.
2. **Azure Key Vault**: Set up an Azure Key Vault to store your keys and secrets.
3. **Azure DevOps**: Have an Azure DevOps organization and a project set up.

## Step-by-Step Guide

#### Step 1: Create and Configure Azure Key Vault
1. **Create Key Vault**:
   - Go to the Azure portal.
   - Navigate to **Create a resource** > **Security** > **Key Vault**.
   - Fill in the necessary details and create the Key Vault.

2. **Add Secrets and Keys**:
   - Navigate to your Key Vault.
   - Under **Settings**, select **Secrets** or **Keys**.
   - Add the secrets or keys you want to manage.

#### Step 2: Set Up Azure DevOps Service Connection
1. **Service Connection**:
   - In Azure DevOps, go to your project.
   - Navigate to **Project Settings** > **Service connections**.
   - Create a new service connection for Azure Resource Manager.
   - Authenticate and grant necessary permissions to allow Azure DevOps to access your Azure Key Vault.

#### Step 3: Create Azure DevOps Pipeline
1. **Create Pipeline**:
   - In your Azure DevOps project, go to **Pipelines** > **Create Pipeline**.
   - Select the repository where your code resides.

2. **Define Pipeline YAML**:
   - Use the following example YAML to set up a pipeline that will handle the rotation of secrets and keys:

   ```yaml
   trigger:
   - main

   pool:
     vmImage: 'ubuntu-latest'

   variables:
     azureSubscription: 'your-service-connection-name'
     keyVaultName: 'your-key-vault-name'
     secretName: 'your-secret-name'
     newSecretValue: 'new-secret-value' # Replace this with the method to generate new secret value

   stages:
   - stage: RotateSecrets
     jobs:
     - job: Rotate
       steps:
       - task: AzureCLI@2
         inputs:
           azureSubscription: $(azureSubscription)
           scriptType: 'bash'
           scriptLocation: 'inlineScript'
           inlineScript: |
             az keyvault secret set --vault-name $(keyVaultName) --name $(secretName) --value $(newSecretValue)
   ```

   Replace the placeholders with your actual service connection name, key vault name, secret name, and the method to generate a new secret value.

3. **Commit and Run**:
   - Commit the pipeline YAML file to your repository.
   - Run the pipeline to rotate the secret.

#### Step 4: Automate the Key Rotation
1. **Set Up Schedule**:
   - In Azure DevOps, you can set a schedule for your pipeline to run at regular intervals (e.g., daily, weekly).
   - Navigate to your pipeline, click on **Triggers**, and set up the scheduled triggers.

2. **Add Notification**:
   - Configure notifications in Azure DevOps to alert you about the pipeline's success or failure.
   - Go to **Project Settings** > **Notifications**, and set up a new notification rule for pipeline completion.

### Additional Considerations
- **Key Rotation**: For key rotation, use similar steps but focus on keys instead of secrets. Azure CLI commands for keys are slightly different, for example:
  ```bash
  az keyvault key create --vault-name $(keyVaultName) --name $(keyName) --ops encrypt decrypt --kty RSA
  ```

- **Automate Secret Generation**: You can integrate the generation of new secret values with tools like Azure Functions or custom scripts to dynamically generate new values.

- **Security Best Practices**: Ensure that the service principal used in the Azure DevOps service connection has minimal necessary permissions, ideally limited to the Key Vault.

By following these steps, you can automate the rotation of keys and secrets stored in Azure Key Vault using Azure DevOps, enhancing the security of your applications.

# Accessing Azure Key Vault Using User Managed Identity in .NET Full Framework and .NET 6 Applications

## Introduction

Azure Key Vault provides a secure way to manage and access secrets, keys, and certificates. Using a User Managed Identity (UMI) simplifies access to Azure Key Vault by eliminating the need for credentials in the code. This paper explains how to configure and access Azure Key Vault using UMI in both .NET Full Framework and .NET 6 applications.

## Prerequisites

1. **Azure Subscription**: An active Azure subscription.
2. **Azure Key Vault**: An existing Key Vault instance.
3. **User Managed Identity**: A User Managed Identity created and assigned to the application.
4. **Development Environment**: Visual Studio or any other IDE supporting .NET development.

## Setting Up Azure Key Vault and User Managed Identity

1. **Create Azure Key Vault**:
   - Go to the Azure portal.
   - Navigate to "Create a resource" > "Security + Identity" > "Key Vault".
   - Configure the Key Vault and create it.

2. **Add Secrets to Key Vault**:
   - Navigate to the created Key Vault.
   - Go to "Secrets" and add new secrets.

3. **Create a User Managed Identity**:
   - Go to "Create a resource" > "Identity" > "User Managed Identity".
   - Create and configure the identity.

4. **Assign Managed Identity to Azure Key Vault**:
   - Navigate to the Key Vault's "Access policies".
   - Add a new access policy, select the User Managed Identity, and assign necessary permissions (e.g., Get, List secrets).

## Accessing Azure Key Vault in .NET Full Framework

### Step 1: Install NuGet Packages
Add the following NuGet packages to your project:
```sh
Install-Package Microsoft.Azure.KeyVault
Install-Package Microsoft.Azure.Services.AppAuthentication
```

### Step 2: Configure Azure AD Authentication
Create a class for Key Vault service:
```csharp
using Microsoft.Azure.KeyVault;
using Microsoft.Azure.Services.AppAuthentication;
using System.Threading.Tasks;

public class KeyVaultService
{
    private readonly KeyVaultClient _keyVaultClient;

    public KeyVaultService()
    {
        var azureServiceTokenProvider = new AzureServiceTokenProvider();
        _keyVaultClient = new KeyVaultClient(new KeyVaultClient.AuthenticationCallback(azureServiceTokenProvider.KeyVaultTokenCallback));
    }

    public async Task<string> GetSecretAsync(string vaultBaseUrl, string secretName)
    {
        var secret = await _keyVaultClient.GetSecretAsync($"{vaultBaseUrl}/secrets/{secretName}")
                                          .ConfigureAwait(false);
        return secret.Value;
    }
}
```

### Step 3: Retrieve Secrets
Use the `KeyVaultService` class to retrieve secrets:
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var keyVaultService = new KeyVaultService();
        string secretValue = await keyVaultService.GetSecretAsync("https://your-key-vault-name.vault.azure.net", "your-secret-name");
        
        Console.WriteLine($"Secret Value: {secretValue}");
    }
}
```

## Accessing Azure Key Vault in .NET 6

### Step 1: Install NuGet Packages
Add the following NuGet packages to your project:
```sh
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

### Step 2: Configure Azure AD Authentication
Create a class for Key Vault service:
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using System.Threading.Tasks;

public class KeyVaultService
{
    private readonly SecretClient _secretClient;

    public KeyVaultService(string keyVaultUrl)
    {
        _secretClient = new SecretClient(new Uri(keyVaultUrl), new DefaultAzureCredential());
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value;
    }
}
```

### Step 3: Retrieve Secrets
Use the `KeyVaultService` class to retrieve secrets:
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var keyVaultService = new KeyVaultService("https://your-key-vault-name.vault.azure.net");
        string secretValue = await keyVaultService.GetSecretAsync("your-secret-name");

        Console.WriteLine($"Secret Value: {secretValue}");
    }
}
```

## Conclusion

By following these steps, you can securely access Azure Key Vault secrets in both .NET Full Framework and .NET 6 applications using User Managed Identity. This approach enhances security by removing hardcoded credentials and utilizing Azure's identity management capabilities. For further details and advanced configurations, refer to the [Azure Key Vault documentation](https://docs.microsoft.com/en-us/azure/key-vault/).


# Design Paper: Using Azure Key Vault for Storing Secrets Instead of Azure DevOps Secrets Variable

## Introduction

Azure DevOps provides a convenient way to store and manage secrets using its Secrets variable feature. However, using Azure Key Vault offers a more secure, scalable, and flexible approach for managing secrets. This paper outlines the advantages, architecture, implementation steps, and best practices for using Azure Key Vault instead of Azure DevOps Secrets variables.

## Advantages of Using Azure Key Vault

1. **Enhanced Security**:
   - **Centralized Management**: Azure Key Vault provides a centralized repository for managing secrets, keys, and certificates.
   - **Access Control**: Fine-grained access policies ensure that only authorized applications and users can access secrets.
   - **Auditing and Monitoring**: Detailed logging and monitoring capabilities help track access and usage patterns.

2. **Scalability**:
   - **Automated Secret Rotation**: Azure Key Vault supports automated secret rotation, reducing the risk of using outdated or compromised secrets.
   - **Integration with Azure Services**: Seamlessly integrates with other Azure services, enabling a scalable solution for managing secrets across different environments and applications.

3. **Compliance**:
   - **Regulatory Compliance**: Azure Key Vault helps meet regulatory requirements by providing a secure and compliant way to manage sensitive information.

## Architecture

The architecture involves the following components:
1. **Azure Key Vault**: Stores secrets, keys, and certificates.
2. **Azure DevOps Pipeline**: Retrieves secrets from Azure Key Vault during build and release processes.
3. **Azure Active Directory (AAD)**: Manages authentication and authorization for accessing Azure Key Vault.

## Implementation Steps

### Step 1: Create Azure Key Vault

1. **Navigate to Azure Portal**.
2. **Create a New Key Vault**:
   - Go to "Create a resource" > "Security + Identity" > "Key Vault".
   - Configure the Key Vault (name, resource group, region) and create it.

3. **Add Secrets**:
   - Navigate to the created Key Vault.
   - Go to "Secrets" and add new secrets.

### Step 2: Configure Azure DevOps to Use Azure Key Vault

1. **Service Connection in Azure DevOps**:
   - Go to "Project settings" > "Service connections".
   - Add a new service connection for Azure Resource Manager, selecting the appropriate subscription and Key Vault.

2. **Grant Access to Azure DevOps**:
   - In the Azure portal, navigate to your Key Vault.
   - Go to "Access policies" and add a new policy.
   - Assign necessary permissions (e.g., Get, List) to the Azure DevOps service principal.

### Step 3: Update Azure DevOps Pipeline to Retrieve Secrets

1. **Install Azure Key Vault Task**:
   - Ensure the Azure Key Vault task is available in your Azure DevOps pipeline.

2. **Retrieve Secrets in Pipeline**:
   - Update your pipeline YAML or classic pipeline to include tasks for retrieving secrets from Azure Key Vault.
   
   Example YAML snippet:
   ```yaml
   trigger:
   - main

   pool:
     vmImage: 'ubuntu-latest'

   variables:
     - group: 'KeyVaultSecrets'

   steps:
   - task: AzureKeyVault@2
     inputs:
       azureSubscription: '<Service Connection Name>'
       KeyVaultName: '<KeyVault Name>'
       SecretsFilter: '*'
       RunAsPreJob: true

   - script: echo $(mySecret)
     displayName: 'Display secret'
   ```

### Step 4: Use Retrieved Secrets in Pipeline Tasks
   - Use the retrieved secrets in subsequent tasks within your pipeline.
   
   Example:
   ```yaml
   steps:
   - script: |
       echo "Using secret $(mySecret) in build process"
     displayName: 'Use Secret in Build'
   ```

## Best Practices

1. **Least Privilege Principle**:
   - Grant the minimum necessary permissions to Azure DevOps and other applications.

2. **Automated Secret Rotation**:
   - Implement automated secret rotation to minimize the risk of using stale secrets.

3. **Monitoring and Auditing**:
   - Enable diagnostic logging for Azure Key Vault and monitor access patterns.

4. **Environment Segregation**:
   - Use separate Key Vaults for different environments (e.g., development, testing, production).

5. **Secure Access**:
   - Use managed identities or service principals with Azure AD for secure access to Azure Key Vault.

## Conclusion

Switching from Azure DevOps Secrets variables to Azure Key Vault provides enhanced security, scalability, and compliance for managing sensitive information. By following the steps and best practices outlined in this paper, organizations can implement a robust solution for securely managing secrets across their DevOps pipelines and applications. For further details and advanced configurations, refer to the [Azure Key Vault documentation](https://docs.microsoft.com/en-us/azure/key-vault/).




# Microsoft Graph API to read from Entra ID

# Prerequisites
'''pwsh
$tenantId = "<Your-Tenant-ID>"
$clientId = "<Your-Client-ID>"
$clientSecret = "<Your-Client-Secret>"
$resource = "https://graph.microsoft.com/"
$authority = "https://login.microsoftonline.com/$tenantId"

# Acquire Access Token
$body = @{
    grant_type    = "client_credentials"
    client_id     = $clientId
    client_secret = $clientSecret
    scope         = "https://graph.microsoft.com/.default"
}

$tokenResponse = Invoke-RestMethod -Method Post -Uri "$authority/oauth2/v2.0/token" -ContentType "application/x-www-form-urlencoded" -Body $body
$accessToken = $tokenResponse.access_token

# Function to make Graph API calls
function Invoke-GraphApi {
    param (
        [string]$uri,
        [string]$method = "GET",
        $body = $null
    )
    $headers = @{
        Authorization = "Bearer $accessToken"
    }
    return Invoke-RestMethod -Method $method -Uri $uri -Headers $headers -Body $body -ContentType "application/json"
}

# 1. List all users
$users = Invoke-GraphApi -uri "https://graph.microsoft.com/v1.0/users"
$users.value | ForEach-Object {
    [PSCustomObject]@{
        UserPrincipalName = $_.userPrincipalName
        DisplayName       = $_.displayName
        Email             = $_.mail
        JobTitle          = $_.jobTitle
        Department        = $_.department
    }
}

# 2. List all groups
$groups = Invoke-GraphApi -uri "https://graph.microsoft.com/v1.0/groups"
$groups.value | ForEach-Object {
    [PSCustomObject]@{
        GroupName = $_.displayName
        GroupId   = $_.id
        GroupType = $_.groupTypes
    }
}

# 3. List all roles assigned to a user
$userId = "<User-ID>" # Replace with the actual User ID
$userRoles = Invoke-GraphApi -uri "https://graph.microsoft.com/v1.0/users/$userId/appRoleAssignments"
$userRoles.value | ForEach-Object {
    [PSCustomObject]@{
        RoleName       = $_.resourceDisplayName
        RoleId         = $_.appRoleId
        AssignedDate   = $_.createdDateTime
        ResourceId     = $_.resourceId
    }
}
'''
