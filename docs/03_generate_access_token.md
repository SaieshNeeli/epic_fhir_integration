# Step 3 — Generate JWT and Request Access Token

After registering the application and uploading the public key, the backend must authenticate with Epic using the OAuth 2.0 Backend Services flow.

In this flow, the backend generates a signed JWT using the private RSA key created in Step 1.  
This JWT is sent to the Epic OAuth token endpoint to obtain an access token.

---

## OAuth Token Endpoint

https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token

---

## JWT Claims Required

The JWT must contain the following claims.

| Claim | Description |
|------|-------------|
| iss | Client ID from Epic |
| sub | Client ID |
| aud | Epic token endpoint |
| jti | Unique identifier (UUID) |
| exp | Expiration timestamp |

Example JWT payload:

{
 "iss": "CLIENT_ID",
 "sub": "CLIENT_ID",
 "aud": "https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token",
 "jti": "random-uuid",
 "exp": 1710000000
}

The expiration time should be within **5 minutes**.


## Python Example

The following script generates a signed JWT and exchanges it for an access token.

import jwt
import time
import uuid
import requests
import os

client_id = "YOUR_CLIENT_ID"
aud = "https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token"

payload = {
    "iss": client_id,
    "sub": client_id,
    "aud": aud,
    "jti": str(uuid.uuid4()),
    "exp": int(time.time()) + 300
}

with open("private_key.pem", "r") as f:
    private_key = f.read()

assertion = jwt.encode(payload, private_key, algorithm="RS384")

data = {
    "grant_type": "client_credentials",
    "client_assertion_type":
    "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": assertion,
    "scope": "system/*.read"
}

response = requests.post(aud, data=data)

print(response.status_code)
print(response.json())
