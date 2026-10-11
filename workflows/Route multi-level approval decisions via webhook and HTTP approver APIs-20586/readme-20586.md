Route multi-level approval decisions via webhook and HTTP approver APIs

https://n8nworkflows.xyz/workflows/route-multi-level-approval-decisions-via-webhook-and-http-approver-apis-20586


# Route multi-level approval decisions via webhook and HTTP approver APIs

### 1. Workflow Overview

This workflow functions as an automated, multi-level approval engine. It exposes an HTTP POST webhook endpoint to receive approval requests (e.g., purchase orders, vendor contracts, system access changes), evaluates them against dynamic policy rules, calculates a risk score, routes the request sequentially through up to three approval tiers via HTTP endpoints, and finally returns an audit-ready approval or rejection decision to the caller.

The execution logic is structured into three functional blocks:
- **1.1 Input Reception & Policy Initialization:** Receives the incoming JSON payload via webhook, validates required parameters, establishes configuration thresholds, and standardizes request attributes.
- **1.2 Multi-Level Authorization Engine:** Computes a composite risk score based on amount, department, request type, and urgency. Determines the required authorization levels and processes each tier sequentially (Level 1, Level 2, and Level 3) using intermediate wait nodes and HTTP approver calls.
- **1.3 Decision Aggregation & Response Delivery:** Aggregates status codes across all evaluated levels, determines the final `approved`, `auto_approved`, or `rejected` state, assembles an audit trail report, and responds to the webhook caller with the appropriate HTTP status code (200 for success/auto-approval, 403 for rejection).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Policy Initialization
This block captures the incoming HTTP request, sanitizes and validates payload fields, and embeds internal configuration parameters and policy limits into the workflow state.

- **Nodes Involved:** 
  - `Webhook - Approval Request`
  - `Code - Validate & Load Policies`
  - `Code - Evaluate Policies`

- **Node Details:**
  - **`Webhook - Approval Request`**
    - *Type & Role:* `n8n-nodes-base.webhook` (Version 2). Acts as the primary HTTP trigger for the workflow.
    - *Configuration:* Listens on path `approval-request` via HTTP POST method using the response node mode.
    - *Key Expressions/Variables:* `webhookId: enterprise-approval-webhook`.
    - *Connections:* Input: None (Trigger). Output: `Code - Validate & Load Policies`.
    - *Version Requirements:* Version 2.
    - *Edge Cases/Failures:* Incoming requests lacking a valid JSON body will result in empty property evaluations down-stream.

  - **`Code - Validate & Load Policies`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). JavaScript execution node that defines global configuration rules, sets endpoint fallback URLs, validates mandatory fields (`requester`, non-negative `amount`), and normalizes payload parameters.
    - *Configuration:* Uses custom JavaScript utilizing helper functions to safely reference upstream node outputs.
    - *Key Expressions/Variables:* Reads body elements from `Webhook - Approval Request`. Supports simulation overrides via `simulateStatus`.
    - *Connections:* Input: `Webhook - Approval Request`. Output: `Code - Evaluate Policies`.
    - *Version Requirements:* Version 2.
    - *Edge Cases/Failures:* Throws expression evaluation errors if upstream payload structure is missing expected types.

  - **`Code - Evaluate Policies`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Computes a numeric risk score (capped at 100) based on request amount, department, type, and urgency, and determines which approval tiers are required.
    - *Configuration:* Evaluates policy thresholds (`autoApproveBelow: 500`, `level1Below: 5000`, `level2Below: 25000`, high-risk departments/types).
    - *Key Expressions/Variables:* References validation output from `Code - Validate & Load Policies`.
    - *Connections:* Input: `Code - Validate & Load Policies`. Output: `Wait - Pre Level 1`.
    - *Version Requirements:* Version 2.
    - *Edge Cases/Failures:* Fails if configuration objects from the previous node are undefined.

---

#### 2.2 Multi-Level Authorization
This block executes the sequential approval gate checks. It introduces short pauses before contacting external approver APIs for Levels 1, 2, and 3, and processes each response to verify approval continuation criteria.

- **Nodes Involved:**
  - `Wait - Pre Level 1`
  - `HTTP Request - Level 1 Approver`
  - `Code - Process Level 1`
  - `Wait - Pre Level 2`
  - `HTTP Request - Level 2 Approver`
  - `Code - Process Level 2`
  - `Wait - Pre Level 3`
  - `HTTP Request - Level 3 Approver`
  - `Code - Process Level 3`

- **Node Details:**
  - **`Wait - Pre Level 1`**
    - *Type & Role:* `n8n-nodes-base.wait` (Version 1.1). Introduces a short delay before executing the Level 1 check.
    - *Configuration:* Pause amount set to 1 second.
    - *Connections:* Input: `Code - Evaluate Policies`. Output: `HTTP Request - Level 1 Approver`.
    - *Version Requirements:* Version 1.1.

  - **`HTTP Request - Level 1 Approver`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Sends a POST request to the Level 1 approver endpoint with request context and risk scoring.
    - *Configuration:* URL evaluated dynamically from workflow data. Configured with `continueOnFail: true` and response settings set to never error on non-2xx status codes while returning the full response.
    - *Key Expressions/Variables:* `={{ $json.urls.level1 }}`, custom header `X-Request-Id`.
    - *Connections:* Input: `Wait - Pre Level 1`. Output: `Code - Process Level 1`.
    - *Version Requirements:* Version 4.2.
    - *Edge Cases/Failures:* Network timeouts or DNS resolution failures on external endpoints; handled gracefully via `continueOnFail` and status code tracking.

  - **`Code - Process Level 1`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Inspects the HTTP response status code from Level 1, records the decision object, and determines whether escalation to Level 2 is required.
    - *Configuration:* JavaScript evaluation verifying if HTTP status falls within the `200–299` range.
    - *Connections:* Input: `HTTP Request - Level 1 Approver`. Output: `Wait - Pre Level 2`.
    - *Version Requirements:* Version 2.

  - **`Wait - Pre Level 2`**
    - *Type & Role:* `n8n-nodes-base.wait` (Version 1.1). Pause before Level 2 authorization check.
    - *Configuration:* Pause amount set to 1 second.
    - *Connections:* Input: `Code - Process Level 1`. Output: `HTTP Request - Level 2 Approver`.
    - *Version Requirements:* Version 1.1.

  - **`HTTP Request - Level 2 Approver`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Sends a POST request to the Level 2 approver endpoint.
    - *Configuration:* Uses dynamic URL parameter `={{ $json.urls.level2 }}`, `continueOnFail: true`, full response capture.
    - *Connections:* Input: `Wait - Pre Level 2`. Output: `Code - Process Level 2`.
    - *Version Requirements:* Version 4.2.

  - **`Code - Process Level 2`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Records the Level 2 decision state and checks if escalation to Level 3 is required.
    - *Connections:* Input: `HTTP Request - Level 2 Approver`. Output: `Wait - Pre Level 3`.
    - *Version Requirements:* Version 2.

  - **`Wait - Pre Level 3`**
    - *Type & Role:* `n8n-nodes-base.wait` (Version 1.1). Pause before Level 3 authorization check.
    - *Configuration:* Pause amount set to 1 second.
    - *Connections:* Input: `Code - Process Level 2`. Output: `HTTP Request - Level 3 Approver`.
    - *Version Requirements:* Version 1.1.

  - **`HTTP Request - Level 3 Approver`**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Sends a POST request to the Level 3 (executive) approver endpoint.
    - *Configuration:* Uses dynamic URL parameter `={{ $json.urls.level3 }}`, `continueOnFail: true`, full response capture.
    - *Connections:* Input: `Wait - Pre Level 3`. Output: `Code - Process Level 3`.
    - *Version Requirements:* Version 4.2.

  - **`Code - Process Level 3`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Records the final Level 3 decision state.
    - *Connections:* Input: `HTTP Request - Level 3 Approver`. Output: `Code - Build Final Decision`.
    - *Version Requirements:* Version 2.

---

#### 2.3 Decision Aggregation & Response Delivery
This block consolidates individual decision nodes, determines the overarching workflow state, formats the audit trail report, and responds to the original webhook caller.

- **Nodes Involved:**
  - `Code - Build Final Decision`
  - `Code - Finalize Response`
  - `Respond to Webhook`

- **Node Details:**
  - **`Code - Build Final Decision`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Iterates through all evaluated approval levels to synthesize a unified status (`approved`, `auto_approved`, or `rejected`) and lists any rejected levels.
    - *Connections:* Input: `Code - Process Level 3`. Output: `Code - Finalize Response`.
    - *Version Requirements:* Version 2.

  - **`Code - Finalize Response`**
    - *Type & Role:* `n8n-nodes-base.code` (Version 2). Prepares the HTTP status code (403 for rejections, 200 otherwise) and packages the full approval report and audit trail.
    - *Connections:* Input: `Code - Build Final Decision`. Output: `Respond to Webhook`.
    - *Version Requirements:* Version 2.

  - **`Respond to Webhook`**
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` (Version 1.1). Returns the structured JSON payload and headers back to the original HTTP webhook requester.
    - *Configuration:* Responds with JSON using dynamic HTTP status code and custom response header `X-Request-Id`.
    - *Key Expressions/Variables:* Response code: `={{ $json.httpStatus }}`, Body: `={{ $json.body }}`.
    - *Connections:* Input: `Code - Finalize Response`. Output: None (Terminal).
    - *Version Requirements:* Version 1.1.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook - Approval Request` | `n8n-nodes-base.webhook` | Receives the approval request payload via POST. | None | `Code - Validate & Load Policies` | ## Enterprise Approval Engine<br><br>Receives an approval request, evaluates dynamic policies to determine required authorization levels, runs multi-level approval checks, and returns a final decision with full audit trail.<br><br>### All nodes (1-line each)<br>1. Webhook - Approval Request – Receives the approval request payload via POST.<br>...<br>15. Respond to Webhook – Returns the final decision and audit trail to the caller. |
| `Code - Validate & Load Policies` | `n8n-nodes-base.code` | Validates the request and loads the dynamic policy rules. | `Webhook - Approval Request` | `Code - Evaluate Policies` | ## Enterprise Approval Engine<br>...<br>## 1. Intake & Policy Evaluation<br><br>Webhook - Approval Request – Receives the approval request payload via POST.<br>Code - Validate & Load Policies – Validates the request and loads the dynamic policy rules.<br>Code - Evaluate Policies – Applies dynamic policies to determine required approval levels and risk score. |
| `Code - Evaluate Policies` | `n8n-nodes-base.code` | Applies dynamic policies to determine required approval levels and risk score. | `Code - Validate & Load Policies` | `Wait - Pre Level 1` | ## Enterprise Approval Engine<br>...<br>## 1. Intake & Policy Evaluation<br>... |
| `Wait - Pre Level 1` | `n8n-nodes-base.wait` | Short pause before Level-1 authorization check. | `Code - Evaluate Policies` | `HTTP Request - Level 1 Approver` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br><br>Wait - Pre Level 1 – Short pause before Level-1 authorization check.<br>... |
| `HTTP Request - Level 1 Approver` | `n8n-nodes-base.httpRequest` | Notifies or checks Level-1 approver (manager / auto-rule). | `Wait - Pre Level 1` | `Code - Process Level 1` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Code - Process Level 1` | `n8n-nodes-base.code` | Records Level-1 decision and decides if higher levels are needed. | `HTTP Request - Level 1 Approver` | `Wait - Pre Level 2` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Wait - Pre Level 2` | `n8n-nodes-base.wait` | Pause before Level-2 authorization check. | `Code - Process Level 1` | `HTTP Request - Level 2 Approver` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `HTTP Request - Level 2 Approver` | `n8n-nodes-base.httpRequest` | Notifies or checks Level-2 approver (director / finance). | `Wait - Pre Level 2` | `Code - Process Level 2` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Code - Process Level 2` | `n8n-nodes-base.code` | Records Level-2 decision and decides if final level is required. | `HTTP Request - Level 2 Approver` | `Wait - Pre Level 3` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Wait - Pre Level 3` | `n8n-nodes-base.wait` | Pause before Level-3 (executive) authorization check. | `Code - Process Level 2` | `HTTP Request - Level 3 Approver` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `HTTP Request - Level 3 Approver` | `n8n-nodes-base.httpRequest` | Notifies or checks Level-3 approver (VP / executive). | `Wait - Pre Level 3` | `Code - Process Level 3` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Code - Process Level 3` | `n8n-nodes-base.code` | Records Level-3 decision. | `HTTP Request - Level 3 Approver` | `Code - Build Final Decision` | ## Enterprise Approval Engine<br>...<br>## 2. Multi-Level Authorization<br>... |
| `Code - Build Final Decision` | `n8n-nodes-base.code` | Aggregates all level results into a final approve/reject decision. | `Code - Process Level 3` | `Code - Finalize Response` | ## Enterprise Approval Engine<br>...<br>## 3. Decision & Respond<br><br>Code - Build Final Decision – Aggregates all level results into a final approve/reject decision.<br>Code - Finalize Response – Assembles the complete approval report with audit trail.<br>Respond to Webhook – Returns the final decision and audit trail to the caller. |
| `Code - Finalize Response` | `n8n-nodes-base.code` | Assembles the complete approval report with audit trail. | `Code - Build Final Decision` | `Respond to Webhook` | ## Enterprise Approval Engine<br>...<br>## 3. Decision & Respond<br>... |
| `Respond to Webhook` | `n8n-nodes-base.respondToWebhook` | Returns the final decision and audit trail to the caller. | `Code - Finalize Response` | None | ## Enterprise Approval Engine<br>...<br>## 3. Decision & Respond<br>... |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to recreate the workflow in n8n manually:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`, v2).
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `approval-request`.
   - Set **Response Mode** to `Response Node`.
   - Set **Webhook ID** to `enterprise-approval-webhook`.

2. **Add Validation and Policy Loading:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Code - Validate & Load Policies`.
   - Configure the JavaScript execution to parse incoming parameters (`requestId`, `type`, `amount`, `department`, `requester`, `description`, `urgency`), initialize default thresholds, and handle optional simulation codes.
   - Connect `Webhook - Approval Request` output to this node.

3. **Add Policy Evaluation:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Code - Evaluate Policies`.
   - Configure the JavaScript to calculate a `riskScore` based on amount limits, high-risk departments/types, and urgency, and determine `requiredLevels`.
   - Connect `Code - Validate & Load Policies` output to this node.

4. **Set Up Level 1 Authorization Pipeline:**
   - Add a **Wait** node (`n8n-nodes-base.wait`, v1.1) named `Wait - Pre Level 1` with a pause amount of `1` second. Connect `Code - Evaluate Policies` to it.
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, v4.2) named `HTTP Request - Level 1 Approver`.
     - Set **Method** to `POST`.
     - Set **URL** to `={{ $json.urls.level1 }}`.
     - Enable **Continue On Fail** (`continueOnFail: true`).
     - Under **Options**, set **Never Error on Status Code** and enable **Full Response**.
     - Add header parameters: `content-type: application/json` and `X-Request-Id: ={{ $json.requestId }}`.
     - Set **Body** to JSON string containing level, request ID, amount, requester, and risk score.
   - Connect `Wait - Pre Level 1` to `HTTP Request - Level 1 Approver`.
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Code - Process Level 1` to process the response status code and check escalation criteria. Connect `HTTP Request - Level 1 Approver` to it.

5. **Set Up Level 2 Authorization Pipeline:**
   - Add a **Wait** node named `Wait - Pre Level 2` (amount: `1`). Connect `Code - Process Level 1` to it.
   - Add an **HTTP Request** node named `HTTP Request - Level 2 Approver` configured identically to Level 1, referencing `={{ $json.urls.level2 }}`. Connect `Wait - Pre Level 2` to it.
   - Add a **Code** node named `Code - Process Level 2` to evaluate Level 2 outcomes and determine Level 3 escalation. Connect `HTTP Request - Level 2 Approver` to it.

6. **Set Up Level 3 Authorization Pipeline:**
   - Add a **Wait** node named `Wait - Pre Level 3` (amount: `1`). Connect `Code - Process Level 2` to it.
   - Add an **HTTP Request** node named `HTTP Request - Level 3 Approver` configured identically to previous tiers, referencing `={{ $json.urls.level3 }}`. Connect `Wait - Pre Level 3` to it.
   - Add a **Code** node named `Code - Process Level 3` to record Level 3 decisions. Connect `HTTP Request - Level 3 Approver` to it.

7. **Build Final Decision and Respond:**
   - Add a **Code** node named `Code - Build Final Decision` to aggregate all level results into a final status (`approved`, `auto_approved`, or `rejected`). Connect `Code - Process Level 3` to it.
   - Add a **Code** node named `Code - Finalize Response` to assign HTTP status codes (`403` for rejections, `200` otherwise) and assemble the audit report. Connect `Code - Build Final Decision` to it.
   - Add a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`, v1.1).
     - Set **Respond With** to `JSON`.
     - Set **Response Body** to `={{ $json.body }}`.
     - Set **Response Code** to `={{ $json.httpStatus }}`.
     - Add response header `X-Request-Id` set to `={{ $json.requestId }}`.
   - Connect `Code - Finalize Response` to `Respond to Webhook`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Enterprise Approval Engine Architecture | Designed for webhook-based multi-tier authorization workflows with automated fallback/simulation options using httpbin endpoints. |