# Epic FHIR Backend Integration

## Overview
Backend integration with Epic's FHIR R4 APIs for [your app's purpose].

## Tech Stack
- Language/Framework: Node.js / Python / Java
- FHIR Version: R4 (primary), STU3 (fallback where needed)
- Auth: OAuth 2.0 — Backend Services (JWT client credentials)
- Sandbox: https://fhir.epic.com/

## Quick Start
1. Clone repo
2. Copy `.env.example` → `.env` and fill in credentials
3. Run `npm install`
4. Run `npm run test:sandbox`

## Key Links
- Epic on FHIR Portal: https://fhir.epic.com/
- API Specifications: https://fhir.epic.com/Specifications
- OAuth 2.0 Docs: https://fhir.epic.com/Documentation (OAuth 2.0 Tutorial)
- Sandbox Test Data: [https://fhir.epic.com/Documentation ](https://fhir.epic.com/Documentation?docId=testpatients)(Sandbox Test Data)
- App Registration: [ https://open.epic.com/](https://fhir.epic.com/Developer/Apps)

## App Details
- Client ID: (store in .env, reference here as CLIENT_ID)
- App Type: Backend / Clinician-Facing / Patient-Facing
- Registered Scopes: [list all scopes]
