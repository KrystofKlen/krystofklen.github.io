+++
date = '2026-10-30T20:15:49+01:00'
draft = false
title = "Agent's user account in Entra ID"
categories = ['Entra ID', 'AI Agents']
tags = ['agent impersonating user','ai agent', 'agent identity']
+++

## What is agent's user account
An agent user account is an account used by an AI agent or automated system to access a website, application, or service where user-like interaction is required.

### When to use it
- When you want an AI agent to behave like a digital employee, with its own identity in everyday business collaboration (callendar, mailbox, teams, etc.).
- When an agent requires access to services that rely on a user identity and at the same time don't necessarily need to use a real user identity.

### When not to use it
- When an agent only needs to access APIs, databases or backend services (use agent identity instead).
- When an agent performs background automation without user interaction (use agent identity instead).
- When an agent only needs application permissions to access resources (use agent identity or application identity, depending on how the agent is implemented).
- When an agent acts on behalf of an existing employee using delegated permissions (use the OBO flow instead).

### Agent's user account vs On-behaf-of (OBO) flow
With the OBO flow, the agent acts on behalf of a real user. Depending on the downstream application, it may not be possible to distinguish whether an action was performed by the actual human user or by an agent using that user's identity.

With an agent user account, a dedicated account is created specifically for the agent. The agent has access to user-level capabilities, but it is always clear that the account is dedicated to that particular agent and is not associated with a real human user.

### Agent's user account vs Agent Identity
An agent identity is the base identity for an agent in Entra ID. An agent user account builds on top of an existing agent identity. Each agent identity can have at most one associated agent user account, and each agent user account is linked to exactly one parent agent identity.

- Use an agent identity when a non-human identity is required. 
- Use an agent user account when a user identity is required and the action does not necessarily need to be performed by a real, existing user.

### Technicalities to be aware of
- An agent user account cannot have credentials such as passwords or passkeys. The only supported credential type is a reference to its parent agent identity.
- It is not created automatically. An administrator must explicitly create the account and associate it with an existing agent identity.
- For security reasons, an agent user account cannot be assigned privileged administrator roles by design.
- An agent user account requires a license, just like a regular user account, to access Microsoft 365 services.
- If you grant permissions at the blueprint level, they are inherited by all child agents.


## Technical demo
We will create agent's user account and use it to write to blob storage.

### 1. Add permission to the blueprint
We need to add permission `AgentIdUser.ReadWrite.IdentityParentedBy` to the agent blueprint. This is because agent's user account is created by blueprint.
```powershell
$tenantId = "82d1ce97-862e-414f-9ea0-cf39ed243627"
$blueprintAppId = "8e73f354-6cca-40fb-ac0a-7b61d7c64bcf"
$blueprintPrincipalId = "12b0fefc-6331-45a2-89d9-d476963db100"
$permission = "AgentIdUser.ReadWrite.IdentityParentedBy"

Connect-Entra -TenantId $tenantId

Add-EntraPermissionToCreateAgentUsersToAgentIdentityBlueprintPrincipal `
    -AgentBlueprintId $blueprintAppId
    
```

### 2. Create agent user
We will login as blueprint and create agent's user Account for pre-selected agent identity that has the same blueprint which we're currently using to authenticate.
```powershell
Connect-MgGraph `
    -TenantId $tenantId `
    -Scopes "AgentIdUser.ReadWrite.IdentityParentedBy" `
    -ContextScope Process

# App ID of the parent agent
$identityParentId = "763f1591-5e15-4d11-b10b-b4668ec41667"

$headers = @{
    Authorization = "Bearer $AccessToken" # Access token for the blueprint
    "Content-Type" = "application/json"
}

$body = @{
    "@odata.type"      = "#microsoft.graph.agentUser"
    accountEnabled     = $true
    displayName        = "user-agent-1"
    mailNickname       = "user-agent-1"
    userPrincipalName  = "user-agent-1@krystofklenoutlook.onmicrosoft.com"
    identityParentId   = $identityParentId
} | ConvertTo-Json

$response = Invoke-RestMethod `
    -Method Post `
    -Uri "https://graph.microsoft.com/v1.0/users/microsoft.graph.agentUser" `
    -Headers $headers `
    -Body $body

$response | ConvertTo-Json -Depth 10

```

![user agent created](/images/agents_user_account/user_agent_1.png)

### 3. Grant user permissions
Agent's user account is used exclusively by it's parent agent identity. In order for agent identity to act on behalf of the user account, we need to grant user permissions on the agent identity.

We can also grant the consent on the blueprint level. In that case all child agents and their user accounts would automatically have the permissions granted.
```powershell
$graphSp = Get-MgServicePrincipal `
    -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

New-MgOauth2PermissionGrant -BodyParameter @{
    clientId    = $AgentIdentityAppId        # Agent Identity SERVICE PRINCIPAL object ID
    consentType = "Principal"
    principalId = $AgentUserObjectId        # Agent User object ID
    resourceId  = $graphSp.Id          # Microsoft Graph service principal object ID
    scope       = "User.Read Mail.Read Calendars.Read"
}
```
We can see the permissions in the Entra portal.
![](/images/agents_user_account/user_permissions_1.png)

### 4. Sign in as Agent User

The authentication flow for agent's user account happens as follows:
![](/images/agents_user_account/auth_flow_agent_user.png)
![](/images/agents_user_account/agent_user_auth_flow.png)
```powershell
$AgentBlueprintAppId = "8e73f354-6cca-40fb-ac0a-7b61d7c64bcf"
$AgentIdentityAppId = "763f1591-5e15-4d11-b10b-b4668ec41667"
$AgentUserObjectId = "a79a2890-a1cb-4b33-878d-7aa77ee58055"
$TokenEndpoint = "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token"

# Request to get T1
$body1 = @{
    client_id     = $AgentBlueprintAppId
    scope         = "api://AzureADTokenExchange/.default"
    grant_type    = "client_credentials"
    client_secret = $ClientSecret

    # Tells Entra which Agent Identity the blueprint is acting for
    fmi_path      = $AgentIdentityAppId
}

$response1 = Invoke-RestMethod `
    -Method POST `
    -Uri $TokenEndpoint `
    -ContentType "application/x-www-form-urlencoded" `
    -Body $body1

$T1 = $response1.access_token

# Request to get T2, Agent Identity uses the token acquired from it's parent blueprint.
$body2 = @{
    client_id             = $AgentIdentityAppId
    scope                 = "api://AzureADTokenExchange/.default"
    grant_type            = "client_credentials"
    client_assertion_type = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
    client_assertion      = $T1
}

$response2 = Invoke-RestMethod `
    -Method POST `
    -Uri $TokenEndpoint `
    -ContentType "application/x-www-form-urlencoded" `
    -Body $body2

$T2 = $response2.access_token

# Request to get TR, Agent Identity uses both T1 and T2
$body3 = @{
    client_id                          = $AgentIdentityAppId
    scope                              = "https://graph.microsoft.com/.default"
    client_assertion_type              = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
    client_assertion                   = $T1
    user_federated_identity_credential = $T2
    username                            = "user-agent-1@krystofklenoutlook.onmicrosoft.com"
    grant_type                          = "user_fic"
    requested_token_use                 = "on_behalf_of"
}

$response3 = Invoke-RestMethod `
    -Method POST `
    -Uri $TokenEndpoint `
    -ContentType "application/x-www-form-urlencoded" `
    -Body $body3

$AgentUserAccessToken = $response3.access_token
```



### 5. Notable token claims
- `scp`: Scope of the token - these are the permissions we granted on he Agent Identity.
- `idtyp`: This is different to agent identity token, where we have `idtyp=app`, it allows the service receiving the toke to clearly distinguish between app request made by agent (`idtyp=app`) and request made by agent impersonating the user account (`idtyp=user`) .
- `oid`: Object id of the user represented by the token - this is the Object ID of Agent's User account we've created.

![](/images/agents_user_account/token_agent_user.png)

### 6. Storage account read
To demonstrate the usage of Agent's User account, we will read file content from a file stored in azure blob storage.

We will assign the Azure RBAC role on the Blob Storage first. We can treat Agent's User account as a user when assigning Azure RBAC roles.
![](/images/agents_user_account/adding_azure_RBAC_role.png)


After the role is assigned, we need to grant the user permissions. Agent Identity will impersonate it's child Agent's User account to access the Blob Storage. This impersonation can not happen without user (the Agent's User account in this case) granting the permission, therefore we need to grant `user_impersonation` permission on Agent Identity for the Agent's User account. 
```powershell
$StorageSp = Get-MgServicePrincipal -Filter "displayName eq 'Azure Storage'"
$StorageSpObjectId = $StorageSp.Id

New-MgOauth2PermissionGrant -BodyParameter @{
    clientId    = $AgentIdentityAppId
    consentType = "Principal"
    principalId = $AgentUserObjectId
    resourceId  = $StorageSpObjectId
    scope       = "user_impersonation"
}
```
The granted permission is visible in the Entra ID portal. It allows the agent identity to impersonate its child agent user accounts and access Blob Storage on their behalf.

![](/images/agents_user_account/user_permissions_2.png)

We will use the same authentication flow as before. However, this time we need to change the token scope to target Blob Storage, as it is the service we want to access.


```powershell
$body3 = @{
    client_id                          = $AgentIdentityAppId
    scope                              = "https://storage.azure.com/.default"
    client_assertion_type              = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
    client_assertion                   = $T1
    user_federated_identity_credential = $T2
    username                            = "user-agent-1@krystofklenoutlook.onmicrosoft.com"
    grant_type                          = "user_fic"
    requested_token_use                 = "on_behalf_of"
}

$response3 = Invoke-RestMethod `
    -Method POST `
    -Uri $TokenEndpoint `
    -ContentType "application/x-www-form-urlencoded" `
    -Body $body3

$AgentUserAccessToken = $response3.access_token
```


Now we'll read from the Blob Storage.
```powershell
$storageAccount = "rgphishing9aa0"
$container = "test"
$blobName = "test_file.txt"

$uri = "https://$storageAccount.blob.core.windows.net/$container/$blobName"

$headers = @{
    Authorization  = "Bearer $AgentUserAccessToken"
    "x-ms-version" = "2023-11-03"
}

$response = Invoke-WebRequest `
    -Method Get `
    -Uri $uri `
    -Headers $headers

$response.Content
```
![](/images/agents_user_account/read_content.png)




