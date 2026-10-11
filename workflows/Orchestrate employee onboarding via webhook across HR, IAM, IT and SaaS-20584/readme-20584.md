Orchestrate employee onboarding via webhook across HR, IAM, IT and SaaS

https://n8nworkflows.xyz/workflows/orchestrate-employee-onboarding-via-webhook-across-hr--iam--it-and-saas-20584


# Orchestrate employee onboarding via webhook across HR, IAM, IT and SaaS

### 1. Workflow Overview

This workflow automates the employee onboarding process across Human Resources (HR), Identity and Access Management (IAM), Information Technology (IT), and Software-as-a-Service (SaaS) systems. It functions as an orchestration engine that receives a new-hire POST request, validates and normalizes the payload, and executes provisioning requests sequentially across multiple target platforms. It evaluates the outcomes of each step, aggregates the results into a final status report, and responds to the original caller with a dynamic HTTP status and a JSON report.

The workflow logic is categorized into three functional blocks:
- **1.1 Intake & HR:** Receives the webhook payload, validates and normalizes employee records, handles simulation parameters, and executes the HR system provisioning request.
- **1.2 IAM & IT:** Provisions user identities and access groups via the IAM system, followed by core IT infrastructure provisioning (email, device tickets).
- **1.3 SaaS & Finish:** Provisions access to configured SaaS applications, aggregates system statuses, compiles the final report, and returns the response to the webhook caller.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Intake & HR

##### Overview
This block receives the new-hire request, performs validation on mandatory employee fields, normalizes the data structure, handles testing simulation flags, and invokes the HR system API.

##### Nodes Involved
- Webhook - New Hire Request
- Code - Validate & Normalize
- Wait - Pre HR
- HTTP Request - HR System
- Code - Process HR Result

---

##### Node Details: Webhook - New Hire Request
- **Type and technical role:** `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the primary entry point for incoming HTTP POST requests containing employee onboarding payloads.
- **Configuration choices:** Configured to listen on path `employee-onboard` using method `POST`. Response mode is set to use a dedicated response node (`responseNode`).
- **Key expressions or variables:** None.
- **Input and output connections:** Inputs: External HTTP POST. Outputs: `Code - Validate & Normalize`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Malformed JSON bodies or missing headers may cause downstream validation failures if fields are entirely undefined.

##### Node Details: Code - Validate & Normalize
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Validates required fields (`firstName`, `lastName`, email format, `startDate`), normalizes naming conventions, assigns default values for missing attributes, and builds global configuration objects.
- **Configuration choices:** Runs custom JavaScript to extract attributes from body parameters (`firstName`, `last_name`, etc.), handles email sanitization, and checks for an optional `simulateStatus` variable for testing HTTP status responses against `httpbin.org/status/{code}`.
- **Key expressions or variables:** Reads from `Webhook - New Hire Request`. Generates unique `requestId` and `employeeId` values if not provided.
- **Input and output connections:** Inputs: `Webhook - New Hire Request`. Outputs: `Wait - Pre HR`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Validation failure arrays are populated if mandatory fields are missing; however, the workflow passes `valid: false` downstream without halting execution immediately.

##### Node Details: Wait - Pre HR
- **Type and technical role:** `n8n-nodes-base.wait` (Timing control). Inserts a controlled pause before hitting the HR system API.
- **Configuration choices:** Set to wait for `1` second.
- **Key expressions or variables:** None.
- **Input and output connections:** Inputs: `Code - Validate & Normalize`. Outputs: `HTTP Request - HR System`.
- **Version-specific requirements:** Version 1.1.
- **Edge cases or potential failure types:** None.

##### Node Details: HTTP Request - HR System
- **Type and technical role:** `n8n-nodes-base.httpRequest` (Outbound API call). Sends employee data to the HRIS endpoint.
- **Configuration choices:** Method: `POST`. URL mapped dynamically from configuration. Sends body as JSON with content-type `application/json` and an `X-Request-Id` header. Configured with `neverError: true` and `fullResponse: true` to prevent workflow termination on non-2xx status codes.
- **Key expressions or variables:** `{{ $json.urls.hr }}`, `{{ $json.config.timeoutMs }}`, `{{ JSON.stringify(...) }}`, `{{ $json.requestId }}`.
- **Input and output connections:** Inputs: `Wait - Pre HR`. Outputs: `Code - Process HR Result`.
- **Version-specific requirements:** Version 4.2.
- **Edge cases or potential failure types:** Network timeouts, unreachable endpoints, or HTTP 4xx/5xx errors (mitigated by `continueOnFail: true` and `neverError: true`).

##### Node Details: Code - Process HR Result
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Evaluates the HR system's HTTP response status and extracts or generates the employee identifier for downstream IAM provisioning.
- **Configuration choices:** Parses response bodies safely, verifies status code ranges (`200–299`), and constructs the `iamPayload`.
- **Key expressions or variables:** References `Code - Validate & Normalize` and current node input.
- **Input and output connections:** Inputs: `HTTP Request - HR System`. Outputs: `Wait - Pre IAM`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Malformed JSON responses from the HR API fallback to empty objects.

---

#### Block 1.2: IAM & IT

##### Overview
This block manages identity provisioning via the IAM system and subsequently provisions core IT infrastructure and resources (such as email accounts and hardware tickets).

##### Nodes Involved
- Wait - Pre IAM
- HTTP Request - IAM / Identity
- Code - Process IAM Result
- Wait - Pre IT
- HTTP Request - IT Provisioning
- Code - Process IT Result

---

##### Node Details: Wait - Pre IAM
- **Type and technical role:** `n8n-nodes-base.wait` (Timing control). Inserts a brief delay prior to calling the IAM API.
- **Configuration choices:** Set to wait for `1` second.
- **Key expressions or variables:** None.
- **Input and output connections:** Inputs: `Code - Process HR Result`. Outputs: `HTTP Request - IAM / Identity`.
- **Version-specific requirements:** Version 1.1.
- **Edge cases or potential failure types:** None.

##### Node Details: HTTP Request - IAM / Identity
- **Type and technical role:** `n8n-nodes-base.httpRequest` (Outbound API call). Sends identity creation payloads to the IAM provider.
- **Configuration choices:** Method: `POST`. URL set to `{{ $json.urls.iam }}`. Sends JSON payload with `X-Request-Id` header. Configured with `neverError: true` and `fullResponse: true`.
- **Key expressions or variables:** `{{ $json.urls.iam }}`, `{{ JSON.stringify($json.iamPayload) }}`, `{{ $json.requestId }}`.
- **Input and output connections:** Inputs: `Wait - Pre IAM`. Outputs: `Code - Process IAM Result`.
- **Version-specific requirements:** Version 4.2.
- **Edge cases or potential failure types:** API downtime, schema mismatches, or authentication failures.

##### Node Details: Code - Process IAM Result
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Extracts the newly created user ID from the IAM response and constructs payloads for both IT provisioning and SaaS application provisioning.
- **Configuration choices:** Evaluates HTTP status success, resolves user ID aliases (`userId`, `id`, `sub`), and carries forward global context.
- **Key expressions or variables:** References `Code - Process HR Result` and current input.
- **Input and output connections:** Inputs: `HTTP Request - IAM / Identity`. Outputs: `Wait - Pre IT`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Missing identifiers in response bodies trigger fallback ID generation (`user-{employeeId}`).

##### Node Details: Wait - Pre IT
- **Type and technical role:** `n8n-nodes-base.wait` (Timing control). Inserts a pause before triggering IT provisioning.
- **Configuration choices:** Set to wait for `1` second.
- **Key expressions or variables:** None.
- **Input and output connections:** Inputs: `Code - Process IAM Result`. Outputs: `HTTP Request - IT Provisioning`.
- **Version-specific requirements:** Version 1.1.
- **Edge cases or potential failure types:** None.

##### Node Details: HTTP Request - IT Provisioning
- **Type and technical role:** `n8n-nodes-base.httpRequest` (Outbound API call). Submits IT resource requests (email accounts, hardware tickets).
- **Configuration choices:** Method: `POST`. URL set to `{{ $json.urls.it }}`. Sends JSON body with request headers. Configured with `neverError: true` and `fullResponse: true`.
- **Key expressions or variables:** `{{ $json.urls.it }}`, `{{ JSON.stringify($json.itPayload) }}`, `{{ $json.requestId }}`.
- **Input and output connections:** Inputs: `Wait - Pre IT`. Outputs: `Code - Process IT Result`.
- **Version-specific requirements:** Version 4.2.
- **Edge cases or potential failure types:** Service desk API failures or timeouts.

##### Node Details: Code - Process IT Result
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Verifies IT provisioning success, captures ticket identifiers, and forwards context for SaaS provisioning.
- **Configuration choices:** Parses HTTP responses, determines IT success states, and aggregates results.
- **Key expressions or variables:** References `Code - Process IAM Result`.
- **Input and output connections:** Inputs: `HTTP Request - IT Provisioning`. Outputs: `Wait - Pre SaaS`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Unparseable response bodies default to null ticket IDs.

---

#### Block 1.3: SaaS & Finish

##### Overview
This block provisions access to required SaaS applications, aggregates success metrics across all four integrated systems (HR, IAM, IT, SaaS), formats the final status report, and responds to the webhook caller.

##### Nodes Involved
- Wait - Pre SaaS
- HTTP Request - SaaS Apps
- Code - Build Final Status
- Code - Finalize Response
- Respond to Webhook

---

##### Node Details: Wait - Pre SaaS
- **Type and technical role:** `n8n-nodes-base.wait` (Timing control). Inserts a pause before calling the SaaS provisioning API.
- **Configuration choices:** Set to wait for `1` second.
- **Key expressions or variables:** None.
- **Input and output connections:** Inputs: `Code - Process IT Result`. Outputs: `HTTP Request - SaaS Apps`.
- **Version-specific requirements:** Version 1.1.
- **Edge cases or potential failure types:** None.

##### Node Details: HTTP Request - SaaS Apps
- **Type and technical role:** `n8n-nodes-base.httpRequest` (Outbound API call). Triggers SaaS application access grants (Slack, Google Workspace, Notion, GitHub).
- **Configuration choices:** Method: `POST`. URL set to `{{ $json.urls.saas }}`. Sends JSON body containing app arrays and user identifiers. Configured with `neverError: true` and `fullResponse: true`.
- **Key expressions or variables:** `{{ $json.urls.saas }}`, `{{ JSON.stringify($json.saasPayload) }}`, `{{ $json.requestId }}`.
- **Input and output connections:** Inputs: `Wait - Pre SaaS`. Outputs: `Code - Build Final Status`.
- **Version-specific requirements:** Version 4.2.
- **Edge cases or potential failure types:** Rate limiting or failures in third-party SaaS management APIs.

##### Node Details: Code - Build Final Status
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Consolidates success and failure flags from HR, IAM, IT, and SaaS systems to determine the overall onboarding status (`completed`, `partial`, or `failed`).
- **Configuration choices:** Evaluates boolean flags for all four domains, counts successful integrations, and generates a timestamp.
- **Key expressions or variables:** References `Code - Process IT Result`.
- **Input and output connections:** Inputs: `HTTP Request - SaaS Apps`. Outputs: `Code - Finalize Response`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** None.

##### Node Details: Code - Finalize Response
- **Type and technical role:** `n8n-nodes-base.code` (JavaScript execution). Determines the appropriate HTTP response code based on provisioning results (`201` for completed, `207` for partial, `502` for failure) and structures the final response body.
- **Configuration choices:** Maps summary statuses to HTTP status codes and prepares the payload wrapper.
- **Key expressions or variables:** References `Code - Build Final Status`.
- **Input and output connections:** Inputs: `Code - Build Final Status`. Outputs: `Respond to Webhook`.
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** None.

##### Node Details: Respond to Webhook
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` (Webhook Response). Returns the final HTTP response and headers to the original webhook caller.
- **Configuration choices:** Responds with `json`. Response code configured dynamically (`{{ $json.httpStatus }}`). Includes an `X-Request-Id` response header.
- **Key expressions or variables:** `{{ $json.httpStatus }}`, `{{ $json.requestId }}`, `{{ $json.body }}`.
- **Input and output connections:** Inputs: `Code - Finalize Response`. Outputs: None.
- **Version-specific requirements:** Version 1.1.
- **Edge cases or potential failure types:** Connection drops with the initial caller before the response is fully transmitted.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - New Hire Request | n8n-nodes-base.webhook | Receives employee onboarding payload via POST | (External) | Code - Validate & Normalize | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 1. Intake & HR<br><br>Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>Wait - Pre HR – Short pause before calling the HR system.<br>HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>Code - Process HR Result – Checks HR response and prepares the IAM payload. |
| Code - Validate & Normalize | n8n-nodes-base.code | Validates required fields and normalizes the employee record | Webhook - New Hire Request | Wait - Pre HR | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 1. Intake & HR<br><br>Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>Wait - Pre HR – Short pause before calling the HR system.<br>HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>Code - Process HR Result – Checks HR response and prepares the IAM payload. |
| Wait - Pre HR | n8n-nodes-base.wait | Short pause before calling the HR system | Code - Validate & Normalize | HTTP Request - HR System | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 1. Intake & HR<br><br>Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>Wait - Pre HR – Short pause before calling the HR system.<br>HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>Code - Process HR Result – Checks HR response and prepares the IAM payload. |
| HTTP Request - HR System | n8n-nodes-base.httpRequest | Creates or updates the employee record in the HRIS | Wait - Pre HR | Code - Process HR Result | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 1. Intake & HR<br><br>Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>Wait - Pre HR – Short pause before calling the HR system.<br>HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>Code - Process HR Result – Checks HR response and prepares the IAM payload. |
| Code - Process HR Result | n8n-nodes-base.code | Checks HR response and prepares the IAM payload | HTTP Request - HR System | Wait - Pre IAM | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 1. Intake & HR<br><br>Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>Wait - Pre HR – Short pause before calling the HR system.<br>HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>Code - Process HR Result – Checks HR response and prepares the IAM payload. |
| Wait - Pre IAM | n8n-nodes-base.wait | Pause before identity provisioning | Code - Process HR Result | HTTP Request - IAM / Identity | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| HTTP Request - IAM / Identity | n8n-nodes-base.httpRequest | Creates the user identity, groups and initial access | Wait - Pre IAM | Code - Process IAM Result | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| Code - Process IAM Result | n8n-nodes-base.code | Extracts the new user ID and prepares IT + SaaS payloads | HTTP Request - IAM / Identity | Wait - Pre IT | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| Wait - Pre IT | n8n-nodes-base.wait | Pause before IT provisioning | Code - Process IAM Result | HTTP Request - IT Provisioning | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| HTTP Request - IT Provisioning | n8n-nodes-base.httpRequest | Creates email, device ticket and core IT accounts | Wait - Pre IT | Code - Process IT Result | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| Code - Process IT Result | n8n-nodes-base.code | Confirms IT resources and prepares SaaS calls | HTTP Request - IT Provisioning | Wait - Pre SaaS | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 2. IAM & IT<br><br>Wait - Pre IAM – Pause before identity provisioning.<br>HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>Wait - Pre IT – Pause before IT provisioning.<br>HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>Code - Process IT Result – Confirms IT resources and prepares SaaS calls. |
| Wait - Pre SaaS | n8n-nodes-base.wait | Pause before SaaS application provisioning | Code - Process IT Result | HTTP Request - SaaS Apps | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 3. SaaS & Finish<br><br>Wait - Pre SaaS – Pause before SaaS application provisioning.<br>HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>Code - Build Final Status – Aggregates results from all systems into a single status object.<br>Code - Finalize Response – Assembles the final onboarding report for the caller.<br>Respond to Webhook – Returns the complete onboarding status to the caller. |
| HTTP Request - SaaS Apps | n8n-nodes-base.httpRequest | Provisions access to Slack, Google Workspace, Notion, etc | Wait - Pre SaaS | Code - Build Final Status | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 3. SaaS & Finish<br><br>Wait - Pre SaaS – Pause before SaaS application provisioning.<br>HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>Code - Build Final Status – Aggregates results from all systems into a single status object.<br>Code - Finalize Response – Assembles the final onboarding report for the caller.<br>Respond to Webhook – Returns the complete onboarding status to the caller. |
| Code - Build Final Status | n8n-nodes-base.code | Aggregates results from all systems into a single status object | HTTP Request - SaaS Apps | Code - Finalize Response | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 3. SaaS & Finish<br><br>Wait - Pre SaaS – Pause before SaaS application provisioning.<br>HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>Code - Build Final Status – Aggregates results from all systems into a single status object.<br>Code - Finalize Response – Assembles the final onboarding report for the caller.<br>Respond to Webhook – Returns the complete onboarding status to the caller. |
| Code - Finalize Response | n8n-nodes-base.code | Assembles the final onboarding report for the caller | Code - Build Final Status | Respond to Webhook | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 3. SaaS & Finish<br><br>Wait - Pre SaaS – Pause before SaaS application provisioning.<br>HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>Code - Build Final Status – Aggregates results from all systems into a single status object.<br>Code - Finalize Response – Assembles the final onboarding report for the caller.<br>Respond to Webhook – Returns the complete onboarding status to the caller. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns the complete onboarding status to the caller | Code - Finalize Response | (External) | ## Employee Onboarding Orchestration Engine<br><br>Receives a new-hire payload and orchestrates provisioning across HR, IAM, IT and SaaS systems in a controlled sequence with status tracking, retries and a final summary.<br><br>### All nodes (1-line each)<br>1. Webhook - New Hire Request – Receives the employee onboarding payload via POST.<br>2. Code - Validate & Nonlinear – Validates required fields and normalizes the employee record.<br>3. Wait - Pre HR – Short pause before calling the HR system.<br>4. HTTP Request - HR System – Creates or updates the employee record in the HRIS.<br>5. Code - Process HR Result – Checks HR response and prepares the IAM payload.<br>6. Wait - Pre IAM – Pause before identity provisioning.<br>7. HTTP Request - IAM / Identity – Creates the user identity, groups and initial access.<br>8. Code - Process IAM Result – Extracts the new user ID and prepares IT + SaaS payloads.<br>9. Wait - Pre IT – Pause before IT provisioning.<br>10. HTTP Request - IT Provisioning – Creates email, device ticket and core IT accounts.<br>11. Code - Process IT Result – Confirms IT resources and prepares SaaS calls.<br>12. Wait - Pre SaaS – Pause before SaaS application provisioning.<br>13. HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>14. Code - Build Final Status – Aggregates results from all systems into a single status object.<br>15. Code - Finalize Response – Assembles the final onboarding report for the caller.<br>16. Respond to Webhook – Returns the complete onboarding status to the caller.<br>## 3. SaaS & Finish<br><br>Wait - Pre SaaS – Pause before SaaS application provisioning.<br>HTTP Request - SaaS Apps – Provisions access to Slack, Google Workspace, Notion, etc.<br>Code - Build Final Status – Aggregates results from all systems into a single status object.<br>Code - Finalize Response – Assembles the final onboarding report for the caller.<br>Respond to Webhook – Returns the complete onboarding status to the caller. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Webhook - New Hire Request`.
   - Set HTTP Method to `POST`, Path to `employee-onboard`, and Response Mode to `Response Node`.

2. **Add Data Validation & Normalization:**
   - Add a **Code** node named `Code - Validate & Normalize`.
   - Connect `Webhook - New Hire Request` output to this node.
   - Insert JavaScript logic to validate mandatory fields (`firstName`, `lastName`, `email`, `startDate`), generate unique request/employee identifiers, and define endpoint configurations.

3. **Configure HR Provisioning Block:**
   - Add a **Wait** node named `Wait - Pre HR` (Amount: `1` second). Connect `Code - Validate & Normalize` to it.
   - Add an **HTTP Request** node named `HTTP Request - HR System`. Set method to `POST`, URL to `{{ $json.urls.hr }}`.
   - Configure headers (`content-type: application/json`, `X-Request-Id: {{ $json.requestId }}`) and send body as JSON containing employee details. Enable `Never Error` and `Full Response` in options, and toggle `Continue On Fail`.
   - Add a **Code** node named `Code - Process HR Result` to evaluate the HR HTTP response status and assemble the `iamPayload`.

4. **Configure IAM Provisioning Block:**
   - Add a **Wait** node named `Wait - Pre IAM` (Amount: `1` second). Connect `Code - Process HR Result` to it.
   - Add an **HTTP Request** node named `HTTP Request - IAM / Identity`. Set method to `POST`, URL to `{{ $json.urls.iam }}`.
   - Configure JSON body to `{{ JSON.stringify($json.iamPayload) }}` with headers. Enable `Never Error`, `Full Response`, and `Continue On Fail`.
   - Add a **Code** node named `Code - Process IAM Result` to extract the created `userId` and prepare payloads for IT and SaaS steps.

5. **Configure IT Provisioning Block:**
   - Add a **Wait** node named `Wait - Pre IT` (Amount: `1` second). Connect `Code - Process IAM Result` to it.
   - Add an **HTTP Request** node named `HTTP Request - IT Provisioning`. Set method to `POST`, URL to `{{ $json.urls.it }}`.
   - Configure JSON body to `{{ JSON.stringify($json.itPayload) }}` with headers. Enable `Never Error`, `Full Response`, and `Continue On Fail`.
   - Add a **Code** node named `Code - Process IT Result` to confirm IT resources and forward the SaaS payload.

6. **Configure SaaS Provisioning & Completion Block:**
   - Add a **Wait** node named `Wait - Pre SaaS` (Amount: `1` second). Connect `Code - Process IT Result` to it.
   - Add an **HTTP Request** node named `HTTP Request - SaaS Apps`. Set method to `POST`, URL to `{{ $json.urls.saas }}`.
   - Configure JSON body to `{{ JSON.stringify($json.saasPayload) }}` with headers. Enable `Never Error`, `Full Response`, and `Continue On Fail`.
   - Add a **Code** node named `Code - Build Final Status` to aggregate results across HR, IAM, IT, and SaaS into a summary object.
   - Add a **Code** node named `Code - Finalize Response` to compute the dynamic HTTP response code (`201`, `207`, or `502`) and structure the final response envelope.
   - Add a **Respond to Webhook** node named `Respond to Webhook`. Set response body to `{{ $json.body }}`, response code to `{{ $json.httpStatus }}`, and include the `X-Request-Id` response header.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Orchestrates employee onboarding across HR, IAM, IT, and SaaS systems via a sequential webhook architecture. | Workflow Purpose & Domain Architecture |