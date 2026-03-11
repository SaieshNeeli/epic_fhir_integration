# Step 3 — Generate RSA Keys

Epic's OAuth 2.0 Backend Services flow requires an **RSA key pair**.  
Your backend signs JWTs with the **private key**. Epic verifies them using the **public key** you upload during app registration.

> 📋 **Developer Responsibility (from Epic's Guidelines):**  
> As a developer, you are solely responsible for your application and how it interacts with Epic Community Members' systems — including all security, privacy, and data handling aspects. Key management is a critical part of this responsibility.  
> Reference: [open.epic Terms of Use](https://fhir.epic.com/Documentation?docId=developerguidelines)

---

## Prerequisites

- `openssl` installed on your machine
- A terminal / command prompt

Verify `openssl` is available:

```bash
openssl version
```

---

## Key Generation Commands

### 1. Generate Private Key (2048-bit RSA)

```bash
openssl genrsa -out private_key.pem 2048
```

### 2. Extract Public Key

```bash
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

### 3. Create Self-Signed Certificate (Valid for 1 Year)

```bash
openssl req -new -x509 -key private_key.pem -out public_cert.pem -days 365
```

---

## Files Generated

| File | Purpose | Commit to Git? |
|------|---------|---------------|
| `private_key.pem` | Signs your JWT — keep this secret | ❌ Never |
| `public_key.pem` | Upload to Epic during app registration | ❌ Never |
| `public_cert.pem` | Certificate reference | ❌ Never |

> ⚠️ **All three files must be added to `.gitignore` immediately.**  
> If `private_key.pem` is ever committed — even once — rotate your keys and revoke the app registration.

---

## Epic Developer Guidelines — Security Requirements

The following rules are sourced directly from Epic's official developer guidelines and apply to key management and your overall integration.

### You Are Responsible for Your App's Security

Epic's developer guidelines make clear that as a developer you bear full responsibility for:

- Security vulnerabilities in your app
- Privacy breaches involving patient data
- How your app accesses and handles data from Epic Community Members' systems

This means your RSA private key is not just a technical secret — it is a legal and compliance boundary. A leaked private key can allow unauthorized actors to impersonate your backend and access protected health information.

### Reporting Security Vulnerabilities

If you identify a security vulnerability — including a leaked key or unauthorized access — you are required to report it to Epic immediately:

```
https://www.epic.com/epic/page/reporting-potential-security-vulnerability
```

### Sandbox Rules — No Real Patient Data

Epic's guidelines explicitly state:

- The sandbox is for testing purposes only
- The sandbox must be populated with **sample or synthetic data only**
- You must never put **individually identifiable information** into Epic-provided sandboxes
- Epic may wipe sandbox data at any time at its own discretion

> This applies to your key-protected API calls during testing. Never use real patient records — even anonymized ones — in sandbox testing.

### Direct Access Requires Approval

Direct access to an Epic Community Member's live system is not automatic. Even after your app is approved, accessing a production Epic environment requires explicit approval from both the Epic customer and Epic itself. Your keys will need to be re-registered for each production environment.

---

## Recommended Key Storage

Store keys in a dedicated folder that is fully excluded from Git:

```
epic-fhir-integration/
└── keys/               ← entire folder in .gitignore
    ├── private_key.pem
    ├── public_key.pem
    └── public_cert.pem
```

Add this to your `.gitignore`:

```gitignore
# RSA Keys — never commit
*.pem
*.key
*.p12
keys/
```

### Production Key Storage

In production environments, never store keys on disk as plain files. Use a dedicated secrets manager:

| Platform | Recommended Service |
|----------|-------------------|
| AWS | Secrets Manager or Parameter Store (SecureString) |
| Azure | Azure Key Vault |
| GCP | Google Secret Manager |
| Self-hosted | HashiCorp Vault |

---

## Using Keys in Code

Reference the private key via an environment variable — never hardcode the path or value:

```python
import os

key_path = os.getenv("EPIC_PRIVATE_KEY_PATH")

with open(key_path, "r") as f:
    private_key = f.read()
```

In production, load directly from a secrets manager:

```python
import boto3  # AWS example

def get_private_key():
    client = boto3.client("secretsmanager")
    secret = client.get_secret_value(SecretId="epic/private-key")
    return secret["SecretString"]
```

---

## Key Rotation

Epic recommends rotating your RSA keys periodically. When rotating:

1. Generate a new key pair using the commands above
2. Upload the new `public_key.pem` to your app registration on [open.epic.com](https://open.epic.com)
3. Update your secrets manager or `.env` to point to the new private key
4. Verify token generation still works in sandbox before switching production
5. Securely delete the old private key from all storage locations

---

## What Epic Does With Your Public Key

When you upload `public_key.pem` to Epic's app registration portal, Epic stores it against your Client ID. Every time your backend sends a signed JWT, Epic uses your public key to verify:

- The JWT was signed by you (not tampered with)
- The signature matches the registered Client ID
- The token has not expired (`exp` claim is in the future)
- The token has not been replayed (unique `jti` claim)

---

## Compliance Note — Insights Condition (45 CFR 170.407)

Epic is required under federal regulation (45 CFR 170.407) to collect and report information about applications that connect to its certified software. This includes your app's name, developer details, intended purpose, and usage patterns. Key-based authentication is part of how Epic identifies and tracks your application's activity.

Reference: [https://www.ecfr.gov/current/title-45/section-170.407](https://www.ecfr.gov/current/title-45/section-170.407)

---

## Summary Checklist

Before moving to the next step, confirm:

- [ ] `private_key.pem` generated and stored only in `keys/` folder
- [ ] `public_key.pem` generated and ready to upload to Epic
- [ ] `keys/` and `*.pem` added to `.gitignore`
- [ ] No key files staged or committed in Git (`git status` is clean)
- [ ] Key path set in `.env` using `EPIC_PRIVATE_KEY_PATH`
- [ ] Sandbox testing plan uses only synthetic patient data

---

## Next Step

Once your keys are generated and stored securely, proceed to:

**[Step 4 → Generate JWT and Request Access Token](./step-04-generate-jwt-token.md)**

Upload `public_key.pem` to Epic during app registration (Step 2) if not already done.

---

## Reference Links

| Resource | URL |
|----------|-----|
| Epic Developer Guidelines | https://fhir.epic.com/Documentation?docId=developerguidelines |
| open.epic Terms of Use | https://open.epic.com |
| Security Vulnerability Reporting | https://www.epic.com/epic/page/reporting-potential-security-vulnerability |
| App Registration Portal | https://open.epic.com |
| Insights Condition Regulation | https://www.ecfr.gov/current/title-45/section-170.407 |
