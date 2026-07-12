+++
date = '2026-07-12T13:59:41+02:00'
draft = true
title = 'Default Azure Credential'
categories = ['azure']
tags = ['credential', 'Azure', 'authentication', 'identity']
+++
# Default azure credential

Generally, there are 3 ways to authenticate to azure:
1. Environment variables: Ideal on application running on premises.
2. Managed identity: Ideal for application running in azure cloud.
3. Development tools: If you're signed in to Azure through Visual Studio, Visual Studio code, Azure CLI or Azure Powershell.

### Examples
#### 1. Environment variables: Service principal.
Using environment variables with a Service Principal is a traditional approach for authenticating applications with Azure services. The application uses a registered Entra ID application (Service Principal) together with its client ID and client secret to obtain an access token.

As long as the Service Principal has the required permissions (for example, permission to read secrets from Azure Key Vault), the application can successfully authenticate and retrieve the required resources.

However, this approach requires the necessary environment variables to be configured in the execution environment. If any required values are missing or incorrect, authentication will fail and the application will not be able to access the Key Vault.

```python
# Load .env
load_dotenv()

tenant_id = os.environ["AZURE_TENANT_ID"]
client_id = os.environ["AZURE_CLIENT_ID"]
client_secret = os.environ["AZURE_CLIENT_SECRET"]

# Create the credential
credential = ClientSecretCredential(
    tenant_id=tenant_id,
    client_id=client_id,
    client_secret=client_secret,
)
```
#### 2. Managed identity: Ideal for application running in azure cloud.
Managed Identity is the recommended authentication approach for applications that are deployed and running within Azure. It allows the application to authenticate to Azure services without requiring developers to manage credentials.

However, a limitation of Managed Identity is that it is only available within supported Azure environments. This can make local development and testing more challenging, as the same authentication method cannot be used directly when running the code on a developer machine.


```python
# Authenticate using the Container Instance's managed identity
credential = ManagedIdentityCredential()

# Create the Key Vault client
client = SecretClient(
    vault_url=vault_url,
    credential=credential,
)
```
#### 3. Development tools: Azure CLI
For local development, Azure CLI authentication can be used to provide credentials to the application. After signing in with Azure CLI, the application retrieves the authenticated user credentials through `AzureCliCredential` and uses them to securely access Azure resources.

```python
# Authenticate using the Azure CLI
credential = AzureCliCredential()

# Connect to the Key Vault
vault_url = "https://test-vault-2312.vault.azure.net/"

client = SecretClient(
    vault_url=vault_url,
    credential=credential,
)
```

```
az login
```

## Default Credentials

#### Problem it solves
Azure provides multiple authentication methods depending on where an application is running. For example, an application running locally may use Azure CLI credentials, while an application deployed to Azure may use Managed Identity.

Managing these different authentication methods separately can lead to environment-specific code changes and additional configuration complexity.

#### How it solves the problem

`DefaultAzureCredential` automatically attempts multiple credential types in a predefined order. The application uses the first available credential that can successfully authenticate.

The authentication chain is evaluated in the following order:
1. Environment Credential
2. WorkloadIdentityCredential
3. ManagedIdentityCredential
4. VisualStudioCredential
5. VisualStudioCodeCredential
6. AzureCliCredential
7. AzurePowerShellCredential
8. AzureDeveloperCLICredential
9. InteractiveBrowserCredential

- if credential is found: It tries to authenticate using that credential
- if credential is not found: It tries the next authentication method in the specified order

This allows the same application code to run across different environments (local development, CI/CD pipelines, and Azure-hosted environments) without changing the authentication implementation.

```python
credential = DefaultAzureCredential()

client = SecretClient(
    vault_url="https://test-vault-2312.vault.azure.net/",
    credential=credential,
)
```

### Examples
#### Default azure credential - logged in azure cli
From the logs, we can see the authentication methods attempted. `DefaultAzureCredential` successfully retrieved credentials from Azure CLI and used them for authentication. Since my account has the required **Key Vault Administrator** role, the secret value was successfully retrieved.

```
INFO:azure.identity._credentials.environment:No environment configuration found.
...
INFO:azure.identity._credentials.managed_identity:ManagedIdentityCredential will use IMDS
...
INFO:azure.core.pipeline.policies.http_logging_policy:Response status: 401
...
INFO:azure.identity._credentials.chained:DefaultAzureCredential acquired a token from AzureCliCredential
...
INFO:azure.core.pipeline.policies.http_logging_policy:Response status: 200
...
secret abcd
```

#### Environment variables - service principal
First, we need to export the necessary env variables:

```
export AZURE_TENANT_ID=82d1ce97-862e-414f-9ea0-cf39ed243627
export AZURE_CLIENT_ID=f08606de-2a9f-4e55-a7d8-b6c996df17a1
export AZURE_CLIENT_SECRET=<client-secret>
```

Default Credentials will detect those values and will try to authenticate, since the app's service principal has **Key Vault Secrets User** role assigned to the scope of the secret, it will retrieve the secret value

```
INFO:azure.identity._credentials.environment:Environment is configured for ClientSecretCredential
...
INFO:azure.core.pipeline.policies.http_logging_policy:Response status: 200
...
secret abcd
```

If the role assignment is removed, the authentication flow remains the same. The credential chain stops at the first available credential (environment variables representing a Service Principal). After successful authentication, the request is denied by Azure RBAC due to insufficient permissions.

```
INFO:azure.identity._credentials.environment:Environment is configured for ClientSecretCredential
...
INFO:azure.core.pipeline.policies.http_logging_policy:Response status: 200
...
azure.core.exceptions.HttpResponseError: (Forbidden) Caller is not authorized to perform action on resource.
...
Inner error: {
    "code": "ForbiddenByRbac"
}
```

#### Managed Identity

In this example, I created a container image and deployed it to Azure Container Instances. The container is configured with a system-assigned managed identity, which has the **Key Vault Secrets User** role assigned with the required scope.

![Azure Key Vault: RBAC assignment](/images/default-credentials/role_assignment.png)

After starting the container, Log Analytics shows that `DefaultAzureCredential` successfully obtained credentials through **Managed Identity** and used them to authenticate. The application was then able to successfully retrieve the secret from Azure Key Vault.

![Log Analytics: Managed identity credentials obtained](/images/default-credentials/managed_identity_token_acquired.png)

![Log Analytics: Credential obtained](/images/default-credentials/credential_retrieval.png)
