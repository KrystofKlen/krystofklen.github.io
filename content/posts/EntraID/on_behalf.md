+++
date = '2026-10-02T22:02:52+02:00'
draft = false
title = 'On On-Behalf-Of (OBO) flow for agents'
categories = ['Entra ID','AI']
tags = ['agent impersonating user','ai agent', 'agent identity']
+++
## What is On-Behalf-Of flow for agents
An On-Behalf-Of flow means an AI agent performs an action for a specific user, using permissions delegated to it by that user.

The agent temporarily acts as that user when interacting with another system.

For example, an AI assistant could send an email on your behalf, where you remain the owner of the action and permissions. 

The agent does **not** have to inherit all of the user’s permissions - it can be restricted and scoped to only the specific activities it is authorized to perform.


## Usage
### When to use
- Use the On-Behalf-Of flow when an AI agent must perform actions using an existing user's delegated identity and permissions. Choose this model only when the operation strictly requires the user's explicit authorization and identity, and alternative patterns, such as a fully autonomous agent identity or a dedicated agent user account, are unsuitable.

### 

### When NOT to use
- The system on which the action is performed cannot distinguish between direct user actions and an agent acting on behalf of a user.
- The agent needs to perform actions as a user, but those actions do not need to be associated with an individual human user. In such cases, the agent should use its own dedicated user account to perform the actions, rather than using a personal user account.


## Theory
![](/images/on-behalf-of-for-agent/flow.png)
1. The user authenticates against Entra ID using the client app, Entra returns user access token (Tc)
    - The token must be scoped to the Agent Blueprint.
2. The user sends the TC to the agent identity blueprint.
    - This basically means that the client app will send the token to another app, which handles the blueprint authentication.
3. The agent identity blueprint authenticates against Entra ID. Entra ID returns the T1 exchange token.
4. Now the agent identity sends T1 + Tc to Entra ID.
5. Entra ID validates both tokens, and if everything is correct, it returns a resource access token (TR).
    - Agent identity must have granted API permissions, so that the token has the scope it needs to be granted access to resources in other services.
6. The access token (TR) is used by the agent identity to access the resource.
## DEMO
Goal: Let the agent to act on behalf of user, delete a file in blob storage and see the action in logs.

### Prerequisites in Entra
- Created client app `client-app-test` in app registrations.
- Created blueprint `blueprint_1` and agent identity `agent-1`.

### Python code simulating client app
```python
import base64
import hashlib
import os
import urllib.parse
import webbrowser
from http.server import BaseHTTPRequestHandler, HTTPServer
import requests

# 1. Configuration
TENANT_ID = "82d1ce97-862e-414f-9ea0-cf39ed243627"
CLIENT_ID = "ba282b40-b558-4fc7-8aac-d9ca4031422c"
REDIRECT_URI = "http://localhost:8400"
PORT = 8400
BLUEPRINT_APP_ID = "8e73f354-6cca-40fb-ac0a-7b61d7c64bcf"

# Basic scopes for user login
SCOPES = f"{BLUEPRINT_APP_ID}/Blob.Delete"

auth_code = None


# 2. PKCE Helper Functions
def generate_pkce():
    code_verifier = (
        base64.urlsafe_b64encode(os.urandom(32)).decode("utf-8").rstrip("=")
    )
    code_challenge = (
        base64.urlsafe_b64encode(
            hashlib.sha256(code_verifier.encode("utf-8")).digest()
        )
        .decode("utf-8")
        .rstrip("=")
    )
    return code_verifier, code_challenge


# 3. Local Server Handler to Catch Redirect
class OAuthHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        global auth_code
        parsed_url = urllib.parse.urlparse(self.path)
        query_params = urllib.parse.parse_qs(parsed_url.query)

        if "code" in query_params:
            auth_code = query_params["code"][0]
            self.send_response(200)
            self.send_header("Content-type", "text/html")
            self.end_headers()
            self.wfile.write(
                b"<html><body><h2>Authorization successful!</h2><p>You can close this tab now.</p></body></html>"
            )
        else:
            # Capture error details sent back in URL
            error_msg = query_params.get("error_description", ["Unknown error"])[0]
            print(f"\n[LOGIN ERROR]: {error_msg}")
            
            self.send_response(400)
            self.send_header("Content-type", "text/html")
            self.end_headers()
            self.wfile.write(
                f"<html><body><h2>Login Error</h2><p>{error_msg}</p></body></html>".encode("utf-8")
            )

    def log_message(self, format, *args):
        return


# --- MAIN EXECUTION ---
if __name__ == "__main__":
    verifier, challenge = generate_pkce()

    # Build Entra ID Login URL
    params = {
        "client_id": CLIENT_ID,
        "response_type": "code",
        "redirect_uri": REDIRECT_URI,
        "scope": SCOPES,
        "code_challenge": challenge,
        "code_challenge_method": "S256",
    }
    auth_url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/authorize?{urllib.parse.urlencode(params)}"

    # Start Local Web Server
    server = HTTPServer(("localhost", PORT), OAuthHandler)
    print("1. Opening browser for Entra ID login...")
    webbrowser.open(auth_url)

    # Wait for browser to redirect back with the authorization code
    print(f"2. Waiting for login response on {REDIRECT_URI}...")
    server.handle_request()  # Handles exactly 1 request and stops

    if auth_code:
        print("\n3. Code captured! Exchanging for Access Token...")

        # Exchange Authorization Code for Token
        token_url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
        
        payload = {
            "client_id": CLIENT_ID,
            "grant_type": "authorization_code",
            "code": auth_code,
            "redirect_uri": REDIRECT_URI,
            "code_verifier": verifier,
            "scope": SCOPES,
        }

        # Explicit URL-encoding & Content-Type header fix
        headers = {"Content-Type": "application/x-www-form-urlencoded"}
        encoded_payload = urllib.parse.urlencode(payload)

        response = requests.post(token_url, data=encoded_payload, headers=headers)
        token_data = response.json()

        if "access_token" in token_data:
            print("\n================ ACCESS TOKEN ================")
            print(token_data["access_token"])
            print("==============================================")
        elif "id_token" in token_data:
            print("\n================ ID TOKEN ================")
            print(token_data["id_token"])
            print("==========================================")
        else:
            print("\nError fetching token:")
            print(token_data)
    else:
        print("\nFailed to capture authorization code.")
```        

### DEMO Steps

1) When a user logs in to the client app, Entra issues a token with an audience. According to Entra documentation, this audience must be the App ID of the blueprint. We will achieve this by granting the client app permission to call the "blueprint's API" on the user's behalf. That's why we need to add OAuth2.0 permission scopes on blueprint as the first step. We will add the following into blueprint's manifest and save it.
    ```json
    "oauth2PermissionScopes": [
        {
            "adminConsentDescription": "Allows the app to delete blobs in Azure Storage on behalf of the signed-in user.",
            "adminConsentDisplayName": "Delete Storage Blobs",
            "id": "3f9c0e21-8214-41e9-921a-429a28e82110",
            "isEnabled": true,
            "type": "User",
            "userConsentDescription": "Allows the app to delete files in storage on your behalf.",
            "userConsentDisplayName": "Delete Storage Files",
            "value": "Blob.Delete"
        }
    ]
    ```
    ![](/images/on-behalf-of-for-agent/blueprint_expose_api.png)

2) Now we will grant the permission on the client app
    ![](/images/on-behalf-of-for-agent/add_blob_delete_api_perm_client_app.png)

3) In the Python code used for the user to sign in to the client app, we will set the scope: 
    ```python
    SCOPES = f"{BLUEPRINT_APP_ID}/Blob.Delete"
    ```

4) Add `http://localhost:8400` to redirect URI in `client-app-test` (This represents the URI on which our python code sets up a simple server simulating the client app to which user logs in. Entra needs to know this URI so that it knows where to redirect user after successful login)

5) Run the Python code and log in. You are prompted to approve the on-behalf permission for the client app.
    ![](/images/on-behalf-of-for-agent/log_in.png)

    The program prints JWT to the console, copy the JWT to view the claims (for example by using `www.jwt.io`)
    ![](/images/on-behalf-of-for-agent/jwt_1.png)

    There are two important claims to note:
    - `aud`: The audience of the token is the Agent Blueprint. This fulfills Microsoft's requirement to have the User's access token (Tc) scoped to the Agent Blueprint. 
    - `azp`: The authorized party is the id of the client app.

    We achieved this by granting API permissions to the client app. By granting these permissions, Entra knows that the token will be used by the client app to act on behalf of the user and that it will be sent to the blueprint.

6) Get token T1 (blueprint authentication)

    ```powershell
    $T1Body = @{
        client_id     = $BlueprintClientId
        client_secret = $BlueprintSecret # For testing purposes, we use client secret to authenticate blueprint
        scope         = "api://AzureADTokenExchange/.default"
        fmi_path      = $AgentIdentityAppId # Agent identity to which access token to resources (in our case Blob storage) will be issued
        grant_type    = "client_credentials" 
    }

    $T1Response = Invoke-RestMethod -Uri $TokenEndpoint `
            -Method Post `
            -ContentType "application/x-www-form-urlencoded" `
            -Body $T1Body

    $T1 = $T1Response.access_token
    ```

7) Before getting TR, we need to grant the `user_impersonation` permission on the agent identity.

    ```powershell
    $AgentSP   = Get-MgServicePrincipal -Filter "appId eq '763f1591-5e15-4d11-b10b-b4668ec41667'"
    $StorageSP = Get-MgServicePrincipal -Filter "displayName eq 'Azure Storage'"
    New-MgOAuth2PermissionGrant -BodyParameter @{
            ClientId    = $AgentSP.Id
            ResourceId  = $StorageSP.Id
            Scope       = "user_impersonation"
            ConsentType = "AllPrincipals"
        }
    ```
    - We can see that the delegated permission was added, now the agent can act on the user's behalf when accessing the Azure Storage API.
    ![](/images/on-behalf-of-for-agent/deleg_perm.png)

8) Let's get TR:

    ```powershell
    $TRBody = @{
        client_id             = $AgentIdentityAppId
        scope                 = "https://storage.azure.com/user_impersonation"
        grant_type            = "urn:ietf:params:oauth:grant-type:jwt-bearer"
        client_assertion_type = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
        client_assertion      = $T1
        assertion             = $Tc
        requested_token_use   = "on_behalf_of"
    }

    $TRResponse = Invoke-RestMethod -Uri $TokenEndpoint `
            -Method Post `
            -ContentType "application/x-www-form-urlencoded" `
            -Body $TRBody

    $TR = $TRResponse.access_token
    ```
    We can note a few important claims:
    ![](/images/on-behalf-of-for-agent/tkn_tr.png)
    - `aud`: As expected (following the previous step where we added API permissions for Azure Storage), this is scoped to `storage.azure.com`, which is the intended receiver of the token.
    - `appid`: Object ID of the Agent Identity. Represents the app (in this case agent identity) that claimed and obtained the token.
    - `idtyp`: Stands for Identity Type. It specifies whether the token was issued for a user context (delegated access) or an app-only context (service-to-service)
    - `oid`: Object ID of the subject. In this case the Object ID of the user.
    - `scp`: Scope we're allowed for the audience. This comes from the previously configured API permissions.

    We can see that the scope we configured in Blueprint technically has no effect (because the `Blob.Delete` is not part of TR token claims). It was just to enable us to have audience of Tc pointed to the Blueprint.
    We can also see that the token itself has enough info to determine that agent is impersonating a user.

9) Assign the appropriate Azure RBAC role to the user impersonated by the agent.
![](/images/on-behalf-of-for-agent/role_assignment.png)

10) Let's test the file delete
![](/images/on-behalf-of-for-agent/test_blob.png)

    ```powershell
    # 1. Target Blob Details
    $StorageAccount = "rgphishing9aa0"
    $Container      = "test"
    $BlobName       = "test_file.txt"

    # 2. Construct Headers for Azure Storage REST API
    $Headers = @{
        "Authorization" = "Bearer $TR"
        "x-ms-version"   = "2023-11-03" # Azure Storage API version
    }

    # 3. Storage REST API Endpoint
    $BlobUri = "https://$StorageAccount.blob.core.windows.net/$Container/$BlobName"

    # 4. Execute Delete Request
    $Response = Invoke-RestMethod -Uri $BlobUri `
                                  -Method Delete `
                                  -Headers $Headers `
                                  -StatusCodeVariable "StatusCode"
    ```
    ![](/images/on-behalf-of-for-agent/blob_deleted.png)
    As expected, the file `test_file.txt` was deleted.


11) See logs
    - Deleted by the agent.
    ![](/images/on-behalf-of-for-agent/logs_blob_obo.png)
    - Deleted by user directly in Azure Portal.
    ![](/images/on-behalf-of-for-agent/logs_direct_azure_action.png)

    | Audit Log Field | On-Behalf-Of (OBO) Flow (`agent-1`) | Direct Azure Portal Action |
    | :--- | :--- | :--- |
    | **`RequesterAppId`** | `763f1591-5e15-4d11-b10b-b4668ec41667` (`agent-1`) | `691458b9-1327-4635-9f55-ed83a7f1b41c` (Azure Portal) |
    | **`RequesterUpn`** | `testuser1@krystofklenoutlook.onmicrosoft.com` | `testuser1@krystofklenoutlook.onmicrosoft.com` |
    | **`RequesterObjectId`** | `b9cb6e69-6a7a-4aa0-8f52-2a9badadaef9` (User GUID) | `b9cb6e69-6a7a-4aa0-8f52-2a9badadaef9` (User GUID) |
    | **`AuthenticationType`** | `OAuth` | `OAuth` |
    | **`RequesterAudience`** | `https://storage.azure.com` | `https://storage.azure.com` |
    | **`AuthorizationDetails`** | Delegated User RBAC (`b9cb6e69...`) | Delegated User RBAC (`b9cb6e69...`) |
    | **Key Takeaway** | Proves programmatic execution by `agent-1` acting on behalf of the user. | Proves interactive execution directly by the user within the Azure Portal UI. |
    
