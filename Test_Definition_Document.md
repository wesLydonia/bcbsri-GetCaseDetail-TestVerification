# Test Definition Document (TDD)
## Project: GetCaseDetailTestingVerification
**Client:** BCBSRI  
**Author:** Wes Hartmann — Lydonia Technologies  
**Version:** 1.0.0  
**Date:** April 21, 2026  
**Studio Version:** 26.0.189.0 | **Framework:** Windows | **Language:** VB.NET  

---

## 1. Overview

### 1.1 Purpose
This document defines the testing strategy, test cases, verifications, and traceability matrix for the **GetCaseDetailTestingVerification** automation project. The project is a standalone testing and verification utility that:

1. Accepts a Salesforce attachment file name as input.
2. Locates the Salesforce Case associated with that attachment.
3. Retrieves full Case field details via the Salesforce REST API.
4. Classifies the Case as **Clinical** or **Non-Clinical** based on its field values.

### 1.2 Scope
| In Scope | Out of Scope |
|---|---|
| OAuth token acquisition (GetAccessToken) | Salesforce UI automation |
| Attachment-to-Case lookup (FindCaseByAttachment) | Bulk/batch processing |
| Case detail retrieval via REST API (GetCaseDetail) | Data write-back to Salesforce |
| Case classification logic | End-to-end UI regression testing |
| Output argument population (Main) | Performance/load testing |

### 1.3 System Under Test
- **Target System:** Salesforce (bcbsrihub.my.salesforce.com)
- **API Version:** Salesforce REST API v64.0
- **Authentication:** OAuth 2.0 (Bearer token)

---

## 2. Project Architecture

### 2.1 Workflow Inventory
| Workflow | Type | Role |
|---|---|---|
| `Main.xaml` | Entry Point | Orchestrates the full end-to-end flow |
| `GetAccessToken.xaml` | Sub-Workflow | Acquires Salesforce OAuth Bearer token |
| `FindCaseByAttachment.xaml` | Sub-Workflow | Queries Salesforce to find CaseId by attachment filename |
| `GetCaseDetail.xaml` | Sub-Workflow | Retrieves full Case field data and derives Classification |
| `Test_GetCaseDetail.xaml` | Test Case | Verifies GetCaseDetail output against known data |

### 2.2 Dependencies
| Package | Version |
|---|---|
| UiPath.System.Activities | 25.12.2 |
| UiPath.Testing.Activities | 25.10.1 |
| UiPath.WebAPI.Activities | 2.4.0 |

---

## 3. Workflow Interface Specifications

### 3.1 Main.xaml

#### Input Arguments
| Argument | Type | Required | Description |
|---|---|---|---|
| `in_AttachmentName` | String | ✅ Yes | The attachment file name to search for (e.g., `faxreceive.2026-04-17-16-24-27_5B8.pdf`) |

#### Output Arguments
| Argument | Type | Description |
|---|---|---|
| `out_CaseId` | String | Salesforce internal Case ID (18-char) |
| `out_CaseNumber` | String | Human-readable Case Number (e.g., 00123456) |
| `out_CaseJson` | String | Full Case record as pretty-printed JSON |
| `out_Found` | Boolean | True if a matching Case was found; False otherwise |
| `out_MatchMethod` | String | Describes how the match was made (e.g., exact, partial) |

#### Internal Variables
| Variable | Type | Description |
|---|---|---|
| `accessToken` | String | OAuth Bearer token |
| `dataApiUrl` | String | Base Salesforce API URL (hardcoded: `https://bcbsrihub.my.salesforce.com/services/data/v64.0`) |
| `caseId` | String | Internal CaseId from FindCaseByAttachment |
| `matchMethod` | String | Match method from FindCaseByAttachment |
| `found` | Boolean | Found flag from FindCaseByAttachment |
| `caseResponse` | HttpResponseSummary | Raw HTTP response from Case detail GET |
| `caseJsonText` | String | Parsed/formatted JSON body of Case |
| `caseNumberLocal` | String | CaseNumber extracted from JSON |

### 3.2 GetAccessToken.xaml
| Direction | Argument | Type | Description |
|---|---|---|---|
| Out | `out_AccessToken` | String | Salesforce OAuth 2.0 Bearer token |

### 3.3 FindCaseByAttachment.xaml
| Direction | Argument | Type | Description |
|---|---|---|---|
| In | `in_AccessToken` | String | Salesforce Bearer token |
| In | `in_DataApiUrl` | String | Salesforce REST API base URL |
| In | `in_AttachmentName` | String | Attachment file name to search for |
| Out | `out_CaseId` | String | Matching Salesforce Case ID |
| Out | `out_MatchMethod` | String | Method used to find the Case |
| Out | `out_Found` | Boolean | Whether a Case was found |

### 3.4 GetCaseDetail.xaml
| Direction | Argument | Type | Description |
|---|---|---|---|
| In | `in_AccessToken` | String | Salesforce Bearer token |
| In | `in_CaseId` | String | Salesforce Case ID to retrieve |
| In | `in_CaseNumber` | String | Human-readable Case Number |
| In/Out | `io_DtCases` | DataTable | DataTable populated with Case field data |
| Out | `out_Classification` | String | Derived classification: "Clinical" or "Non-Clinical" |
| Out | `out_caseDetailResponse` | HttpResponseSummary | Raw HTTP response |

#### DataTable Schema (`io_DtCases`)
| Column | Source |
|---|---|
| CaseId | Case.Id |
| CaseNumber | Case.CaseNumber |
| Status | Case.Status |
| Origin | Case.Origin |
| Subject | Case.Subject |
| Priority | Case.Priority |
| RecordTypeId | Case.RecordTypeId |
| OwnerId | Case.OwnerId |
| OwnerName | Case.Owner.Name |
| OwnerRole | Case.Owner.UserRole.Name |
| EntitlementId | Case.EntitlementId |
| CreatedDate | Case.CreatedDate |
| LastModifiedDate | Case.LastModifiedDate |
| SlaStartDate | Case.SlaStartDate |
| IsClosed | Case.IsClosed |
| IsDeleted | Case.IsDeleted |
| SuppliedName | Case.SuppliedName |
| SuppliedEmail | Case.SuppliedEmail |
| CC_GAU_Case_Type__c | Custom field |
| CC_GAU_Case_Sub_Type__c | Custom field |
| CC_GAU_Team__c | Custom field |
| CC_Category__c | Custom field |
| CC_Final_Category__c | Custom field |
| CC_Case_Due_Date__c | Custom field |
| CC_Case_Age_in_Days__c | Custom field |
| CC_Determination__c | Custom field |
| CC_Outcome__c | Custom field |
| CC_Type__c | Custom field |
| CC_Sub_Status__c | Custom field |
| CC_Master_SLA_Status__c | Custom field |
| CC_Case_Owner_s_Department__c | Custom field |
| Description | Case.Description |
| Classification | Derived by automation logic |

---

## 4. Test Strategy

### 4.1 Testing Approach
- **Framework:** UiPath Testing Activities (`VerifyExpression`)
- **Test Runner:** UiPath Studio / Orchestrator Test Execution
- **Test Type:** Functional Integration Tests (live Salesforce sandbox/production calls)
- **Execution Mode:** Local (unattended-compatible)
- **Continue on Failure:** Enabled (`ContinueOnFailure = True`) on all verifications so all checks run and failures are reported rather than halting execution

### 4.2 Test Data Strategy
Test cases use known, stable Salesforce Case records with pre-confirmed Classification values. The `ExpectedClassification` argument serves as the SME-validated oracle value for comparison — logged for human review since the classification logic may be evolving.

### 4.3 Test Environment
| Setting | Value |
|---|---|
| Salesforce Org | bcbsrihub.my.salesforce.com |
| Auth Method | OAuth 2.0 (credentials managed by GetAccessToken) |
| API Version | v64.0 |
| HTTP Timeout | 20,000 ms |
| HTTP Retry Count | 2 |
| Retry Policy | Basic (with jitter) |
| Retry Status Codes | 408, 429, 500, 502, 503, 504 |

---

## 5. Test Cases

### TC-001 — GetCaseDetail: Happy Path (Known Clinical Case)
**Workflow:** `Test_GetCaseDetail.xaml`  
**Objective:** Verify that GetCaseDetail correctly retrieves data and classifies a known Clinical Case.

#### Given / When / Then
| Phase | Detail |
|---|---|
| **Given** | A valid `CaseId` and `CaseNumber` for a known Clinical Salesforce Case |
| **When** | `GetAccessToken` is invoked to obtain a Bearer token, then `GetCaseDetail` is invoked with that token and the Case identifiers |
| **Then** | The output DataTable (`dtCases`) contains exactly 1 row, Classification is non-empty, Classification is "Clinical" or "Non-Clinical", the CaseId in `dtCases` matches the input, and the Classification column in `dtCases` matches `out_Classification` |

#### Input Arguments
| Argument | Example Value |
|---|---|
| `TestName` | "Clinical Case - Happy Path" |
| `CaseId` | _(18-char Salesforce Case ID for a known Clinical case)_ |
| `CaseNumber` | _(e.g., "00123456")_ |
| `ExpectedClassification` | "Clinical" |

#### Verifications
| ID | Assertion | Activity | Continue on Failure |
|---|---|---|---|
| V-001 | `dtCases IsNot Nothing AndAlso dtCases.Rows.Count = 1` | Verify: dtCases has 1 row | ✅ |
| V-002 | `Not String.IsNullOrEmpty(classification)` | Verify: Classification is not empty | ✅ |
| V-003 | `classification = "Clinical" OrElse classification = "Non-Clinical"` | Verify: Classification is Clinical or Non-Clinical | ✅ |
| V-004 | `dtCases.Rows(0)("CaseId").ToString() = CaseId` | Verify: CaseId in dtCases row matches input | ✅ |
| V-005 | `dtCases.Rows(0)("Classification").ToString() = classification` | Verify: Classification column in row matches output | ✅ |

#### Expected Result
- All 5 verifications pass ✅
- Log shows: `CLASSIFICATION RESULT: Case XXXXX classified as: Clinical`

---

### TC-002 — GetCaseDetail: Happy Path (Known Non-Clinical Case)
**Workflow:** `Test_GetCaseDetail.xaml`  
**Objective:** Verify GetCaseDetail correctly classifies a known Non-Clinical Case.

#### Input Arguments
| Argument | Example Value |
|---|---|
| `TestName` | "Non-Clinical Case - Happy Path" |
| `CaseId` | _(18-char Salesforce Case ID for a known Non-Clinical case)_ |
| `CaseNumber` | _(e.g., "00234567")_ |
| `ExpectedClassification` | "Non-Clinical" |

#### Verifications
Same 5 assertions as TC-001 (V-001 through V-005).

#### Expected Result
- All 5 verifications pass ✅
- Log shows: `CLASSIFICATION RESULT: Case XXXXX classified as: Non-Clinical`

---

### TC-003 — Main: End-to-End Happy Path (Attachment Found)
**Workflow:** `Main.xaml`  
**Objective:** Verify the full end-to-end flow when a valid attachment filename is provided and a matching Case exists.

#### Input Arguments
| Argument | Value |
|---|---|
| `in_AttachmentName` | `faxreceive.2026-04-17-16-24-27_5B8.pdf` _(or any known attachment)_ |

#### Expected Outputs
| Output | Expected |
|---|---|
| `out_Found` | `True` |
| `out_CaseId` | Non-empty, 18-character Salesforce ID |
| `out_CaseNumber` | Non-empty, numeric Case Number string |
| `out_CaseJson` | Non-empty, valid JSON string containing `"CaseNumber"` |
| `out_MatchMethod` | Non-empty string (e.g., "exact" or "partial") |

#### Pass Criteria
- `out_Found = True`
- `out_CaseId` is not null or empty
- `out_CaseNumber` is not null or empty
- `out_CaseJson` is valid parseable JSON
- `out_MatchMethod` is not null or empty
- Log includes: `GetCaseDetailByAttachment END — Found=True`

---

### TC-004 — Main: Attachment Not Found
**Workflow:** `Main.xaml`  
**Objective:** Verify the automation gracefully handles the scenario where no Case matches the given attachment filename.

#### Input Arguments
| Argument | Value |
|---|---|
| `in_AttachmentName` | `nonexistent_file_99999.pdf` |

#### Expected Outputs
| Output | Expected |
|---|---|
| `out_Found` | `False` |
| `out_CaseId` | Empty string or null |
| `out_CaseNumber` | Empty string or null |
| `out_CaseJson` | Empty string or null |

#### Pass Criteria
- `out_Found = False`
- No exception is thrown
- Log includes warning: `No case found for attachment 'nonexistent_file_99999.pdf'`

---

### TC-005 — Main: Invalid / Empty Input (Guard Clause)
**Workflow:** `Main.xaml`  
**Objective:** Verify that the automation throws a `BusinessRuleException` when `in_AttachmentName` is null or whitespace.

#### Input Arguments
| Scenario | `in_AttachmentName` |
|---|---|
| Null input | `Nothing` |
| Empty string | `""` |
| Whitespace only | `"   "` |

#### Expected Result
- A `BusinessRuleException` is thrown with message: `"in_AttachmentName is required."`
- Execution stops at the validation guard clause
- No API calls are made

---

### TC-006 — GetAccessToken: Token Acquisition
**Workflow:** `GetAccessToken.xaml`  
**Objective:** Verify that a non-empty, valid Salesforce OAuth Bearer token is returned.

#### Expected Result
- `out_AccessToken` is not null or empty
- Token can be used in a subsequent API call without a 401 Unauthorized response

---

### TC-007 — FindCaseByAttachment: Returns Correct CaseId
**Workflow:** `FindCaseByAttachment.xaml`  
**Objective:** Verify that a known attachment filename resolves to the correct CaseId.

#### Input Arguments
| Argument | Value |
|---|---|
| `in_AccessToken` | _(valid token from GetAccessToken)_ |
| `in_DataApiUrl` | `https://bcbsrihub.my.salesforce.com/services/data/v64.0` |
| `in_AttachmentName` | _(known attachment filename)_ |

#### Expected Outputs
| Output | Expected |
|---|---|
| `out_Found` | `True` |
| `out_CaseId` | The expected 18-char Salesforce Case ID |
| `out_MatchMethod` | Non-empty string |

---

## 6. Negative / Edge Case Tests

| TC-ID | Scenario | Input | Expected Behavior |
|---|---|---|---|
| TC-NEG-001 | Attachment filename with special characters | `case (copy) [final] #1.pdf` | No exception; graceful not-found or match |
| TC-NEG-002 | Very long attachment filename (>255 chars) | _(255+ char string)_ | No exception; graceful handling |
| TC-NEG-003 | Salesforce API returns 401 Unauthorized | Expired/invalid token | Exception or ContinueOnError fallback; `out_Found = False` |
| TC-NEG-004 | Salesforce API returns 503 Service Unavailable | Network disruption | Retry policy activates (2 retries); graceful failure |
| TC-NEG-005 | Case JSON response body is empty | API returns 200 but empty body | `out_CaseJson = ""`, no crash; `out_Found = True`, `out_CaseNumber = ""` |
| TC-NEG-006 | CaseId resolves but Case has been deleted | `IsDeleted = True` | Data returned as-is; `dtCases` has 1 row |

---

## 7. Verification Traceability Matrix

| Verification ID | Description | Workflow | Activity ID | Covered By |
|---|---|---|---|---|
| V-001 | dtCases has exactly 1 row | Test_GetCaseDetail.xaml | Verify_1 | TC-001, TC-002 |
| V-002 | Classification output is not empty | Test_GetCaseDetail.xaml | Verify_2 | TC-001, TC-002 |
| V-003 | Classification is "Clinical" or "Non-Clinical" | Test_GetCaseDetail.xaml | Verify_3 | TC-001, TC-002 |
| V-004 | CaseId in DataTable matches input CaseId | Test_GetCaseDetail.xaml | Verify_4 | TC-001, TC-002 |
| V-005 | Classification column in DataTable matches out_Classification | Test_GetCaseDetail.xaml | Verify_5 | TC-001, TC-002 |
| V-006 | out_Found = True for known attachment | Main.xaml | _(manual)_ | TC-003 |
| V-007 | out_CaseId / out_CaseNumber non-empty on found | Main.xaml | _(manual)_ | TC-003 |
| V-008 | out_Found = False for unknown attachment | Main.xaml | _(manual)_ | TC-004 |
| V-009 | BusinessRuleException on empty in_AttachmentName | Main.xaml | If_Validate | TC-005 |

---

## 8. Test Execution Instructions

### 8.1 Running Test_GetCaseDetail.xaml
1. Open the project in UiPath Studio.
2. Open `Test_GetCaseDetail.xaml`.
3. Set the input arguments (`TestName`, `CaseId`, `CaseNumber`, `ExpectedClassification`) via **Debug > Start with Arguments** or via Orchestrator Test parameters.
4. Run or debug the workflow.
5. Review the **Output panel** for log messages and **Test Results panel** for VerifyExpression pass/fail results.

### 8.2 Running Main.xaml End-to-End
1. Open `Main.xaml` in Studio.
2. Ensure `in_AttachmentName` is set to a valid test attachment filename (default is pre-configured in the XAML: `faxreceive.2026-04-17-16-24-27_5B8.pdf`).
3. Run the workflow.
4. Check the Output panel for log lines: `GetCaseDetailByAttachment START` → `GetCaseDetailByAttachment END`.
5. Validate output arguments in the Output panel.

### 8.3 Expected Log Output (Happy Path)
```
[Info]  GetCaseDetailByAttachment START — AttachmentName=faxreceive.2026-04-17-16-24-27_5B8.pdf
[Info]  Case fetched: CaseId=<id> CaseNumber=<number> HttpStatus=OK
[Info]  GetCaseDetailByAttachment END — Found=True CaseId=<id> CaseNumber=<number> MatchMethod=<method>
```

---

## 9. Known Limitations & Notes

| # | Note |
|---|---|
| 1 | **Classification logic review:** The `ExpectedClassification` argument in `Test_GetCaseDetail.xaml` is logged for SME review, not asserted. Classification correctness requires human sign-off until the business rules are finalized. |
| 2 | **Live API dependency:** All tests require a live connection to `bcbsrihub.my.salesforce.com`. Tests cannot run in an isolated/mocked environment without refactoring. |
| 3 | **OAuth credentials:** The `GetAccessToken` workflow requires valid Salesforce credentials/Connected App configuration. Expired credentials will cause all downstream tests to fail. |
| 4 | **ContinueOnError on HTTP:** The HTTP GET in `Main.xaml` uses `ContinueOnError="True"`. If the request fails silently, `caseResponse` will be null — check `out_CaseJson` for emptiness as a secondary signal. |
| 5 | **DataTable schema fixed:** The `io_DtCases` DataTable schema is hardcoded in `Test_GetCaseDetail.xaml`. Any new fields added to `GetCaseDetail.xaml` must also be added to the test initialization code. |
| 6 | **No mock/stub support:** There is currently no mechanism to mock Salesforce API responses. All testing is integration-level. |

---

## 10. Open Items / Recommended Additions

| Priority | Item |
|---|---|
| 🔴 High | Add `VerifyExpression` for `ExpectedClassification` once business rules are finalized with BCBSRI SMEs |
| 🔴 High | Create test data variations JSON file for `Test_GetCaseDetail.xaml` to run multiple Case IDs in one execution |
| 🟡 Medium | Add a test case for `FindCaseByAttachment.xaml` in isolation (TC-007) |
| 🟡 Medium | Add `VerifyExpression` to validate `out_Found = True` and `out_CaseId` non-empty in Main-level tests |
| 🟡 Medium | Add negative test for TC-NEG-001 through TC-NEG-006 as formal test cases in additional `.xaml` files |
| 🟢 Low | Parameterize `dataApiUrl` via an asset or config file rather than hardcoding in `Main.xaml` |
| 🟢 Low | Add HTTP status code assertion (`caseResponse.StatusCode = 200`) as a VerifyExpression |

---

*Document generated by Autopilot — Lydonia Technologies | Session: c441dd16-a543-4257-878c-5a7cd0b379f1*
