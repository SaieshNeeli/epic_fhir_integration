# Step 2 — Register an Application in Epic

After creating your developer account, the next step is to register an application.

Applications must be registered before accessing Epic FHIR APIs.

Epic uses the App Orchard platform to manage developer applications.

Portal:
https://open.epic.com/

---

## Steps to Register the Application

1. Go to the Epic App Orchard portal
2. Sign in with your developer account
3. Navigate to **My Apps**
4. Click **Create New App**

---

## Basic Application Details

Fill in the following details when creating the application.

App Name  
Example: Epic FHIR Backend Integration

App Type  
Backend System

FHIR Version  
R4

Authentication Method  
OAuth 2.0 Backend Services

---

## Scopes

Scopes define what data your application can access.

Common scopes for testing include:

- patient/*.read
- observation.read
- encounter.read
- medicationrequest.read

Choose scopes depending on the APIs your application will use.

---

## After Registration

Once the application is created, Epic will provide:

Client ID

This Client ID will be used in the OAuth authentication flow to request access tokens.

Save the Client ID securely.

---

## Next Step

Proceed to:

Step 3 — Generate RSA Keys
