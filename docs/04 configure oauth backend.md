# 04 — Configure OAuth 2.0 Backend Services

This document covers the full OAuth 2.0 **Client Credentials** flow used by Epic's Backend Services authentication.  
No user login is involved — your backend signs a JWT, exchanges it for an access token, and uses that token to call FHIR APIs.

> 📖 **Epic Reference:**  
> `https://fhir.epic.com/Documentation?docId=oauth2&section=BackendOAuth2Guide`

---

## How the Flow Works

```
Your Backend                        Epic Token Endpoint
     |                                      |
     |  1. Build JWT payload                |
     |  2. Sign JWT with private_key.pem    |
     |                                      |
     |--- POST /oauth2/token -------------->|
     |    grant_type=client_credentials     |
     |    client_assertion=<signed_jwt>     |
     |                                      |
     |<-- { access_token, expires_in } -----|
     |                                      |
     |--- GET /FHIR/R4/Patient ------------>|
     |    Authorization: Bearer <token>     |
     |                                      |
     |<-- FHIR Resource JSON ---------------|
```

---

## What You Need Before Starting

| Item | Where to Get It |
|------|----------------|
| `private_key.pem` | Generated in Step 3 |
| `public_key.pem` | Uploaded to Epic in Step 2 |
| `EPIC_CLIENT_ID` | Provided by Epic after app registration |
| Token endpoint URL | Listed below per environment |

---

## Endpoints

| Environment | Token Endpoint |
|-------------|---------------|
| **Sandbox** | `https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token` |
| **Production** | `https://<customer-domain>/oauth2/token` *(varies per Epic customer)* |

> ⚠️ Production token endpoints are **not** the same as the sandbox.  
> Each Epic customer organisation has its own unique URL. Obtain this from the Epic customer's IT team.

---

## Step 1 — Build the JWT Payload

Your JWT must contain exactly these claims:

| Claim | Value | Notes |
|-------|-------|-------|
| `iss` | Your Client ID | Identifies your app |
| `sub` | Your Client ID | Same as `iss` for backend apps |
| `aud` | Token endpoint URL | Full URL of the token endpoint |
| `jti` | Unique UUID | One-time use ID — prevents token replay |
| `exp` | `now + 300` seconds | Max 5 minutes in the future |
| `nbf` | `now` (optional) | Not-before time |
| `iat` | `now` (optional) | Issued-at time |

> ⚠️ **Common mistake:** `iss` and `sub` must both be your **Client ID**, not any other value.  
> Using the wrong value here causes a `invalid_client` error that can be hard to debug.

---

## Step 2 — Sign the JWT

Epic requires the JWT to be signed using **RS384** (RSA + SHA-384).

```python
import jwt
import uuid
import time

CLIENT_ID  = "your-client-id"
TOKEN_URL  = "https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token"

now = int(time.time())

payload = {
    "iss": CLIENT_ID,
    "sub": CLIENT_ID,
    "aud": TOKEN_URL,
    "jti": str(uuid.uuid4()),   # unique per request — never reuse
    "nbf": now,
    "iat": now,
    "exp": now + 300            # 5 minutes max
}

with open("keys/private_key.pem", "r") as f:
    private_key = f.read()

# Algorithm must be RS384 — Epic rejects RS256 for backend services
signed_jwt = jwt.encode(payload, private_key, algorithm="RS384")
```

### Algorithm Note

| Algorithm | Supported by Epic | Use Case |
|-----------|------------------|----------|
| `RS384` | ✅ Yes — **required** for backend | Backend Services (this guide) |
| `ES384` | ✅ Yes | Alternative if using EC keys |
| `RS256` | ❌ No for backend | SMART on FHIR patient apps only |

---

## Step 3 — Request the Access Token

Send the signed JWT to the token endpoint as a **form-urlencoded POST body** — not as query parameters, not as JSON.

```python
import requests

token_data = {
    "grant_type":            "client_credentials",
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion":      signed_jwt
}

response = requests.post(
    TOKEN_URL,
    data=token_data,                                   # form-urlencoded body
    headers={"Content-Type": "application/x-www-form-urlencoded"}
)

token_response = response.json()
access_token   = token_response.get("access_token")
expires_in     = token_response.get("expires_in")     # typically 300 seconds

print(f"Token received. Expires in {expires_in}s")
```

### Full Request Format (raw HTTP)

```
POST https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzM4NCIsInR5cCI6IkpXVCJ9...
```

> ⚠️ **Most common error:** Passing parameters in the query string (`?grant_type=...`) instead of the **POST body**.  
> This causes a silent failure or `invalid_request`. Always use the request body.

---

## Step 4 — Handle the Token Response

### Success Response

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type":   "Bearer",
  "expires_in":   300,
  "scope":        "system/Patient.read system/Observation.read"
}
```

### Token Caching Strategy

```python
import time

_token_cache = {
    "access_token": None,
    "expires_at":   0
}

def get_access_token():
    now = time.time()

    # Refresh token 30 seconds before it expires (safety buffer)
    if _token_cache["access_token"] and now < _token_cache["expires_at"] - 30:
        return _token_cache["access_token"]

    # Request a new token
    token_response = request_new_token()
    _token_cache["access_token"] = token_response["access_token"]
    _token_cache["expires_at"]   = now + token_response["expires_in"]

    return _token_cache["access_token"]
```

---

## Step 5 — Use the Token to Call FHIR APIs

Include the access token in every API request as a **Bearer token** in the Authorization header.

```python
BASE_URL = "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4"

def fhir_get(path: str) -> dict:
    token = get_access_token()
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept":        "application/fhir+json"
    }
    response = requests.get(f"{BASE_URL}{path}", headers=headers)
    response.raise_for_status()
    return response.json()

# Example calls
patient     = fhir_get("/Patient/eJzlzjFRAAAANAAAAA2")
vitals      = fhir_get("/Observation?patient=eJzlzjFRAAAANAAAAA2&category=vital-signs")
medications = fhir_get("/MedicationRequest?patient=eJzlzjFRAAAANAAAAA2")
```

---

## Scopes for Backend Services

Backend apps use **system-level scopes** — not patient or user scopes.

| Scope Format | Example | Access |
|-------------|---------|--------|
| `system/*.read` | All read-only resources | Broad access |
| `system/Patient.read` | Patient demographics only | Narrow access |
| `system/Observation.read` | Observations only | Narrow access |
| `system/MedicationRequest.read` | Medications only | Narrow access |

> 📌 **Epic Recommendation:** Always request the **minimum scopes** your app actually needs.  
> Requesting `system/*.read` is convenient for development but should be narrowed for production.

Scopes are configured on your app registration on [open.epic.com](https://open.epic.com).

---

## Production vs Sandbox Differences

| Factor | Sandbox | Production |
|--------|---------|------------|
| Client ID | Non-Production Client ID | Production Client ID |
| Token URL | `fhir.epic.com/interconnect-fhir-oauth/...` | Customer-specific URL |
| FHIR Base URL | `fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4` | Customer-specific URL |
| User mapping | Auto-mapped (no setup needed) | ECSA must map client ID to an Epic user account |
| Key pair | Can share across tests | Unique key pair per customer recommended |

> 📌 **On-premise customers:** If the Epic customer hosts their own Epic instance (on-prem), Epic recommends using **unique key pairs per customer** rather than a single shared key.

---

## Environment Variable Reference

All sensitive values must come from environment variables — never hardcoded:

```env
# .env

EPIC_CLIENT_ID=your-client-id-here
EPIC_TOKEN_URL=https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
EPIC_BASE_URL=https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
EPIC_PRIVATE_KEY_PATH=./keys/private_key.pem
EPIC_SCOPE=system/Patient.read system/Observation.read system/MedicationRequest.read
```

```python
import os
from dotenv import load_dotenv

load_dotenv()

CLIENT_ID   = os.getenv("EPIC_CLIENT_ID")
TOKEN_URL   = os.getenv("EPIC_TOKEN_URL")
BASE_URL    = os.getenv("EPIC_BASE_URL")
KEY_PATH    = os.getenv("EPIC_PRIVATE_KEY_PATH")
```

---

## `kid` Header — Public Key Thumbprint

When you upload your public key to the Epic portal, Epic displays a **thumbprint** (also called a fingerprint). This is the `kid` (Key ID) value.

Including `kid` in the JWT header is optional but recommended — it helps Epic locate the correct key when you have multiple keys registered:

```python
# Get thumbprint from openssl
# openssl x509 -noout -fingerprint -sha1 -in public_cert.pem

signed_jwt = jwt.encode(
    payload,
    private_key,
    algorithm="RS384",
    headers={"kid": "YOUR_KEY_THUMBPRINT_FROM_EPIC_PORTAL"}
)
```

---

## Pre-flight Checklist

Before running the token request, verify:

- [ ] `private_key.pem` exists at the path in `.env`
- [ ] `public_key.pem` has been uploaded to Epic app registration
- [ ] `EPIC_CLIENT_ID` matches the Non-Production Client ID from the portal
- [ ] JWT `iss` and `sub` are both set to the same `CLIENT_ID` value
- [ ] JWT `exp` is set to `now + 300` (not more than 5 minutes)
- [ ] JWT `jti` is a unique UUID generated fresh for each request
- [ ] Token POST uses **form-urlencoded body**, not query string or JSON
- [ ] `Content-Type: application/x-www-form-urlencoded` is set in the request header
- [ ] Signing algorithm is `RS384`

---

## Next Step

Once you can successfully retrieve an access token, proceed to:

**[Step 5 → Get Access Token (Full Script)](./05_get_access_token.md)**

---

## Reference Links

| Resource | URL |
|----------|-----|
| Epic OAuth 2.0 Backend Guide | `https://fhir.epic.com/Documentation?docId=oauth2&section=BackendOAuth2Guide` |
| Epic OAuth 2.0 Full Spec | `https://fhir.epic.com/Documentation?docId=oauth2` |
| App Registration Portal | https://open.epic.com |
| JWT Debugger | https://jwt.io |
| RFC 7519 — JWT Standard | https://tools.ietf.org/html/rfc7519 |
| SMART Backend Services Spec | https://hl7.org/fhir/uv/bulkdata/authorization/ |
