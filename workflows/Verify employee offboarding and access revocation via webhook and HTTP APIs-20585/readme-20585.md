Verify employee offboarding and access revocation via webhook and HTTP APIs

https://n8nworkflows.xyz/workflows/verify-employee-offboarding-and-access-revocation-via-webhook-and-http-apis-20585


# Verify employee offboarding and access revocation via webhook and HTTP APIs

### 1. Workflow Overview

This workflow automates the employee offboarding process by accepting an external HTTP webhook request, validating identity data, sequentially revoking and verifying access across Identity and Access Management (IAM), SaaS applications, and IT/HR systems, and finally returning a structured compliance report. 

The process is organized into three main logical blocks:
- **1.1 Intake & IAM Processing:** Receives the employee data payload, validates required fields, provisions request configurations, and executes the IAM identity revocation and verification cycle.
- **1.2 SaaS Access Revocation & Verification:** Pauses briefly before revoking access across a defined list of SaaS platforms (Slack, Google Workspace, GitHub, Notion, Jira) and subsequently verifies that access has been terminated.
- **1.3 IT/HR Finalization & Response:** Executes final IT and HR closure steps (disabling email, closing device tickets, updating employment status), compiles all verification markers into a compliance assessment, and responds to the original webhook caller with a descriptive status code and report.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Intake & IAM Processing

- **Overview:** This block establishes the entry point for offboarding requests, validates input integrity (such as email syntax and mandatory fields), generates baseline configuration mappings, and executes the IAM identity disablement and verification API calls.
- **Nodes Involved:** 
  - `Webhook - Offboarding Request`
  - `Code - Validate & Prepare`
  - `Wait - Pre IAM Revoke`
  - `HTTP Request - IAM Revoke`
  - `Code - Process IAM Revoke`
  - `Wait - Pre IAM Verify`
  - `HTTP Request - IAM Verify`
  - `Code - Process IAM Verify`

##### Node Details:

- **`Webhook - Offboarding Request`**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Trigger). Listens for incoming POST requests containing offboarding parameters.
  - **Configuration Choices:** Configured for `POST` method on path `employee-offboard` using the `responseNode` response mode.
  - **Key Expressions or Variables:** N/A (Trigger node).
  - **Input / Output Connections:** Input: None (Trigger) | Output: `Code - Validate & Prepare`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Unauthenticated or improperly formatted JSON payloads will pass raw data downstream, which is handled subsequently by the validation code node.

- **`Code - Validate & Prepare`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation and validation). Validates incoming properties (`employeeId`, `email`, `lastDay`), establishes configuration settings (including timeout limits and SaaS applications), and structures baseline URL mappings.
  - **Configuration Choices:** JavaScript execution environment checking for required fields, compiling email strings to lowercase, generating fallback user IDs, and optionally evaluating simulation status codes (`simulateStatus`).
  - **Key Expressions or Variables:** Reads properties dynamically from the preceding webhook payload via helper methods targeting `Webhook - Offboarding Request`.
  - **Input / Output Connections:** Input: `Webhook - Offboarding Request` | Output: `Wait - Pre IAM Revoke`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Missing mandatory payload fields populate an `errors` array, though execution proceeds to allow downstream handling.

- **`Wait - Pre IAM Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow control). Introduces a brief delay before hitting the IAM revocation endpoint.
  - **Configuration Choices:** Pauses execution for `1` second.
  - **Key Expressions or Variables:** None.
  - **Input / Output Connections:** Input: `Code - Validate & Prepare` | Output: `HTTP Request - IAM Revoke`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** None notable.

- **`HTTP Request - IAM Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Integration). Calls the external IAM endpoint to disable or delete the user identity.
  - **Configuration Choices:** Executes a `POST` request with a JSON body. Sets headers for `content-type` (`application/json`) and `X-Request-Id`. Configured with `neverError: true` and `fullResponse: true` to prevent workflow interruption on non-2xx status codes. `continueOnFail` is enabled.
  - **Key Expressions or Variables:** URL: `={{ $json.urls.iamRevoke }}`; Timeout: `={{ $json.config.timeoutMs }}`; Body payload serializes action, `userId`, `email`, and `employeeId`.
  - **Input / Output Connections:** Input: `Wait - Pre IAM Revoke` | Output: `Code - Process IAM Revoke`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Network timeouts, unreachable IAM servers, or authentication failures return standard HTTP error codes which are caught safely due to `continueOnFail`.

- **`Code - Process IAM Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Parses the HTTP status code returned by the IAM revoke call to evaluate success (`ok` boolean).
  - **Configuration Choices:** Evaluates status codes between 200 and 299 as successful.
  - **Key Expressions or Variables:** References upstream context data from `Code - Validate & Prepare` and current input status.
  - **Input / Output Connections:** Input: `HTTP Request - IAM Revoke` | Output: `Wait - Pre IAM Verify`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Unexpected response structures default to status `0` (failure).

- **`Wait - Pre IAM Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow control). Pauses execution to allow IAM replication before running verification checks.
  - **Configuration Choices:** Pauses execution for `2` seconds.
  - **Key Expressions or Variables:** None.
  - **Input / Output Connections:** Input: `Code - Process IAM Revoke` | Output: `HTTP Request - IAM Verify`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** None notable.

- **`HTTP Request - IAM Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Integration). Queries the IAM verification API to confirm the identity is fully disabled and lacks active sessions.
  - **Configuration Choices:** Executes a `POST` request with a JSON body (`verify_revoked`), custom `content-type`, and `X-Request-Id` headers. Employs `neverError: true` and `fullResponse: true`. `continueOnFail` is enabled.
  - **Key Expressions or Variables:** URL: `={{ $json.urls.iamVerify }}`; Body contains verification parameters.
  - **Input / Output Connections:** Input: `Wait - Pre IAM Verify` | Output: `Code - Process IAM Verify`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Downstream API failures will return error codes handled gracefully by subsequent processing nodes.

- **`Code - Process IAM Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Aggregates IAM verification status into the cumulative results payload.
  - **Configuration Choices:** Assumes successful verification if HTTP response is within the 2xx range.
  - **Key Expressions or Variables:** References context from `Code - Process IAM Revoke`.
  - **Input / Output Connections:** Input: `HTTP Request - IAM Verify` | Output: `Wait - Pre SaaS Revoke`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** None.

---

#### Block 1.2: SaaS Access Revocation & Verification

- **Overview:** This block pauses execution, calls the SaaS revocation endpoint to remove the user across configured applications (Slack, Google Workspace, GitHub, Notion, Jira), waits for propagation, and checks verification status.
- **Nodes Involved:** 
  - `Wait - Pre SaaS Revoke`
  - `HTTP Request - SaaS Revoke`
  - `Code - Process SaaS Revoke`
  - `Wait - Pre SaaS Verify`
  - `HTTP Request - SaaS Verify`
  - `Code - Process SaaS Verify`

##### Node Details:

- **`Wait - Pre SaaS Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow control). Short pause prior to triggering SaaS app deprovisioning.
  - **Configuration Choices:** Pauses for `1` second.
  - **Key Expressions or Variables:** None.
  - **Input / Output Connections:** Input: `Code - Process IAM Verify` | Output: `HTTP Request - SaaS Revoke`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** None.

- **`HTTP Request - SaaS Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Integration). Calls the SaaS revocation API passing the list of target applications.
  - **Configuration Choices:** `POST` request with JSON payload including the apps array (`slack`, `google_workspace`, `github`, `notion`, `jira`). Uses `neverError: true`, `fullResponse: true`, and enables `continueOnFail`.
  - **Key Expressions or Variables:** URL: `={{ $json.urls.saasRevoke }}`; Body serializes configuration apps array.
  - **Input / Output Connections:** Input: `Wait - Pre SaaS Revoke` | Output: `Code - Process SaaS Revoke`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Partial SaaS service outages may return non-2xx codes, captured safely without breaking the workflow.

- **`Code - Process SaaS Revoke`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Appends SaaS revocation results to the consolidated state object.
  - **Configuration Choices:** Evaluates HTTP success status and merges application arrays.
  - **Key Expressions or Variables:** References context from `Code - Process IAM Verify`.
  - **Input / Output Connections:** Input: `HTTP Request - SaaS Revoke` | Output: `Wait - Pre SaaS Verify`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** None.

- **`Wait - Pre SaaS Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow control). Pause to allow SaaS directory updates to propagate before verification.
  - **Configuration Choices:** Pauses for `2` seconds.
  - **Key Expressions or Variables:** None.
  - **Input / Output Connections:** Input: `Code - Process SaaS Revoke` | Output: `HTTP Request - SaaS Verify`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** None.

- **`HTTP Request - SaaS Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Integration). Verifies that application-level access has been removed across the specified SaaS tools.
  - **Configuration Choices:** `POST` request with `verify_revoked` action payload. Uses `neverError: true`, `fullResponse: true`, and enables `continueOnFail`.
  - **Key Expressions or Variables:** URL: `={{ $json.urls.saasVerify }}`.
  - **Input / Output Connections:** Input: `Wait - Pre SaaS Verify` | Output: `Code - Process SaaS Verify`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Network interruptions or endpoint degradation handled via `continueOnFail`.

- **`Code - Process SaaS Verify`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Confirms SaaS verification status and preserves cumulative metrics.
  - **Configuration Choices:** Parses verification response status.
  - **Key Expressions or Variables:** References context from `Code - Process SaaS Revoke`.
  - **Input / Output Connections:** Input: `HTTP Request - SaaS Verify` | Output: `Wait - Pre IT & HR`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** None.

---

#### Block 1.3: IT/HR Finalization & Response

- **Overview:** This final block executes the IT and HR system finalization (disabling email, closing asset tickets, updating status), aggregates all check results into a compliance report, determines appropriate HTTP response codes, and returns the result to the calling webhook.
- **Nodes Involved:** 
  - `Wait - Pre IT & HR`
  - `HTTP Request - IT & HR Finalize`
  - `Code - Build Final Report`
  - `Code - Finalize Response`
  - `Respond to Webhook`

##### Node Details:

- **`Wait - Pre IT & HR`**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow control). Brief pause prior to triggering IT and HR closure steps.
  - **Configuration Choices:** Pauses for `1` second.
  - **Key Expressions or Variables:** None.
  - **Input / Output Connections:** Input: `Code - Process SaaS Verify` | Output: `HTTP Request - IT & HR Finalize`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** None.

- **`HTTP Request - IT & HR Finalize`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Integration). Finalizes employee offboarding across IT and HR systems.
  - **Configuration Choices:** `POST` request sending employee metadata (`employeeId`, `email`, `userId`, `lastDay`, `reason`). Uses `neverError: true`, `fullResponse: true`, and enables `continueOnFail`.
  - **Key Expressions or Variables:** URL: `={{ $json.urls.itHr }}`.
  - **Input / Output Connections:** Input: `Wait - Pre IT & HR` | Output: `Code - Build Final Report`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Endpoint timeout or failure handled safely.

- **`Code - Build Final Report`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Aggregates results from IAM, SaaS, and IT/HR steps to calculate compliance scores and overall status (`fully_revoked`, `partial`, or `failed`).
  - **Configuration Choices:** Evaluates boolean success criteria across all checks to determine completion state and generates a timestamp.
  - **Key Expressions or Variables:** References context from `Code - Process SaaS Verify`.
  - **Input / Output Connections:** Input: `HTTP Request - IT & HR Finalize` | Output: `Code - Finalize Response`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** None.

- **`Code - Finalize Response`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data transformation). Formulates the HTTP response mapping based on compliance results.
  - **Configuration Choices:** Sets HTTP status codes: `200` for fully revoked, `207` for partial revocation, and `502` for complete failure.
  - **Key Expressions or Variables:** References context from `Code - Build Final Report`.
  - **Input / Output Connections:** Input: `Code - Build Final Report` | Output: `Respond to Webhook`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** None.

- **`Respond to Webhook`**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response). Returns the complete offboarding and verification JSON report back to the original webhook caller.
  - **Configuration Choices:** Responds with JSON data using dynamic response code and `X-Request-Id` response header.
  - **Key Expressions or Variables:** Response Code: `={{ $json.httpStatus }}`; Header `X-Request-Id`: `={{ $json.requestId }}`; Body: `={{ $json.body }}`.
  - **Input / Output Connections:** Input: `Code - Finalize Response` | Output: None (Terminal node).
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** Fails if the HTTP connection to the webhook client has dropped prematurely.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documents high-level architecture and lists workflow steps. | None | None | ## Automated Employee Offboarding System<br><br>Receives an offboarding request, revokes access across IAM, SaaS, IT and HR systems, then verifies that access has actually been removed before returning a final compliance report.<br><br>### All nodes (1-line each)<br>1. Webhook - Offboarding Request – Receives the employee offboarding payload via POST.<br>2. Code - Validate & Prepare – Validates the request and builds the revocation + verification payloads.<br>3. Wait - Pre IAM Revoke – Short pause before revoking identity access.<br>4. HTTP Request - IAM Revoke – Disables or deletes the user identity and removes group memberships.<br>5. Code - Process IAM Revoke – Records IAM revocation result and prepares the verification call.<br>6. Wait - Pre IAM Verify – Pause before verifying IAM revocation.<br>7. HTTP Request - IAM Verify – Checks that the user identity is disabled and has no active sessions.<br>8. Code - Process IAM Verify – Confirms IAM verification status.<br>9. Wait - Pre SaaS Revoke – Pause before SaaS access revocation.<br>10. HTTP Request - SaaS Revoke – Removes the user from Slack, Google Workspace, GitHub and other apps.<br>11. Code - Process SaaS Revoke – Records SaaS revocation results.<br>12. Wait - Pre SaaS Verify – Pause before SaaS verification.<br>13. HTTP Request - SaaS Verify – Confirms the user no longer has active access in SaaS systems.<br>14. Code - Process SaaS Verify – Confirms SaaS verification status.<br>15. Wait - Pre IT & HR – Pause before IT and HR finalization.<br>16. HTTP Request - IT & HR Finalize – Disables email, closes device tickets and updates HR status.<br>17. Code - Build Final Report – Aggregates all revocation and verification results into a compliance report.<br>18. Code - Finalize Response – Assembles the final offboarding status for the caller.<br>19. Respond to Webhook – Returns the complete offboarding + verification report. |
| `Sticky Note - Intake & IAM` | `n8n-nodes-base.stickyNote` | Groups Intake and IAM Revoke/Verify steps. | None | None | ## 1. Intake & IAM Revoke + Verify<br><br>Webhook - Offboarding Request – Receives the employee offboarding payload via POST.<br>Code - Validate & Prepare – Validates the request and builds the revocation + verification payloads.<br>Wait - Pre IAM Revoke – Short pause before revoking identity access.<br>HTTP Request - IAM Revoke – Disables or deletes the user identity and removes group memberships.<br>Code - Process IAM Revoke – Records IAM revocation result and prepares the verification call.<br>Wait - Pre IAM Verify – Pause before verifying IAM revocation.<br>HTTP Request - IAM Verify – Checks that the user identity is disabled and has no active sessions.<br>Code - Process IAM Verify – Confirms IAM verification status. |
| `Sticky Note - SaaS Revoke + Verify` | `n8n-nodes-base.stickyNote` | Groups SaaS Revoke and Verify steps. | None | None | ## 2. SaaS Revoke + Verify<br><br>Wait - Pre SaaS Revoke – Pause before SaaS access revocation.<br>HTTP Request - SaaS Revoke – Removes the user from Slack, Google Workspace, GitHub and other apps.<br>Code - Process SaaS Revoke – Records SaaS revocation results.<br>Wait - Pre SaaS Verify – Pause before SaaS verification.<br>HTTP Request - SaaS Verify – Confirms the user no longer has active access in SaaS systems.<br>Code - Process SaaS Verify – Confirms SaaS verification status. |
| `Sticky Note - IT/HR & Finish` | `n8n-nodes-base.stickyNote` | Groups IT/HR finalization and response steps. | None | None | ## 3. IT / HR & Finish<br><br>Wait - Pre IT & HR – Pause before IT and HR finalization.<br>HTTP Request - IT & HR Finalize – Disables email, closes device tickets and updates HR status.<br>Code - Build Final Report – Aggregates all revocation and verification results into a compliance report.<br>Code - Finalize Response – Assembles the final offboarding status for the caller.<br>Respond to Webhook – Returns the complete offboarding + verification report. |
| `Webhook - Offboarding Request` | `n8n-nodes-base.webhook` | Receives the employee offboarding payload via POST. | None | `Code - Validate & Prepare` | Receives the employee offboarding payload via POST. |
| `Code - Validate & Prepare` | `n8n-nodes-base.code` | Validates the request and builds the revocation + verification payloads. | `Webhook - Offboarding Request` | `Wait - Pre IAM Revoke` | Validates the request and builds the revocation + verification payloads. |
| `Wait - Pre IAM Revoke` | `n8n-nodes-base.wait` | Short pause before revoking identity access. | `Code - Validate & Prepare` | `HTTP Request - IAM Revoke` | Short pause before revoking identity access. |
| `HTTP Request - IAM Revoke` | `n8n-nodes-base.httpRequest` | Disables or deletes the user identity and removes group memberships. | `Wait - Pre IAM Revoke` | `Code - Process IAM Revoke` | Disables or deletes the user identity and removes group memberships. |
| `Code - Process IAM Revoke` | `n8n-nodes-base.code` | Records IAM revocation result and prepares the verification call. | `HTTP Request - IAM Revoke` | `Wait - Pre IAM Verify` | Records IAM revocation result and prepares the verification call. |
| `Wait - Pre IAM Verify` | `n8n-nodes-base.wait` | Pause before verifying IAM revocation. | `Code - Process IAM Revoke` | `HTTP Request - IAM Verify` | Pause before verifying IAM revocation. |
| `HTTP Request - IAM Verify` | `n8n-nodes-base.httpRequest` | Checks that the user identity is disabled and has no active sessions. | `Wait - Pre IAM Verify` | `Code - Process IAM Verify` | Checks that the user identity is disabled and has no active sessions. |
| `Code - Process IAM Verify` | `n8n-nodes-base.code` | Confirms IAM verification status. | `HTTP Request - IAM Verify` | `Wait - Pre SaaS Revoke` | Confirms IAM verification status. |
| `Wait - Pre SaaS Revoke` | `n8n-nodes-base.wait` | Pause before SaaS access revocation. | `Code - Process IAM Verify` | `HTTP Request - SaaS Revoke` | Pause before SaaS access revocation. |
| `HTTP Request - SaaS Revoke` | `n8n-nodes-base.httpRequest` | Removes the user from Slack, Google Workspace, GitHub and other apps. | `Wait - Pre SaaS Revoke` | `Code - Process SaaS Revoke` | Removes the user from Slack, Google Workspace, GitHub and other apps. |
| `Code - Process SaaS Revoke` | `n8n-nodes-base.code` | Records SaaS revocation results. | `HTTP Request - SaaS Revoke` | `Wait - Pre SaaS Verify` | Records SaaS revocation results. |
| `Wait - Pre SaaS Verify` | `n8n-nodes-base.wait` | Pause before SaaS verification. | `Code - Process SaaS Revoke` | `HTTP Request - SaaS Verify` | Pause before SaaS verification. |
| `HTTP Request - SaaS Verify` | `n8n-nodes-base.httpRequest` | Confirms the user no longer has active access in SaaS systems. | `Wait - Pre SaaS Verify` | `Code - Process SaaS Verify` | Confirms the user no longer has active access in SaaS systems. |
| `Code - Process SaaS Verify` | `n8n-nodes-base.code` | Confirms SaaS verification status. | `HTTP Request - SaaS Verify` | `Wait - Pre IT & HR` | Confirms SaaS verification status. |
| `Wait - Pre IT & HR` | `n8n-nodes-base.wait` | Pause before IT and HR finalization. | `Code - Process SaaS Verify` | `HTTP Request - IT & HR Finalize` | Pause before IT and HR finalization. |
| `HTTP Request - IT & HR Finalize` | `n8n-nodes-base.httpRequest` | Disables email, closes device tickets and updates HR status. | `Wait - Pre IT & HR` | `Code - Build Final Report` | Disables email, closes device tickets and updates HR status. |
| `Code - Build Final Report` | `n8n-nodes-base.code` | Aggregates all revocation and verification results into a compliance report. | `HTTP Request - IT & HR Finalize` | `Code - Finalize Response` | Aggregates all revocation and verification results into a compliance report. |
| `Code - Finalize Response` | `n8n-nodes-base.code` | Assembles the final offboarding status for the caller. | `Code - Build Final Report` | `Respond to Webhook` | Assembles the final offboarding status for the caller. |
| `Respond to Webhook` | `n8n-nodes-base.respondToWebhook` | Returns the complete offboarding + verification report. | `Code - Finalize Response` | None | Returns the complete offboarding + verification report. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to build the workflow manually in n8n:

1. **Create Webhook Trigger:**
   - Add a **Webhook** node named `Webhook - Offboarding Request`.
   - Set HTTP Method to `POST`, Path to `employee-offboard`, and Response Mode to `Response Node`.

2. **Add Validation and Preparation Logic:**
   - Add a **Code** node named `Code - Validate & Prepare`.
   - Paste the validation script which extracts parameters (`employeeId`, `email`, `lastDay`, etc.), checks for valid email formats, and defines default target URLs (`https://httpbin.org/post`).
   - Connect `Webhook - Offboarding Request` to `Code - Validate & Prepare`.

3. **Configure IAM Revocation Sequence:**
   - Add a **Wait** node named `Wait - Pre IAM Revoke` (Amount: `1` second). Connect `Code - Validate & Prepare` to it.
   - Add an **HTTP Request** node named `HTTP Request - IAM Revoke` (Method: `POST`, URL: `={{ $json.urls.iamRevoke }}`). Enable `Never Error` and `Full Response` in options, and enable `Continue On Fail`. Configure JSON body serialization and headers (`content-type`, `X-Request-Id`). Connect `Wait - Pre IAM Revoke` to it.
   - Add a **Code** node named `Code - Process IAM Revoke` to evaluate status codes. Connect `HTTP Request - IAM Revoke` to it.

4. **Configure IAM Verification Sequence:**
   - Add a **Wait** node named `Wait - Pre IAM Verify` (Amount: `2` seconds). Connect `Code - Process IAM Revoke` to it.
   - Add an **HTTP Request** node named `HTTP Request - IAM Verify` (Method: `POST`, URL: `={{ $json.urls.iamVerify }}`). Enable `Never Error`, `Full Response`, and `Continue On Fail`. Set JSON body action to `verify_revoked`. Connect `Wait - Pre IAM Verify` to it.
   - Add a **Code** node named `Code - Process IAM Verify`. Connect `HTTP Request - IAM Verify` to it.

5. **Configure SaaS Revocation Sequence:**
   - Add a **Wait** node named `Wait - Pre SaaS Revoke` (Amount: `1` second). Connect `Code - Process IAM Verify` to it.
   - Add an **HTTP Request** node named `HTTP Request - SaaS Revoke` (Method: `POST`, URL: `={{ $json.urls.saasRevoke }}`). Enable `Never Error`, `Full Response`, and `Continue On Fail`. Pass the `apps` array in the JSON body. Connect `Wait - Pre SaaS Revoke` to it.
   - Add a **Code** node named `Code - Process SaaS Revoke`. Connect `HTTP Request - SaaS Revoke` to it.

6. **Configure SaaS Verification Sequence:**
   - Add a **Wait** node named `Wait - Pre SaaS Verify` (Amount: `2` seconds). Connect `Code - Process SaaS Revoke` to it.
   - Add an **HTTP Request** node named `HTTP Request - SaaS Verify` (Method: `POST`, URL: `={{ $json.urls.saasVerify }}`). Enable `Never Error`, `Full Response`, and `Continue On Fail`. Set action to `verify_revoked`. Connect `Wait - Pre SaaS Verify` to it.
   - Add a **Code** node named `Code - Process SaaS Verify`. Connect `HTTP Request - SaaS Verify` to it.

7. **Configure IT/HR Finalization & Response:**
   - Add a **Wait** node named `Wait - Pre IT & HR` (Amount: `1` second). Connect `Code - Process SaaS Verify` to it.
   - Add an **HTTP Request** node named `HTTP Request - IT & HR Finalize` (Method: `POST`, URL: `={{ $json.urls.itHr }}`). Enable `Never Error`, `Full Response`, and `Continue On Fail`. Connect `Wait - Pre IT & HR` to it.
   - Add a **Code** node named `Code - Build Final Report` to aggregate check results and compute compliance status (`fully_revoked`, `partial`, or `failed`). Connect `HTTP Request - IT & HR Finalize` to it.
   - Add a **Code** node named `Code - Finalize Response` to map compliance status to HTTP response codes (`200`, `207`, `502`). Connect `Code - Build Final Report` to it.
   - Add a **Respond to Webhook** node named `Respond to Webhook` (Respond with: `JSON`, Response Body: `={{ $json.body }}`). Configure response code expression `={{ $json.httpStatus }}` and response header `X-Request-Id`. Connect `Code - Finalize Response` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automated Employee Offboarding System | Manages identity revocation, SaaS deprovisioning, IT/HR closure, and verification reporting via webhook integrations. |