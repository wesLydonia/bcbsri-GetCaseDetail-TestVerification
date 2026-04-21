# Test Case Creation Plan
## Project: GetCaseDetailTestingVerification
**Client:** BCBSRI | **Author:** Wes Hartmann — Lydonia Technologies  
**Version:** 1.0 | **Date:** April 21, 2026 | **Status:** Draft

---

## 1. Purpose

This document is a hands-on, step-by-step plan for building the full test case suite for the **GetCaseDetailTestingVerification** project. It answers:

- **What** test case files need to be created
- **What** each file must verify
- **What** supporting assets (test data, helpers) are needed
- **In what order** to build them
- **How** to validate they are working correctly

By the end of this plan, every workflow in the project will have automated test coverage.

---

## 2. Current State

### 2.1 Workflows That Need Test Coverage

| Workflow | Purpose | Test Coverage Today |
|---|---|---|
| `GetAccessToken.xaml` | Acquires Salesforce OAuth Bearer token | ❌ None |
| `FindCaseByAttachment.xaml` | Resolves attachment filename → CaseId | ❌ None |
| `GetCaseDetail.xaml` | Retrieves full Case fields + derives Classification | ✅ Partial (`Test_GetCaseDetail.xaml`) |
| `Main.xaml` | Orchestrates end-to-end flow, populates all outputs | ❌ None |

### 2.2 Test Files That Already Exist

| File | Registered as Test Case | What It Covers |
|---|---|---|
| `Test_GetCaseDetail.xaml` | ✅ Yes (`project.json`) | GetCaseDetail invocation, DataTable population, Classification non-null, Classification value, CaseId round-trip |

### 2.3 What Is Still Missing

| Gap | Impact |
|---|---|
| No test for token acquisition | Cannot confirm auth works in isolation |
| No test for attachment lookup | Cannot confirm attachment → CaseId resolution in isolation |
| No end-to-end test through `Main.xaml` | Cannot confirm output arguments are all populated correctly |
| No negative/error path tests | No coverage for empty inputs, missing Cases, or API failures |
| No test data variations file | `Test_GetCaseDetail.xaml` only runs one scenario at a time |
| `ExpectedClassification` is logged but not asserted | Classification correctness is not automatically enforced |

---

## 3. Test Case File Inventory (Target State)

When this plan is complete, the following test files will exist:

| # | File | Tests | Priority |
|---|---|---|---|
| 1 | `Test_GetAccessToken.xaml` *(new)* | Token is non-empty; token can authenticate a real API call | 🔴 High |
| 2 | `Test_FindCaseByAttachment.xaml` *(new)* | Known attachment resolves to correct CaseId; unknown attachment returns `Found=False` | 🔴 High |
| 3 | `Test_GetCaseDetail.xaml` *(enhance existing)* | Add `ExpectedClassification` assertion; add HTTP status code check | 🔴 High |
| 4 | `Test_Main_HappyPath.xaml` *(new)* | End-to-end: valid attachment → all 5 output arguments populated correctly | 🔴 High |
| 5 | `Test_Main_NotFound.xaml` *(new)* | End-to-end: unknown attachment → `out_Found=False`, no exception | 🟡 Medium |
| 6 | `Test_Main_InvalidInput.xaml` *(new)* | Empty / null `in_AttachmentName` → `BusinessRuleException` thrown | 🟡 Medium |
| 7 | `TestData_GetCaseDetail.json` *(new)* | Data variation file: multiple Case IDs run through `Test_GetCaseDetail.xaml` | 🟡 Medium |

---

## 4. Supporting Assets to Create First

Before building the test case workflows, create these shared assets. They are dependencies for multiple test cases.

---

### Asset A — Test Data Spreadsheet / Reference Sheet

**Purpose:** A simple reference that maps known attachment filenames to expected CaseId, CaseNumber, and Classification. This is the source of truth for all test case inputs.

**Format:**

| TestName | AttachmentName | CaseId | CaseNumber | ExpectedClassification |
|---|---|---|---|---|
| Clinical - FAX 1 | faxreceive.2026-04-17-16-24-27_5B8.pdf | *(from BCBSRI)* | *(from BCBSRI)* | Clinical |
| Non-Clinical - FAX 2 | *(from BCBSRI)* | *(from BCBSRI)* | *(from BCBSRI)* | Non-Clinical |
| Not Found | nonexistent_99999.pdf | *(N/A)* | *(N/A)* | *(N/A)* |

**Action:** Gather this data from BCBSRI SMEs before any test case creation begins.  
**Owner:** BCBSRI Case Management Team + Wes Hartmann  
**Blocks:** TC-1, TC-2, TC-3, TC-4, TC-5, and the data variation file

---

### Asset B — `TestData_GetCaseDetail.json`

**Purpose:** A UiPath test data variation file that allows `Test_GetCaseDetail.xaml` to run multiple Case scenarios in a single test execution — one row per Case.

**Format (must be a homogeneous JSON array — all objects must have the same keys):**

```json
[
  {
    "TestName": "Clinical - FAX 1",
    "CaseId": "<18-char-Salesforce-Id>",
    "CaseNumber": "00123456",
    "ExpectedClassification": "Clinical"
  },
  {
    "TestName": "Non-Clinical - FAX 2",
    "CaseId": "<18-char-Salesforce-Id>",
    "CaseNumber": "00234567",
    "ExpectedClassification": "Non-Clinical"
  }
]
```

**Rules:**
- All values must be primitives (String, Number, Boolean) — no nested objects or arrays
- All objects must have identical keys
- File must be linked to `Test_GetCaseDetail.xaml` via **Add Test Data Variation** in Studio

**Action:** Create file once Asset A data is confirmed. Then right-click `Test_GetCaseDetail.xaml` → **Add Test Data Variation** → select this file.  
**Owner:** Lydonia  
**Blocks:** TC-3 (enhanced GetCaseDetail test with data variations)

---

## 5. Test Case Build Instructions

Each section below is a complete build specification for one test case file. Follow them in order.

---

### TC-1 — `Test_GetAccessToken.xaml`

**Goal:** Verify that `GetAccessToken.xaml` returns a valid, non-empty Salesforce OAuth token.

**Arguments to Define:**

| Name | Direction | Type | Description |
|---|---|---|---|
| *(none required)* | — | — | No inputs needed; token is acquired from credentials inside GetAccessToken |

**Variables to Define:**

| Name | Type | Purpose |
|---|---|---|
| `accessToken` | String | Stores the token returned by GetAccessToken |
| `tokenVerifyResponse` | HttpResponseSummary | Stores the result of a test API call using the token |

**Activities to Add (in order):**

1. **Log Message** — `"=== TEST: GetAccessToken START ==="`
2. **Invoke Workflow File** → `GetAccessToken.xaml`
   - Out: `out_AccessToken` → `accessToken`
3. **Verify Expression** — `Not String.IsNullOrEmpty(accessToken)`
   - Display Name: `Verify: Token is not empty`
   - ContinueOnFailure: True
4. **Verify Expression** — `accessToken.Length > 20`
   - Display Name: `Verify: Token has realistic length`
   - ContinueOnFailure: True
5. **HTTP Request (NetHttpRequest)** → `GET https://bcbsrihub.my.salesforce.com/services/data/v64.0/limits`
   - Header: `Authorization: Bearer {accessToken}`
   - Result: `tokenVerifyResponse`
   - ContinueOnError: True
6. **Verify Expression** — `tokenVerifyResponse IsNot Nothing AndAlso tokenVerifyResponse.StatusCode = 200`
   - Display Name: `Verify: Token authenticates successfully (HTTP 200)`
   - ContinueOnFailure: True
7. **Log Message** — `"Token length=" & accessToken.Length & " | HTTP Status=" & tokenVerifyResponse.StatusCode.ToString()`
8. **Log Message** — `"=== TEST: GetAccessToken END ==="`

**Pass Criteria:** All 3 VerifyExpressions pass. HTTP 200 from Salesforce `/limits` endpoint confirms the token is accepted.

**Register as Test Case:** Yes — ensure `project.json` `fileInfoCollection` entry is added.

---

### TC-2 — `Test_FindCaseByAttachment.xaml`

**Goal:** Verify that `FindCaseByAttachment.xaml` correctly resolves a known attachment filename to the expected CaseId, and returns `Found=False` for an unknown attachment.

**Arguments to Define:**

| Name | Direction | Type | Description |
|---|---|---|---|
| `AttachmentName` | In | String | The attachment filename to search for |
| `ExpectedCaseId` | In | String | The expected CaseId (empty string for not-found scenarios) |
| `ExpectedFound` | In | Boolean | Whether a match is expected |

**Variables to Define:**

| Name | Type | Purpose |
|---|---|---|
| `accessToken` | String | OAuth token |
| `outCaseId` | String | CaseId returned by FindCaseByAttachment |
| `outMatchMethod` | String | MatchMethod returned |
| `outFound` | Boolean | Found flag returned |

**Activities to Add (in order):**

1. **Log Message** — `"=== TEST: FindCaseByAttachment START — AttachmentName=" & AttachmentName`
2. **Invoke Workflow File** → `GetAccessToken.xaml`
   - Out: `out_AccessToken` → `accessToken`
3. **Invoke Workflow File** → `FindCaseByAttachment.xaml`
   - In: `in_AccessToken` → `accessToken`
   - In: `in_DataApiUrl` → `"https://bcbsrihub.my.salesforce.com/services/data/v64.0"`
   - In: `in_AttachmentName` → `AttachmentName`
   - Out: `out_CaseId` → `outCaseId`
   - Out: `out_MatchMethod` → `outMatchMethod`
   - Out: `out_Found` → `outFound`
4. **Verify Expression** — `outFound = ExpectedFound`
   - Display Name: `Verify: Found flag matches expected`
   - ContinueOnFailure: True
5. **If** `ExpectedFound = True`
   - **Then:**
     - **Verify Expression** — `Not String.IsNullOrEmpty(outCaseId)`
       - Display Name: `Verify: CaseId is not empty when found`
       - ContinueOnFailure: True
     - **Verify Expression** — `outCaseId = ExpectedCaseId`
       - Display Name: `Verify: CaseId matches expected value`
       - ContinueOnFailure: True
     - **Verify Expression** — `Not String.IsNullOrEmpty(outMatchMethod)`
       - Display Name: `Verify: MatchMethod is not empty when found`
       - ContinueOnFailure: True
   - **Else:**
     - **Verify Expression** — `String.IsNullOrEmpty(outCaseId)`
       - Display Name: `Verify: CaseId is empty when not found`
       - ContinueOnFailure: True
6. **Log Message** — `"Found=" & outFound.ToString() & " | CaseId=" & If(outCaseId, "") & " | MatchMethod=" & If(outMatchMethod, "")`
7. **Log Message** — `"=== TEST: FindCaseByAttachment END ==="`

**Test Data Scenarios (to be run as data variations or separate executions):**

| Scenario | AttachmentName | ExpectedCaseId | ExpectedFound |
|---|---|---|---|
| Happy Path | *(known filename from Asset A)* | *(known CaseId from Asset A)* | True |
| Not Found | `nonexistent_99999.pdf` | `""` | False |

**Pass Criteria:** All VerifyExpressions pass for both scenarios.

---

### TC-3 — Enhance `Test_GetCaseDetail.xaml` (Existing)

**Goal:** Add the missing `ExpectedClassification` assertion and an HTTP status code check to the existing test case.

**Current State:** Has 5 VerifyExpressions but `ExpectedClassification` is only logged, not asserted.

**Changes to Make:**

1. **After** `Verify: Classification column in row matches output` — add:
   - **Verify Expression** — `classification = ExpectedClassification`
     - Display Name: `Verify: Classification matches SME-expected value`
     - ContinueOnFailure: True
     - *(Only enable this once BCBSRI SMEs have confirmed the ExpectedClassification values in Asset A)*

2. **After** the `Invoke GetCaseDetail` — add:
   - **Verify Expression** — `caseDetailResponse IsNot Nothing AndAlso caseDetailResponse.StatusCode = 200`
     - Display Name: `Verify: HTTP 200 from Salesforce Case detail`
     - ContinueOnFailure: True

3. **Link TestData_GetCaseDetail.json** (Asset B) to enable multi-scenario execution via data variations.

**Resulting Total Verifications:** 7 per execution (V-001 through V-007)

---

### TC-4 — `Test_Main_HappyPath.xaml`

**Goal:** Verify the full end-to-end flow through `Main.xaml` when a valid attachment filename is provided. Confirms all 5 output arguments are populated.

**Arguments to Define:**

| Name | Direction | Type | Description |
|---|---|---|---|
| `AttachmentName` | In | String | Known attachment filename |
| `ExpectedCaseId` | In | String | Expected CaseId for this attachment |
| `ExpectedCaseNumber` | In | String | Expected CaseNumber |

**Variables to Define:**

| Name | Type | Purpose |
|---|---|---|
| `outCaseId` | String | from Main: out_CaseId |
| `outCaseNumber` | String | from Main: out_CaseNumber |
| `outCaseJson` | String | from Main: out_CaseJson |
| `outFound` | Boolean | from Main: out_Found |
| `outMatchMethod` | String | from Main: out_MatchMethod |

**Activities to Add (in order):**

1. **Log Message** — `"=== TEST: Main HappyPath START — AttachmentName=" & AttachmentName`
2. **Invoke Workflow File** → `Main.xaml`
   - In: `in_AttachmentName` → `AttachmentName`
   - Out: `out_CaseId` → `outCaseId`
   - Out: `out_CaseNumber` → `outCaseNumber`
   - Out: `out_CaseJson` → `outCaseJson`
   - Out: `out_Found` → `outFound`
   - Out: `out_MatchMethod` → `outMatchMethod`
3. **Verify Expression** — `outFound = True`
   - Display Name: `Verify: Case was found`
   - ContinueOnFailure: True
4. **Verify Expression** — `Not String.IsNullOrEmpty(outCaseId)`
   - Display Name: `Verify: CaseId is populated`
   - ContinueOnFailure: True
5. **Verify Expression** — `outCaseId = ExpectedCaseId`
   - Display Name: `Verify: CaseId matches expected`
   - ContinueOnFailure: True
6. **Verify Expression** — `Not String.IsNullOrEmpty(outCaseNumber)`
   - Display Name: `Verify: CaseNumber is populated`
   - ContinueOnFailure: True
7. **Verify Expression** — `outCaseNumber = ExpectedCaseNumber`
   - Display Name: `Verify: CaseNumber matches expected`
   - ContinueOnFailure: True
8. **Verify Expression** — `Not String.IsNullOrEmpty(outCaseJson)`
   - Display Name: `Verify: CaseJson is populated`
   - ContinueOnFailure: True
9. **Verify Expression** — `outCaseJson.Contains("CaseNumber")`
   - Display Name: `Verify: CaseJson contains CaseNumber field`
   - ContinueOnFailure: True
10. **Verify Expression** — `Not String.IsNullOrEmpty(outMatchMethod)`
    - Display Name: `Verify: MatchMethod is populated`
    - ContinueOnFailure: True
11. **Log Message** — `"Found=" & outFound.ToString() & " | CaseId=" & If(outCaseId,"") & " | CaseNumber=" & If(outCaseNumber,"") & " | MatchMethod=" & If(outMatchMethod,"")`
12. **Log Message** — `"=== TEST: Main HappyPath END ==="`

**Pass Criteria:** All 8 VerifyExpressions pass.

---

### TC-5 — `Test_Main_NotFound.xaml`

**Goal:** Verify that `Main.xaml` handles the "no matching Case" scenario gracefully — returns `out_Found=False` without throwing an exception.

**Arguments to Define:** *(none — use a hardcoded unknown filename)*

**Variables to Define:**

| Name | Type | Purpose |
|---|---|---|
| `outFound` | Boolean | from Main |
| `outCaseId` | String | from Main |
| `outCaseJson` | String | from Main |

**Activities to Add (in order):**

1. **Log Message** — `"=== TEST: Main NotFound START ==="`
2. **Try/Catch** *(outer wrapper to catch any unexpected exceptions)*
   - **Try:**
     - **Invoke Workflow File** → `Main.xaml`
       - In: `in_AttachmentName` → `"nonexistent_test_file_99999.pdf"`
       - Out: `out_Found` → `outFound`
       - Out: `out_CaseId` → `outCaseId`
       - Out: `out_CaseJson` → `outCaseJson`
     - **Verify Expression** — `outFound = False`
       - Display Name: `Verify: Not-found flag is False`
       - ContinueOnFailure: True
     - **Verify Expression** — `String.IsNullOrEmpty(outCaseId)`
       - Display Name: `Verify: CaseId is empty when not found`
       - ContinueOnFailure: True
     - **Verify Expression** — `String.IsNullOrEmpty(outCaseJson)`
       - Display Name: `Verify: CaseJson is empty when not found`
       - ContinueOnFailure: True
   - **Catch** `Exception`
     - **Verify Expression** — `False`
       - Display Name: `Verify: No exception should be thrown`
       - ContinueOnFailure: True
     - **Log Message** — `"UNEXPECTED EXCEPTION: " & exception.Message`
3. **Log Message** — `"=== TEST: Main NotFound END ==="`

**Pass Criteria:** No exception is thrown; all 3 VerifyExpressions pass; `outFound = False`.

---

### TC-6 — `Test_Main_InvalidInput.xaml`

**Goal:** Verify that `Main.xaml` throws a `BusinessRuleException` when `in_AttachmentName` is null, empty, or whitespace.

**Arguments to Define:** *(none — test data is hardcoded per scenario)*

**Variables to Define:**

| Name | Type | Purpose |
|---|---|---|
| `exceptionThrown` | Boolean | Tracks whether the expected exception was caught |
| `exceptionMessage` | String | Stores the exception message |

**Activities to Add (in order):**

1. **Log Message** — `"=== TEST: Main InvalidInput START ==="`
2. **Assign** — `exceptionThrown = False`
3. **Try/Catch**
   - **Try:**
     - **Invoke Workflow File** → `Main.xaml`
       - In: `in_AttachmentName` → `""` *(or test with `"   "` for whitespace)*
   - **Catch** `BusinessRuleException`
     - **Assign** — `exceptionThrown = True`
     - **Assign** — `exceptionMessage = exception.Message`
   - **Catch** `Exception`
     - **Assign** — `exceptionThrown = True`
     - **Assign** — `exceptionMessage = "WRONG type: " & exception.Message`
4. **Verify Expression** — `exceptionThrown = True`
   - Display Name: `Verify: An exception was thrown for empty input`
   - ContinueOnFailure: True
5. **Verify Expression** — `exceptionMessage.Contains("in_AttachmentName is required")`
   - Display Name: `Verify: Exception message is correct`
   - ContinueOnFailure: True
6. **Log Message** — `"ExceptionThrown=" & exceptionThrown.ToString() & " | Message=" & If(exceptionMessage, "(none)")`
7. **Log Message** — `"=== TEST: Main InvalidInput END ==="`

**Pass Criteria:** Both VerifyExpressions pass; `exceptionThrown = True`; exception message matches `"in_AttachmentName is required."`.

---

## 6. Build Order & Dependencies

Follow this sequence. Each step either produces an artifact or unblocks a downstream step.

```
Step 1 ── Gather test data from BCBSRI SMEs (Asset A)
            └── Unblocks: all test case inputs

Step 2 ── Create TestData_GetCaseDetail.json (Asset B)
            └── Unblocks: TC-3 data variation linking

Step 3 ── Create Test_GetAccessToken.xaml (TC-1)
            └── Verifies auth works before touching any other workflow
            └── Unblocks: confidence to proceed with API-dependent tests

Step 4 ── Create Test_FindCaseByAttachment.xaml (TC-2)
            └── Verifies the attachment lookup layer in isolation

Step 5 ── Enhance Test_GetCaseDetail.xaml (TC-3)
            └── Add HTTP status assertion + ExpectedClassification assertion
            └── Link TestData_GetCaseDetail.json

Step 6 ── Create Test_Main_HappyPath.xaml (TC-4)
            └── Verifies end-to-end integration of all sub-workflows

Step 7 ── Create Test_Main_NotFound.xaml (TC-5)
            └── Verifies graceful not-found handling

Step 8 ── Create Test_Main_InvalidInput.xaml (TC-6)
            └── Verifies guard clause / input validation

Step 9 ── Run full suite; review Classification log output with BCBSRI SMEs
            └── Produces final test results for sign-off
```

---

## 7. Verification Summary (Full Suite)

Once all test cases are built, the complete suite will contain:

| Test File | Scenarios | VerifyExpressions |
|---|---|---|
| `Test_GetAccessToken.xaml` | 1 | 3 |
| `Test_FindCaseByAttachment.xaml` | 2 (found + not found) | 5 |
| `Test_GetCaseDetail.xaml` (enhanced) | ≥2 via data variations | 7 per scenario |
| `Test_Main_HappyPath.xaml` | 1 | 8 |
| `Test_Main_NotFound.xaml` | 1 | 3 |
| `Test_Main_InvalidInput.xaml` | 1 | 2 |
| **Total** | **8+** | **28+** |

---

## 8. Studio Registration Checklist

After creating each test file, confirm it is registered as a Test Case in Studio:

- [ ] `Test_GetAccessToken.xaml` → right-click → **Set as Test Case** *(or confirm entry in `project.json` → `fileInfoCollection`)*
- [ ] `Test_FindCaseByAttachment.xaml` → **Set as Test Case**
- [ ] `Test_GetCaseDetail.xaml` → already registered ✅
- [ ] `Test_Main_HappyPath.xaml` → **Set as Test Case**
- [ ] `Test_Main_NotFound.xaml` → **Set as Test Case**
- [ ] `Test_Main_InvalidInput.xaml` → **Set as Test Case**
- [ ] `TestData_GetCaseDetail.json` → **Add Test Data Variation** → linked to `Test_GetCaseDetail.xaml`

---

## 9. Definition of Done

The test case creation effort is complete when:

- [ ] All 6 test case `.xaml` files exist and compile without errors
- [ ] All test cases are registered in Studio's Test Results panel
- [ ] `TestData_GetCaseDetail.json` is linked to `Test_GetCaseDetail.xaml`
- [ ] A full test suite run produces results with no **Critical** failures
- [ ] BCBSRI SMEs have reviewed the Classification log output and confirmed values
- [ ] `Verify: Classification matches SME-expected value` in TC-3 is enabled and passing

---

*Document generated by Autopilot — Lydonia Technologies | Session: c441dd16-a543-4257-878c-5a7cd0b379f1*
