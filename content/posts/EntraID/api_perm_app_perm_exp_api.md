+++
date = '2026-09-27T21:31:30+02:00'
draft = true
title = 'API Permissions, Application permissions and Expose an API in Entra ID'
categories = ['Entra ID']
tags = ['api permissions', 'application permissions', 'expose api']
+++
## Intro

The purpose of this article is to explain and show the difference between **Application permissions, API permissions and Expose an API** in Entra ID. These terms are closely related, but understanding them is essential for securing apps in specific contexts. The article introduces problems, explains how these problems can be solved by using API Permissions, Application permissions and Expose API and demonstrates step by step test in Entra ID.

#### Terminology
A lot of confusion is caused by (almost) interchangeable terms, so let's set this straight:
- **API scopes vs API permissions**: API scopes are defined on the app that is exposing and API, they determine the actions that the app exposes through its API. API permissions are granted on the client app that will call the API, it declared "Which scopes of backend API is the client app permitted to call?"
- **App roles vs App permissions**: App roles are defined within the app in which they are used. App permissions are assignable roles that originate in backend app and can be assigned to client app. (Yes this is confusing because roles should be consisted of permissions, but this is the naming that Entra uses in this case)

![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/ev_1.png)
![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/ev_2.png)

## API permissions
#### Problem statement:
I have a resource owner (the user) who owns resources on a server. I have a client app that I want to allow to access those resources. Obviously, I cannot give my password to an unknown app and expect it to pull only the data I want from the server.

Instead, I want the app to access the resources on my behalf, while ensuring that it can access only the resources I explicitly authorize it to access.

#### Solution: 
OAuth 2.0 Authorization Code Flow. A trusted identity provider acts as a mediator between the user, the client application, and the resource server.

1. The client app redirects the user to the identity provider.

2. The user logs in and consents to a specific set of permissions (scopes).

3. The identity provider issues an access token to the client app with the granted scopes attached.

4. The client app requests data from the resource server using this token without requiring further user intervention.

5. The resource server trusts the identity provider, validates the token, and grants access strictly within the specified scopes.

#### Solution in Entra ID:
This scenario is handled using delegated permissions (also known as user-delegated permissions). The application acts on behalf of a signed-in user and is constrained by both the user's permissions and the requested scopes.

Permissions are created in the app representing the resource server (Expose an API → Add a scope) and are consented to on the client app (API permissions).

Consent Options:

- User Consent: Individual users grant the app permission to act on their behalf and access their own data.

- Admin Consent: An administrator grants delegated permissions on behalf of all users in the tenant. Users are not prompted individually.


## App Permissions: 
#### Problem statement:
I want Application A to request resources directly from Application B. Application A operates independently as a background service or daemon, without any active user involvement.


#### Solution: 
Applications A and B rely on a mutually trusted third-party identity provider to handle identity and authorization:

1. Application A authenticates directly with the identity provider using its own app credentials (client ID and secret/certificate).

2. The identity provider authenticates Application A and adds preconfigured roles for Application B to the token claims.

3. Application A presents the token to Application B.

4. Application B validates the token and grants access based on the roles/claims embedded in the token by the trusted third party.

#### Solution in Entra ID:
We use App Roles in Entra ID to solve this problem. We define an App Role on the app representing the resource server (App B). Then, we go to API permissions in the client app (App A) that should access the resources, where we will find the App Role under Application permissions. There, we can grant it.

Consent Requirement:

- Admin Consent Required: Because application permissions grant broad access across the tenant without user interaction, they must be consented to by a Tenant Administrator.



## Expose an API
#### Problem statement: 
I have a backend API application that exposes resources and business logic. Client applications authenticate users and make requests to my backend API on their behalf.

I am concerned about two major risks:

- Malicious or Rogue Clients: A user might log into an untrusted or compromised client app that executes unauthorized or malicious actions on their behalf.

- Over-Privileged Clients (Lack of Scope Boundaries): I do not want every client app to have unlimited access to every endpoint in my backend API. I need fine-grained boundaries defining exactly what each specific client application is allowed to request when acting on a user's behalf.


#### Solution: 
Rely on a mutually trusted Identity Provider (IdP) to handle user authentication, consent, and token issuance:
1. Expose Scopes on the Backend: The backend API defines fine-grained permissions (scopes, e.g., Files.Read, Reports.Write) within the IdP.

2. User Consent & Boundaries: When a user logs into a client app, the IdP displays the specific scopes requested by that client. The user must explicitly consent to these scopes.

3. Token Issuance: Upon consent, the IdP issues an Access Token directly to the client app. This token contains claims identifying the user, the authorized client app, and the specific consented scopes.

4. Backend Validation: When the client app calls the backend API, it presents the access token. The backend validates that:

    - The token was signed by the trusted IdP.

    - The token was intended for this API (aud / audience claim).

    - The token contains the specific scope required for the requested endpoint (scp claim).

This will allow us to control which apps are authorized to use which scopes on the backend API.


#### Solution in Entra ID
We can use Expose an API feature in Entra ID, where we will define the available scopes. We can also choose who is allowed to consent to the scopes on client app, you can select either `Admins and users` or `Admins only`. 

If you select `Admins only`, users cannot consent to the permission themselves. An administrator must grant consent before the client application can obtain a token containing that scope. If you select `Admins and users` users can consent when they log in for the first time. 

If admin consent is required, it must be granted on the client app.

If we have a required Entra ID license, we can also use Conditional Access Policies.

## DEMO

### Set up things in Entra
1. Go to app registrations in Entra ID and create `app-client` and `app-backend`
2. Within app registration/app-backend/Expose an API 
    - Add Application ID URI
    - Add Scope `ReadAllFiles`
![Adding scope](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/add_scope_expose_api.png)
    - Similarly we will add `ModifyAllFiles` scope
3. Go to App roles and click "+ Create app role"
4. Create an App role `FilesReader`
![Create app role](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/FilesReaderRole.png)
![see all roles](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/roles_backend.png)


5. Now go to `app-client`  within app registrations and go to API permissions
6. Click "Add a permission" then "APIs in my Organization" and search for `app-backend` and click it.
    - In delegated permissions, we have option to assign the scopes we created in `app-backend`
![Delegated permissions](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/del_perm.png)
    - In application permissions we have option to assign App roles we created in `app-backend`
![Applications permissions](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/app_perm.png)
7. Select as in the picture and click "Add permissions"
8. Click "Grant admin consent for Default Directory". 
    - This will allow us to use the permissions that require admin consent. 
![Applications permissions](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/grant_admin_consent.png)

### Test scenario: App permissions
```powershell
# 1. Setup Variables
$TenantId = "82d1ce97-862e-414f-9ea0-cf39ed243627"
$ClientId = "5cdaac62-6a56-48ca-a3cd-219f5e43fbe5"
$BackendAppUri = "api://fcfcf276-430e-48aa-9be6-2f499c31eb1b"

# 2. Request Token
$Body = @{
    client_id     = $ClientId
    client_secret = $Secret
    scope         = "$BackendAppUri/.default" # Must use /.default
    grant_type    = "client_credentials"
}

$Response = Invoke-RestMethod -Uri "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token" -Method Post -ContentType "application/x-www-form-urlencoded" -Body $Body

# 3. Output Access Token
$Response.access_token

```
- In the token received, we can see the role `Files.Reader` we created in `app-backend` is now in claims of jwt token issued for `app-client` (claim `appid`). 
![jwt](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/jwt_app.png)

### Test Scenario: Delegated permissions (API Scopes + Expose an API)
We will use a simple python code to set up a local server which simulates the client app.
```python
import base64
import hashlib
import os
import urllib.parse
import webbrowser
from http.server import BaseHTTPRequestHandler, HTTPServer

import requests  # Standard for HTTP requests

# 1. Configuration
TENANT_ID = "82d1ce97-862e-414f-9ea0-cf39ed243627"
CLIENT_ID = "5cdaac62-6a56-48ca-a3cd-219f5e43fbe5"
BACKEND_APP_ID = "api://fcfcf276-430e-48aa-9be6-2f499c31eb1b"
SCOPES = f"{BACKEND_APP_ID}/ReadAllFiles {BACKEND_APP_ID}/ModifyAllFiles"
REDIRECT_URI = "http://localhost:8400"
PORT = 8400

# Global variable to hold captured code
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
            self.send_response(400)
            self.end_headers()

    def log_message(self, format, *args):
        return  # Silence standard HTTP server logs


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
    print(f"1. Opening browser for Entra ID login...")
    webbrowser.open(auth_url)

    # Wait for browser to redirect back with the authorization code
    print("2. Waiting for login response on http://localhost:8400...")
    server.handle_request()  # Handles exactly 1 request and stops

    if auth_code:
        print("\n3. Code captured! Exchanging for Access Token...")

        # Exchange Authorization Code for Token
        token_url = (
            f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
        )
        data = {
            "client_id": CLIENT_ID,
            "grant_type": "authorization_code",
            "code": auth_code,
            "redirect_uri": REDIRECT_URI,
            "code_verifier": verifier,
            "scope": SCOPES,
        }

        response = requests.post(token_url, data=data)
        token_data = response.json()

        if "access_token" in token_data:
            print("\n================ ACCESS TOKEN ================")
            print(token_data["access_token"])
            print("==============================================")
        else:
            print("\nError fetching token:")
            print(token_data)
    else:
        print("\nFailed to capture authorization code.")
```
1) Add `http://localhost:8400` into allowed Redirect URI in Client App registration (the location where the Entra authorization server sends the user and delivers tokens once the user has successfully signed in)
2) Run the python code
    - After running the python code and signing in and consenting to permissions. We can see that the auth was successfull.
![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/tkn_1.png)

    - We can see that both delegated permissions are granted (`ReadAllFiles` as well as `ModifyAllFiles`). Eventhough we didn't explicitly add `ModifyAllFiles` into configured permissions, since it allowed "Admin and users" consent, user could grant consent without admin. This is an important observation, because whenever you create a scope in "Expose an API" - if you select "Who can consent? - Admins and users" you're basically delegating all the control to a user.

3) Now we will see what happens if only admin can grant consent to the permissions. For the demonstration, we I will create one more scope: `ReadConfidentialFiles`, where I will pick option "Admins only" on who can consent.
![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/extra_permission.png)

4) Let's keep things as they are in the portal (do not grant the permission yet) and add the scope `ReadIdentityFiles` to `SCOPES` in out python code:
![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/add_scope.png)

5) Now let's run the code and log in - this will be the result:
![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/need_admin_approval.png)
    - As expected, we are not issued a token because the required permission was not consented by the admin.


6) Now I will grant admin permission on client for `	
ReadConfidentialFiles` and try to run the python code again, this time a token with all 3 scopes is issued.

    ![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/grant_consent.png)

    ![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/tkn_2.png)

## Summary: 
### API Scopes vs App Roles vs Expose an API
- **API Scopes**: Used for delegated permissions - client app A requests info from app B on behalf of the user signed into the app A.
- **Expose an API**: Definition of Scopes in app B, those scopes will then be used by client apps.
- **App Roles**: Used for users/services that directly authenticate against target application. 

![](/images/api_permissions_application_permissions_and_expose_api_in_entra_id/diagram.png)

#### Why are scopes necessary, why can't we just assign roles?
If we used only roles: The token sent to the API would say "roles": ["Admin"]. The reporting app now has full Admin rights over your entire system—even if it only needs to draw a pie chart. If that app is malicious or compromised, it could wipe the database using your admin token. The user does not have control over the client app's requests. Roles are used when a user or service authenticates directly to the target app and does not act on behalf of someone else - the service or user has full control of their actions without a middleman involved.

#### When should you use these features?
Use API permissions, App roles and Expose an API **if you have multiple apps that need to communicate together and authenticate each other**, you can offload the logic of authentication and authorization into Entra ID.
