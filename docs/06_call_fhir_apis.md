# 06 — Call FHIR APIs

This document covers how to call Epic FHIR R4 APIs using the access token from Step 5.  
It includes the base URL, request headers, all tested endpoints, pagination, and Epic's full error code reference.

> 📖 **Epic Reference:**  
> `https://fhir.epic.com/Specifications`  
> `https://fhir.epic.com/Documentation?docId=fhirresourceindex`

---

## Base URL

| Environment | Base URL |
|-------------|----------|
| **Sandbox** | `https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4` |
| **Production** | `https://<customer-domain>/api/FHIR/R4` *(customer-specific)* |

---

## Required Headers for Every Request

```
Authorization: Bearer <access_token>
Accept:        application/fhir+json
```

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <access_token>` | ✅ Yes |
| `Accept` | `application/fhir+json` | ✅ Recommended |
| `Content-Type` | `application/fhir+json` | ✅ Only on POST/PUT |

---

## Base FHIR Client

Create a shared client module so all API modules share the same token and header logic:

```python
# api/fhir_client.py
"""
Shared FHIR HTTP client.
All resource modules import make_request() from here.
"""

import os
import logging
import requests
from dotenv import load_dotenv
from auth.get_access_token import get_access_token

load_dotenv()

BASE_URL = os.getenv("EPIC_BASE_URL")
log      = logging.getLogger(__name__)


def make_request(method: str, path: str, **kwargs) -> dict:
    """
    Make an authenticated request to the Epic FHIR API.

    Args:
        method: HTTP method — "GET", "POST", "PUT"
        path:   FHIR resource path, e.g. "/Patient/abc123"
        **kwargs: Additional args passed to requests (params, json, data)

    Returns:
        Parsed JSON response as dict

    Raises:
        requests.HTTPError: On 4xx / 5xx responses
    """
    token   = get_access_token()
    url     = f"{BASE_URL}{path}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept":        "application/fhir+json",
        "Content-Type":  "application/fhir+json"
    }

    log.info("%s %s", method.upper(), url)
    response = requests.request(method, url, headers=headers, timeout=30, **kwargs)

    if not response.ok:
        log.error("FHIR error %s — %s", response.status_code, response.text)
        response.raise_for_status()

    return response.json()


def fhir_read(resource: str, resource_id: str) -> dict:
    """Read a single FHIR resource by ID."""
    return make_request("GET", f"/{resource}/{resource_id}")


def fhir_search(resource: str, params: dict) -> dict:
    """Search a FHIR resource with query parameters."""
    return make_request("GET", f"/{resource}", params=params)
```

---

## Tested Endpoints

### 1. Patient

**Spec:** `https://fhir.epic.com/Specifications?api=931` (Read) | `https://fhir.epic.com/Specifications?api=932` (Search)

```python
# api/patient.py
from api.fhir_client import fhir_read, fhir_search

def get_patient(patient_id: str) -> dict:
    """Read a single patient by FHIR ID."""
    return fhir_read("Patient", patient_id)

def search_patients(family: str = None, given: str = None,
                    birthdate: str = None, identifier: str = None) -> dict:
    """
    Search patients by demographics.
    At least one parameter is required.
    """
    params = {}
    if family:     params["family"]     = family
    if given:      params["given"]      = given
    if birthdate:  params["birthdate"]  = birthdate
    if identifier: params["identifier"] = identifier
    return fhir_search("Patient", params)
```

**Example calls:**
```python
# Read by FHIR ID
patient = get_patient("eJzlzjFRAAAANAAAAA2")

# Search by name
results = search_patients(family="Lopez", given="Camila")

# Search by birthdate
results = search_patients(family="Smith", birthdate="1990-01-15")
```

---

### 2. Observation — Vital Signs

**Spec:** `https://fhir.epic.com/Specifications?api=968` (Read) | `https://fhir.epic.com/Specifications?api=973` (Search)

```python
# api/observation.py
from api.fhir_client import fhir_read, fhir_search

def get_vital_signs(patient_id: str, date: str = None) -> dict:
    """Search vital signs for a patient."""
    params = {
        "patient":  patient_id,
        "category": "vital-signs"
    }
    if date:
        params["date"] = date     # e.g. "ge2024-01-01"
    return fhir_search("Observation", params)

def get_lab_results(patient_id: str, date: str = None) -> dict:
    """Search lab results for a patient."""
    params = {
        "patient":  patient_id,
        "category": "laboratory"
    }
    if date:
        params["date"] = date
    return fhir_search("Observation", params)

def get_observation(observation_id: str) -> dict:
    """Read a single observation by ID."""
    return fhir_read("Observation", observation_id)
```

**Example calls:**
```python
# All vitals for a patient
vitals = get_vital_signs("eJzlzjFRAAAANAAAAA2")

# Labs after a date
labs = get_lab_results("eJzlzjFRAAAANAAAAA2", date="ge2024-01-01")
```

---

### 3. Encounter

**Spec:** `https://fhir.epic.com/Specifications?api=908` (Read) | `https://fhir.epic.com/Specifications?api=909` (Search)

```python
# api/encounter.py
from api.fhir_client import fhir_read, fhir_search

def get_encounters(patient_id: str, date: str = None,
                   status: str = None) -> dict:
    """
    Search encounters for a patient.

    status options: planned | arrived | in-progress | finished | cancelled
    """
    params = {"patient": patient_id}
    if date:   params["date"]   = date
    if status: params["status"] = status
    return fhir_search("Encounter", params)

def get_encounter(encounter_id: str) -> dict:
    return fhir_read("Encounter", encounter_id)
```

---

### 4. MedicationRequest

**Spec:** `https://fhir.epic.com/Specifications?api=996` (Read) | `https://fhir.epic.com/Specifications?api=997` (Search)

```python
# api/medication.py
from api.fhir_client import fhir_read, fhir_search

def get_medications(patient_id: str, status: str = None) -> dict:
    """
    Search medication requests for a patient.

    status options: active | on-hold | cancelled | completed | stopped
    """
    params = {"patient": patient_id}
    if status:
        params["status"] = status
    return fhir_search("MedicationRequest", params)

def get_medication(medication_id: str) -> dict:
    return fhir_read("MedicationRequest", medication_id)
```

---

### 5. AllergyIntolerance

**Spec:** `https://fhir.epic.com/Specifications?api=946` (Read) | `https://fhir.epic.com/Specifications?api=947` (Search)

```python
# api/allergy.py
from api.fhir_client import fhir_read, fhir_search

def get_allergies(patient_id: str) -> dict:
    return fhir_search("AllergyIntolerance", {"patient": patient_id})

def get_allergy(allergy_id: str) -> dict:
    return fhir_read("AllergyIntolerance", allergy_id)
```

---

### 6. Condition — Problem List

**Spec:** `https://fhir.epic.com/Specifications?api=951` (Read) | `https://fhir.epic.com/Specifications?api=953` (Search)

```python
# api/condition.py
from api.fhir_client import fhir_read, fhir_search

def get_problems(patient_id: str) -> dict:
    """Get active problem list items."""
    return fhir_search("Condition", {
        "patient":  patient_id,
        "category": "problem-list-item"
    })

def get_encounter_diagnoses(patient_id: str) -> dict:
    """Get encounter-level diagnoses."""
    return fhir_search("Condition", {
        "patient":  patient_id,
        "category": "encounter-diagnosis"
    })

def get_condition(condition_id: str) -> dict:
    return fhir_read("Condition", condition_id)
```

---

### 7. DiagnosticReport

**Spec:** `https://fhir.epic.com/Specifications?api=988` (Read) | `https://fhir.epic.com/Specifications?api=989` (Search)

```python
# api/diagnostic_report.py
from api.fhir_client import fhir_read, fhir_search

def get_diagnostic_reports(patient_id: str, category: str = None) -> dict:
    """
    category options: LAB | RAD | cardiology | etc.
    """
    params = {"patient": patient_id}
    if category:
        params["category"] = category
    return fhir_search("DiagnosticReport", params)
```

---

### 8. Immunization

**Spec:** `https://fhir.epic.com/Specifications?api=1070` (Read) | `https://fhir.epic.com/Specifications?api=1071` (Search)

```python
# api/immunization.py
from api.fhir_client import fhir_search

def get_immunizations(patient_id: str) -> dict:
    return fhir_search("Immunization", {"patient": patient_id})
```

---

## Handling Paginated Responses (Bundles)

Epic returns search results as a FHIR **Bundle**. Large result sets are paginated.

```python
def get_all_pages(first_response: dict) -> list:
    """
    Iterate through all pages of a FHIR Bundle response.
    Returns a flat list of all resource entries.
    """
    import requests
    from auth.get_access_token import get_access_token

    all_entries = []
    bundle      = first_response

    while True:
        entries = bundle.get("entry", [])
        all_entries.extend(entries)

        # Find the "next" link in the bundle
        next_url = None
        for link in bundle.get("link", []):
            if link.get("relation") == "next":
                next_url = link.get("url")
                break

        if not next_url:
            break   # No more pages

        token    = get_access_token()
        headers  = {
            "Authorization": f"Bearer {token}",
            "Accept":        "application/fhir+json"
        }
        response = requests.get(next_url, headers=headers, timeout=30)
        response.raise_for_status()
        bundle   = response.json()

    return all_entries


# Usage example
first_page = get_lab_results("eJzlzjFRAAAANAAAAA2")
all_labs   = get_all_pages(first_page)
print(f"Total lab results: {len(all_labs)}")
```

---

## Parsing a FHIR Bundle Response

```python
def extract_resources(bundle: dict, resource_type: str = None) -> list:
    """
    Extract resources from a FHIR Bundle response.

    Args:
        bundle:        The full bundle response dict
        resource_type: Optional filter e.g. "Patient", "Observation"

    Returns:
        List of resource dicts
    """
    resources = []
    for entry in bundle.get("entry", []):
        resource = entry.get("resource", {})
        if resource_type and resource.get("resourceType") != resource_type:
            continue
        resources.append(resource)
    return resources


# Usage
bundle    = get_vital_signs("eJzlzjFRAAAANAAAAA2")
resources = extract_resources(bundle, "Observation")

for obs in resources:
    code  = obs.get("code", {}).get("text", "Unknown")
    value = obs.get("valueQuantity", {})
    print(f"{code}: {value.get('value')} {value.get('unit')}")
```

---

## Epic FHIR Error Codes

These are Epic's official FHIR error codes returned in the `OperationOutcome` response body.

### Fatal Errors — Request will fail, no results returned

| Code | Description | Common Cause |
|------|-------------|-------------|
| `4100` | Invalid parameter in request | Nonexistent patient ID: `?patient=foo` |
| `4102` | Invalid resource ID in read request | `AllergyIntolerance/foo` — bad ID format |
| `4103` | Resource has been deleted | Requesting a deleted record |
| `4104` | Required FHIR element unavailable | SNOMED code missing for smoking status |
| `4107` | Patient record has been merged | Epic returns HTTP redirect — follow it |
| `4110` | No parameters provided in search | `AllergyIntolerance?` — empty search |
| `4111` | Required search parameter missing | `Condition?category=diagnosis` — no patient ID |
| `4112` | Invalid combination of parameters | Two different patient IDs in same request |
| `4113` | Paginated session has expired | Re-issue the original search query |
| `4115` | Required parameter has invalid value | `Condition?patient=ID&category=foo` |
| `4118` | User not authorized for this request | Scope not granted or wrong user context |
| `4127` | Search exceeds 100 results | Add filters to narrow the result set |
| `4130` | Break-the-Glass required | Sensitive encounter — check Epic security config |
| `4131` | Patient is restricted | Hospital employee patient — BTG required |
| `4135` | Maximum document queries reached for today | Retry tomorrow or reduce query frequency |
| `59102` | Invalid content against specification | `?patient=fakepatientid` |
| `59105` | Structural issue — invalid JSON syntax | Malformed request body |
| `59108` | Required element missing | Missing `category` or `code` in Observation search |
| `59111` | Required parameter has invalid value | `?category=labresults` — wrong category name |
| `59141` | Duplicate record attempt | AllergyIntolerance.Create — allergy already exists |
| `59144` | Reference not found | Binary or DocumentReference ID does not exist |
| `59159` | Business rule violation | Missing required patient context |
| `59177` | Unexpected internal error | Epic server error — retry after delay |
| `59187` | No patient-entered flowsheets found | Patient has no PEF assigned (RPM use case) |
| `59188` | Flowsheet row not found by given codes | LOINC code not mapped to a flowsheet row |
| `59189` | Failed to file the reading | Observation.Create filing failure |

### Warning Errors — Request succeeds but partial or advisory

| Code | Description |
|------|-------------|
| `4101` | No results found — valid request but no data documented |
| `4117` | No CVX code for Immunization resource |
| `4119` | Additional data may exist — patient proxy or restricted context |
| `4122` | Unknown query parameter supplied (ignored) |
| `59100` | Unknown parameter in request — treated as informational |
| `59101` | Unknown category value — ignored but request proceeds |
| `59109` | Optional element has invalid value — ignored |
| `59133` | Processing issues or search exceeded 100 results |

---

### Reading an `OperationOutcome` Error Response

When a request fails, Epic returns an `OperationOutcome` resource:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "fatal",
      "code": "invalid",
      "details": {
        "coding": [
          {
            "system": "urn:oid:1.2.840.114350.1.13.0.1.7.2.657369",
            "code": "4111",
            "display": "Required search parameter missing from request"
          }
        ]
      },
      "diagnostics": "patient"
    }
  ]
}
```

### Error Handling in Code

```python
def safe_fhir_get(resource: str, params: dict) -> dict | None:
    """
    FHIR search with structured error handling for Epic error codes.
    Returns None on 4101 (no results), raises on all other errors.
    """
    import requests
    from api.fhir_client import fhir_search

    try:
        return fhir_search(resource, params)

    except requests.HTTPError as e:
        response = e.response
        if response is None:
            raise

        try:
            outcome = response.json()
            issues  = outcome.get("issue", [])

            for issue in issues:
                code = issue.get("details", {}) \
                            .get("coding", [{}])[0] \
                            .get("code", "")

                if code == "4101":
                    # No results — not an error, just empty
                    return {"resourceType": "Bundle", "entry": []}

                if code == "4118":
                    raise PermissionError(
                        "Not authorized. Check app scopes on open.epic.com."
                    )

                if code == "4107":
                    # Patient merged — follow redirect
                    redirect = response.headers.get("Location")
                    raise ValueError(f"Patient record merged. New location: {redirect}")

                if code == "4127":
                    raise ValueError(
                        "Search returned 100+ results. Add more filters to narrow the query."
                    )

        except (ValueError, KeyError):
            pass

        raise  # Re-raise original error for unhandled codes
```

---

## Endpoint Summary Table

| Resource | Operation | Method | Path | Epic Spec |
|----------|-----------|--------|------|-----------|
| Patient | Read | GET | `/Patient/{id}` | [api=931](https://fhir.epic.com/Specifications?api=931) |
| Patient | Search | GET | `/Patient?family=...` | [api=932](https://fhir.epic.com/Specifications?api=932) |
| Patient | $match | POST | `/Patient/$match` | [api=10423](https://fhir.epic.com/Specifications?api=10423) |
| Observation (Vitals) | Read | GET | `/Observation/{id}` | [api=968](https://fhir.epic.com/Specifications?api=968) |
| Observation (Vitals) | Search | GET | `/Observation?patient=...&category=vital-signs` | [api=973](https://fhir.epic.com/Specifications?api=973) |
| Observation (Labs) | Read | GET | `/Observation/{id}` | [api=998](https://fhir.epic.com/Specifications?api=998) |
| Observation (Labs) | Search | GET | `/Observation?patient=...&category=laboratory` | [api=999](https://fhir.epic.com/Specifications?api=999) |
| Encounter | Read | GET | `/Encounter/{id}` | [api=908](https://fhir.epic.com/Specifications?api=908) |
| Encounter | Search | GET | `/Encounter?patient=...` | [api=909](https://fhir.epic.com/Specifications?api=909) |
| MedicationRequest | Read | GET | `/MedicationRequest/{id}` | [api=996](https://fhir.epic.com/Specifications?api=996) |
| MedicationRequest | Search | GET | `/MedicationRequest?patient=...` | [api=997](https://fhir.epic.com/Specifications?api=997) |
| AllergyIntolerance | Read | GET | `/AllergyIntolerance/{id}` | [api=946](https://fhir.epic.com/Specifications?api=946) |
| AllergyIntolerance | Search | GET | `/AllergyIntolerance?patient=...` | [api=947](https://fhir.epic.com/Specifications?api=947) |
| Condition (Problems) | Read | GET | `/Condition/{id}` | [api=951](https://fhir.epic.com/Specifications?api=951) |
| Condition (Problems) | Search | GET | `/Condition?patient=...&category=problem-list-item` | [api=953](https://fhir.epic.com/Specifications?api=953) |
| DiagnosticReport | Read | GET | `/DiagnosticReport/{id}` | [api=988](https://fhir.epic.com/Specifications?api=988) |
| DiagnosticReport | Search | GET | `/DiagnosticReport?patient=...` | [api=989](https://fhir.epic.com/Specifications?api=989) |
| Immunization | Read | GET | `/Immunization/{id}` | [api=1070](https://fhir.epic.com/Specifications?api=1070) |
| Immunization | Search | GET | `/Immunization?patient=...` | [api=1071](https://fhir.epic.com/Specifications?api=1071) |

---

## Checklist Before Testing

- [ ] `.env` has correct `EPIC_BASE_URL` pointing to sandbox
- [ ] Access token fetches successfully (Step 5 passing)
- [ ] All API calls use `requests.get()` not browser URL
- [ ] `Authorization: Bearer <token>` header included in every request
- [ ] Search calls include at least one required parameter (usually `patient`)
- [ ] Paginated results handled — do not assume first page is complete
- [ ] `OperationOutcome` errors are parsed, not just the HTTP status code

---

## Next Step

Proceed to:

**[Step 7 → Sandbox Test Patients](./07_sandbox_test_patients.md)**

---

## Reference Links

| Resource | URL |
|----------|-----|
| Epic FHIR API Specifications | https://fhir.epic.com/Specifications |
| FHIR R4 Resource Index | https://fhir.epic.com/Documentation?docId=fhirresourceindex |
| Search Parameters Guide | https://fhir.epic.com/Documentation?docId=searchparameters |
| FHIR R4 Bundle Spec (HL7) | https://hl7.org/fhir/R4/bundle.html |
| OperationOutcome Spec | https://hl7.org/fhir/R4/operationoutcome.html |
