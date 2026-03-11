# 07 — Sandbox Test Patients

This document lists all known Epic sandbox test patients, their FHIR IDs, MyChart credentials, and ready-to-run test queries for every resource type tested in this integration.

> 📖 **Epic Reference:**  
> `https://fhir.epic.com/Documentation?docId=testpatients`  
> Log in to the Epic on FHIR portal to view the full Sandbox Test Data document.

> ⚠️ **Epic Sandbox Rule:**  
> The sandbox is for testing with **synthetic data only**. You must never enter individually identifiable information into Epic-provided sandboxes. Epic may wipe sandbox data at any time without notice.

---

## Sandbox Base URL

```
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
```

---

## MyChart Test Patients (Patient-Facing Context)

These patients have MyChart portal accounts and are used for **patient-facing SMART on FHIR** apps that require a logged-in patient context.

| Name | MyChart Username | MyChart Password | Context |
|------|-----------------|-----------------|---------|
| Derrick Lin | `fhirjason` | `epicepic1` | Patient-facing OAuth launch |
| Camilla Lopez | — | — | Patient-facing OAuth launch |
| Desiree Powell | — | — | Patient-facing OAuth launch |
| Olivia Roberts | — | — | Patient-facing OAuth launch |

> 📌 **Backend note:** For backend service apps (this integration), you do **not** log in as a MyChart patient. You use your app's Client ID and private key. The MyChart credentials above are only relevant for patient-facing SMART on FHIR launches.

---

## Clinician Context Test Patients

These patients are available in Epic's provider-facing sandbox environment:

| Name | Context |
|------|---------|
| Cadence, Anna | Provider / clinician launch |
| Clin Doc, Henry | Provider / clinician launch |
| Grand Central, John | Provider / clinician launch |
| Optime, Omar | Provider / clinician launch |
| Nelson, Kyle | Provider / clinician launch |

---

## Known Sandbox FHIR Patient IDs (R4)

| Patient Name | FHIR ID (R4) | Source |
|-------------|-------------|--------|
| Derrick Lin | `eq081-VQEgP8drUUqCWzHfw3` | Epic sandbox — publicly documented |
| Jayden Jackson | `eNO3wqOfAltfnWMfWBQ1WmQ3` | Epic IPS sample response |
| example (generic) | `eJzlzjFRAAAANAAAAA2` | Epic API Spec inline examples |

> 📌 **Always re-verify FHIR IDs before testing.** Sandbox resets change all FHIR IDs. Use `Patient.Search` by name to confirm the current ID before running a test suite.

---

## Step 1 — Confirm Sandbox Access

Run this first to verify your token and sandbox connection:

```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Patient/eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

Expected: HTTP `200` with a `Patient` resource JSON body.

---

## Step 2 — Find a Test Patient by Name

```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Patient?family=Lin&given=Derrick" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

```python
from api.patient import search_patients

results  = search_patients(family="Lin", given="Derrick")
for entry in results.get("entry", []):
    p = entry["resource"]
    print(p["id"], p["name"][0]["family"], p["name"][0].get("given", []))
```

---

## Ready-to-Run curl Commands

Replace `YOUR_ACCESS_TOKEN` with a live token from `python auth/get_access_token.py`.

### Patient.Read
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Patient/eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Observation — Vital Signs
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Observation?patient=eq081-VQEgP8drUUqCWzHfw3&category=vital-signs" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Observation — Labs
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Observation?patient=eq081-VQEgP8drUUqCWzHfw3&category=laboratory" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Condition — Problem List
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Condition?patient=eq081-VQEgP8drUUqCWzHfw3&category=problem-list-item" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### MedicationRequest
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/MedicationRequest?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### AllergyIntolerance
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/AllergyIntolerance?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Encounter
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Encounter?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### DiagnosticReport
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/DiagnosticReport?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Immunization
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Immunization?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

### Goal
```bash
curl -X GET \
  "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/Goal?patient=eq081-VQEgP8drUUqCWzHfw3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Accept: application/fhir+json"
```

---

## Python Test Runner — `tests/test_sandbox.py`

```python
"""
tests/test_sandbox.py
---------------------
Runs all FHIR resource queries against the Epic sandbox
using the known test patient Derrick Lin.

Usage:
    python tests/test_sandbox.py
"""

import json
from auth.get_access_token import get_access_token
from api.fhir_client import fhir_read, fhir_search

# ── Sandbox test patient ──────────────────────────────────────────────────────
PATIENT_ID = "eq081-VQEgP8drUUqCWzHfw3"   # Derrick Lin — update after sandbox reset

# ── Test definitions ──────────────────────────────────────────────────────────
TESTS = [
    {
        "name":  "Token Generation",
        "fn":    lambda: get_access_token(),
        "check": lambda r: isinstance(r, str) and len(r) > 0
    },
    {
        "name":  "Patient.Read",
        "fn":    lambda: fhir_read("Patient", PATIENT_ID),
        "check": lambda r: r.get("resourceType") == "Patient"
    },
    {
        "name":  "Patient.Search by name",
        "fn":    lambda: fhir_search("Patient", {"family": "Lin", "given": "Derrick"}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Observation.Search — Vital Signs",
        "fn":    lambda: fhir_search("Observation", {"patient": PATIENT_ID, "category": "vital-signs"}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Observation.Search — Labs",
        "fn":    lambda: fhir_search("Observation", {"patient": PATIENT_ID, "category": "laboratory"}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Condition.Search — Problems",
        "fn":    lambda: fhir_search("Condition", {"patient": PATIENT_ID, "category": "problem-list-item"}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "MedicationRequest.Search",
        "fn":    lambda: fhir_search("MedicationRequest", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "AllergyIntolerance.Search",
        "fn":    lambda: fhir_search("AllergyIntolerance", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Encounter.Search",
        "fn":    lambda: fhir_search("Encounter", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "DiagnosticReport.Search",
        "fn":    lambda: fhir_search("DiagnosticReport", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Immunization.Search",
        "fn":    lambda: fhir_search("Immunization", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
    {
        "name":  "Goal.Search",
        "fn":    lambda: fhir_search("Goal", {"patient": PATIENT_ID}),
        "check": lambda r: r.get("resourceType") == "Bundle"
    },
]


def run_tests():
    passed = 0
    failed = 0

    print(f"\n{'═' * 58}")
    print(f"  Epic FHIR Sandbox Tests  |  Patient: {PATIENT_ID[:18]}...")
    print(f"{'═' * 58}\n")

    for test in TESTS:
        try:
            result = test["fn"]()
            ok     = test["check"](result)
            if ok:
                count = result.get("total", "—") if isinstance(result, dict) else "—"
                print(f"  ✅  {test['name']:<42}  total={count}")
                passed += 1
            else:
                print(f"  ❌  {test['name']}  ← unexpected response shape")
                print(f"      Got: {json.dumps(result)[:120]}")
                failed += 1
        except Exception as e:
            print(f"  ❌  {test['name']}")
            print(f"      Error: {e}")
            failed += 1

    print(f"\n{'─' * 58}")
    print(f"  {passed} passed  |  {failed} failed  |  {len(TESTS)} total")
    print(f"{'─' * 58}\n")

    if failed > 0:
        raise SystemExit(1)


if __name__ == "__main__":
    run_tests()
```

### Run

```bash
python tests/test_sandbox.py
```

### Expected Output

```
══════════════════════════════════════════════════════════
  Epic FHIR Sandbox Tests  |  Patient: eq081-VQEgP8drUUqC...
══════════════════════════════════════════════════════════

  ✅  Token Generation                            total=—
  ✅  Patient.Read                                total=—
  ✅  Patient.Search by name                      total=1
  ✅  Observation.Search — Vital Signs            total=12
  ✅  Observation.Search — Labs                   total=8
  ✅  Condition.Search — Problems                 total=3
  ✅  MedicationRequest.Search                    total=5
  ✅  AllergyIntolerance.Search                   total=2
  ✅  Encounter.Search                            total=6
  ✅  DiagnosticReport.Search                     total=4
  ✅  Immunization.Search                         total=3
  ✅  Goal.Search                                 total=2

──────────────────────────────────────────────────────────
  12 passed  |  0 failed  |  12 total
──────────────────────────────────────────────────────────
```

---

## Handling Empty Bundle Responses

An empty `entry` array with `total=0` is **valid** — it means no data is documented for that resource on this test patient. Do not treat it as an error.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

Your code should handle this as:

```python
entries = response.get("entry", [])   # safe — returns [] if no entry key
if not entries:
    print("No results — valid empty response")
```

---

## If the Sandbox FHIR ID Has Changed

After a sandbox reset, error `4102` means the FHIR ID is stale. Re-discover it:

```python
from api.fhir_client import fhir_search

results = fhir_search("Patient", {"family": "Lin", "given": "Derrick"})
for entry in results.get("entry", []):
    print("New FHIR ID:", entry["resource"]["id"])
```

Update `PATIENT_ID` in `tests/test_sandbox.py` and re-run.

---

## Sandbox Limitations

| Limitation | What to Do |
|-----------|-----------|
| Data resets without notice | Re-run `Patient.Search` to get fresh FHIR IDs |
| Sparse clinical data | Some resources return `total=0` — expected |
| FHIR IDs are not stable | Never hardcode them in production logic |
| No write persistence | POSTed resources may disappear after a reset |
| Not for performance testing | Do not run load or stress tests against sandbox |
| Synthetic data only | PHI must never be entered — Epic policy requirement |

---

## Checklist Before Moving to Production

- [ ] `tests/test_sandbox.py` runs with 0 failures
- [ ] All resource searches return valid `Bundle` responses
- [ ] Empty bundles are handled gracefully in all code paths
- [ ] No real patient data has been entered into the sandbox
- [ ] FHIR IDs are not hardcoded — always looked up dynamically

---

## Next Step

**[Step 8 → Common Errors & Troubleshooting](./08_common_errors.md)**

---

## Reference Links

| Resource | URL |
|----------|-----|
| Sandbox Test Data Docs | `https://fhir.epic.com/Documentation?docId=testpatients` |
| Developer Testing Guide | `https://fhir.epic.com/Documentation?docId=testingguidance` |
| Epic FHIR Portal | https://fhir.epic.com |
| API Specifications | https://fhir.epic.com/Specifications |
