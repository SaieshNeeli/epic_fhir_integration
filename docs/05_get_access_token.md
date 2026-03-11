# 05 — Get Access Token

This document provides the **complete, production-ready Python script** to generate a signed JWT and exchange it for an Epic FHIR access token.  
This is the working implementation of the flow explained in `04_configure_oauth_backend.md`.

> 📖 **Epic Reference:**  
> `https://fhir.epic.com/Documentation?docId=oauth2&section=BackendOAuth2Guide`

---

## Prerequisites

| Requirement | Status |
|-------------|--------|
| `private_key.pem` in `keys/` folder | From Step 3 |
| `public_key.pem` uploaded to Epic | From Step 2 |
| `EPIC_CLIENT_ID` saved in `.env` | From Step 2 |
| Python packages installed | See below |

### Install Dependencies

```bash
pip install pyjwt requests python-dotenv cryptography
```

| Package | Purpose |
|---------|---------|
| `pyjwt` | Build and sign the JWT |
| `requests` | HTTP POST to token endpoint |
| `python-dotenv` | Load `.env` variables |
| `cryptography` | Required by pyjwt for RS384 signing |

> ⚠️ **`cryptography` is required.** Without it, `pyjwt` silently falls back to HS256 and Epic will reject the token with `invalid_client`.

---

## Project File Location

```
epic-fhir-integration/
├── auth/
│   └── get_access_token.py     ← this file
├── keys/
│   └── private_key.pem         ← gitignored
└── .env                        ← gitignored
```

---

## `.env` Setup

```env
EPIC_CLIENT_ID=your-non-production-client-id
EPIC_TOKEN_URL=https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
EPIC_PRIVATE_KEY_PATH=./keys/private_key.pem
```

---

## Full Script — `auth/get_access_token.py`

```python
"""
Epic FHIR — Backend OAuth 2.0 Access Token
-------------------------------------------
Generates a signed RS384 JWT and exchanges it for a Bearer access token
using the OAuth 2.0 Client Credentials flow.

Epic Reference:
  https://fhir.epic.com/Documentation?docId=oauth2&section=BackendOAuth2Guide
"""

import os
import time
import uuid
import logging

import jwt
import requests
from dotenv import load_dotenv

# ── Load environment variables ────────────────────────────────────────────────
load_dotenv()

CLIENT_ID   = os.getenv("EPIC_CLIENT_ID")
TOKEN_URL   = os.getenv("EPIC_TOKEN_URL")
KEY_PATH    = os.getenv("EPIC_PRIVATE_KEY_PATH")

# ── Logging ───────────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
log = logging.getLogger(__name__)

# ── In-memory token cache ─────────────────────────────────────────────────────
_token_cache = {
    "access_token": None,
    "expires_at":   0
}


def _load_private_key() -> str:
    """Load RSA private key from the path specified in .env."""
    if not KEY_PATH:
        raise EnvironmentError("EPIC_PRIVATE_KEY_PATH is not set in .env")

    if not os.path.exists(KEY_PATH):
        raise FileNotFoundError(
            f"Private key not found at: {KEY_PATH}\n"
            "Generate it with: openssl genrsa -out keys/private_key.pem 2048"
        )

    with open(KEY_PATH, "r") as f:
        return f.read()


def _build_jwt(private_key: str) -> str:
    """
    Build and sign a JWT for Epic's OAuth 2.0 client credentials flow.

    Required claims (per Epic specification):
      iss  — Client ID (identifies your app)
      sub  — Client ID (same as iss for backend apps)
      aud  — Token endpoint URL
      jti  — Unique UUID per request (prevents token replay)
      exp  — Expiry: no more than 5 minutes from now
      nbf  — Not-before: current time
      iat  — Issued-at: current time
    """
    if not CLIENT_ID:
        raise EnvironmentError("EPIC_CLIENT_ID is not set in .env")
    if not TOKEN_URL:
        raise EnvironmentError("EPIC_TOKEN_URL is not set in .env")

    now = int(time.time())

    payload = {
        "iss": CLIENT_ID,       # Must match registered Client ID exactly
        "sub": CLIENT_ID,       # Must be same as iss for backend services
        "aud": TOKEN_URL,       # Full token endpoint URL
        "jti": str(uuid.uuid4()),  # Unique per request — never reuse
        "nbf": now,
        "iat": now,
        "exp": now + 300        # 5 minutes max — Epic rejects anything longer
    }

    # Algorithm must be RS384 — Epic rejects RS256 for backend services
    signed_jwt = jwt.encode(payload, private_key, algorithm="RS384")

    log.info("JWT built successfully (jti=%s)", payload["jti"])
    return signed_jwt


def _request_token(signed_jwt: str) -> dict:
    """
    POST signed JWT to Epic's token endpoint.

    CRITICAL: Parameters must be in the POST BODY as form-urlencoded.
    Do NOT pass them as query parameters or as JSON — Epic will reject both.
    """
    request_body = {
        "grant_type":            "client_credentials",
        "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        "client_assertion":      signed_jwt
    }

    headers = {
        "Content-Type": "application/x-www-form-urlencoded"
    }

    log.info("Requesting access token from: %s", TOKEN_URL)

    response = requests.post(
        TOKEN_URL,
        data=request_body,
        headers=headers,
        timeout=15
    )

    if response.status_code != 200:
        log.error(
            "Token request failed — HTTP %s: %s",
            response.status_code,
            response.text
        )
        response.raise_for_status()

    token_data = response.json()
    log.info(
        "Access token received. Expires in %ss",
        token_data.get("expires_in", "unknown")
    )
    return token_data


def get_access_token() -> str:
    """
    Return a valid Epic access token, using the in-memory cache if available.

    Refreshes the token automatically when it is within 30 seconds of expiry.
    This is the main function to call from your API modules.

    Returns:
        str: A valid Bearer access token

    Raises:
        EnvironmentError: If required env vars are missing
        FileNotFoundError: If private key file is not found
        requests.HTTPError: If the token endpoint returns an error
    """
    now = time.time()

    # Return cached token if still valid (with 30s safety buffer)
    if _token_cache["access_token"] and now < _token_cache["expires_at"] - 30:
        log.debug("Using cached access token")
        return _token_cache["access_token"]

    log.info("Fetching new access token...")

    private_key = _load_private_key()
    signed_jwt  = _build_jwt(private_key)
    token_data  = _request_token(signed_jwt)

    # Update cache
    _token_cache["access_token"] = token_data["access_token"]
    _token_cache["expires_at"]   = now + token_data.get("expires_in", 300)

    return _token_cache["access_token"]


# ── Run directly to test ──────────────────────────────────────────────────────
if __name__ == "__main__":
    try:
        token = get_access_token()
        print("\n✅ Access token received successfully")
        print(f"\nToken (first 80 chars):\n{token[:80]}...")
        print(f"\nExpires at (epoch): {_token_cache['expires_at']}")
    except FileNotFoundError as e:
        print(f"\n❌ Key file error: {e}")
    except EnvironmentError as e:
        print(f"\n❌ Environment error: {e}")
    except requests.HTTPError as e:
        print(f"\n❌ Token request failed: {e}")
```

---

## Run the Script

```bash
python auth/get_access_token.py
```

### Expected Output (Success)

```
2024-01-15 10:23:01 [INFO] Fetching new access token...
2024-01-15 10:23:01 [INFO] JWT built successfully (jti=a3f9b2c1-...)
2024-01-15 10:23:01 [INFO] Requesting access token from: https://fhir.epic.com/...
2024-01-15 10:23:02 [INFO] Access token received. Expires in 300s

✅ Access token received successfully

Token (first 80 chars):
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbGllbnRfaWQiOiJkNDUwNDljMy0z...

Expires at (epoch): 1705316582.4
```

---

## Importing Into Other Modules

Once the script is working, use `get_access_token()` from your API service modules:

```python
# api/patient.py
from auth.get_access_token import get_access_token
import requests
import os

BASE_URL = os.getenv("EPIC_BASE_URL")

def get_patient(patient_id: str) -> dict:
    token = get_access_token()       # auto-cached + auto-refreshed
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept":        "application/fhir+json"
    }
    response = requests.get(f"{BASE_URL}/Patient/{patient_id}", headers=headers)
    response.raise_for_status()
    return response.json()
```

---

## Epic's Official Validation Rules

Epic requires all of the following to be true for a token request to succeed:

| Rule | What to Check |
|------|--------------|
| OAuth 2.0 must be enabled | Confirmed during app registration |
| Public key must be uploaded | Done in Step 2 |
| Use `client_assertion` not `client-assertion` | Check for hyphen vs underscore |
| Do not double-encode `client_assertion_type` | Pass the raw URN string |
| URL must end in `/token` not `/authorize` | Check your `EPIC_TOKEN_URL` |
| Use HTTP `POST` not `GET` | `requests.post()` — not `.get()` |
| Use `Content-Type: application/x-www-form-urlencoded` | Set in headers |
| Parameters in request **body** not URL | Use `data=` not `params=` |
| `iat` and `nbf` must **not** be in the future | Use `int(time.time())` |
| `exp` must be in the future | `now + 300` |
| `exp` must be **max 5 minutes** from now | Do not use `+ 3600` |

---

## Verify Your JWT Before Sending

If you are getting `invalid_client` or `400` errors, decode your JWT at **https://jwt.io** before sending it to Epic.

Paste the token into the Debugger section and confirm:

```json
{
  "iss": "your-client-id",        ← must match exactly
  "sub": "your-client-id",        ← must match iss
  "aud": "https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token",
  "jti": "some-uuid-here",
  "nbf": 1705316282,              ← current time or earlier
  "iat": 1705316282,              ← current time or earlier
  "exp": 1705316582               ← exactly 300s after iat
}
```

Also confirm the **Header** shows:

```json
{
  "alg": "RS384",
  "typ": "JWT"
}
```

---

## Token Response Fields

| Field | Description |
|-------|-------------|
| `access_token` | Bearer token to use in API calls |
| `token_type` | Always `Bearer` in Epic's implementation |
| `expires_in` | Seconds until expiry — typically `300` |
| `scope` | Scopes actually granted (may differ from requested) |

> 📌 If the `scope` field in the response is narrower than what you requested, check your app's registered scopes on [open.epic.com](https://open.epic.com). Scopes not registered on the app will be silently dropped.

---

## Summary Checklist

Before calling APIs, confirm:

- [ ] Script runs without errors: `python auth/get_access_token.py`
- [ ] Response `status_code` is `200`
- [ ] `access_token` is present in the response
- [ ] `token_type` is `Bearer`
- [ ] Token is being cached (not re-requested on every API call)
- [ ] `.env` and `keys/` are both in `.gitignore`

---

## Next Step

Once you have a working access token, proceed to:

**[Step 6 → Call FHIR APIs](./06_call_fhir_apis.md)**

---

## Reference Links

| Resource | URL |
|----------|-----|
| Epic OAuth 2.0 Backend Guide | `https://fhir.epic.com/Documentation?docId=oauth2&section=BackendOAuth2Guide` |
| Epic App Registration | https://open.epic.com |
| JWT Debugger | https://jwt.io |
| PyJWT Docs | https://pyjwt.readthedocs.io |
| RFC 7519 — JWT Standard | https://tools.ietf.org/html/rfc7519 |
