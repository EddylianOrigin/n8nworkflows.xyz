Route purchase approvals by spend, department and vendor risk with Postgres, email and Slack

https://n8nworkflows.xyz/workflows/route-purchase-approvals-by-spend--department-and-vendor-risk-with-postgres--email-and-slack-20587


# Route purchase approvals by spend, department and vendor risk with Postgres, email and Slack

### 1. Workflow Overview

This workflow automates the intake, risk assessment, policy evaluation, and approval routing of corporate purchase requests. It evaluates spend limits (including cumulative 30-day vendor spend to prevent order-splitting), department budget utilization, self-approval rules, and vendor risk levels to determine whether a request should be auto-approved, rejected outright, or routed through a sequential human approval chain (Manager, Procurement, Security, Finance, CFO).

The logic is organized into the following functional blocks:
- **1.1 One-time Setup:** Initializes required PostgreSQL schemas, constraints, sequences, and demo data.
- **1.2 Intake & Validation:** Receives incoming purchase requests via webhook, performs input sanitization, validates currency rates, and converts amounts to USD.
- **1.3 Policy Evaluation:** Queries PostgreSQL for vendor data, departmental budget status, and recent 30-day spend history to apply corporate compliance rules.
- **1.4 Acknowledgment & Routing:** Persists the request as pending, responds to the webhook caller immediately, and switches based on the initial policy determination.
- **1.5 Finalization, Notification & Audit:** Commits approved requests (generating PO numbers and updating budget consumption), writes audit trail entries, notifies the requester via email, and sends an audit message to Slack.
- **1.6 Sequential Approval Chain:** Iterates through required human approvers one level at a time via interactive email links, handling timeout reminders, expirations, and decision logging.

---

### 2. Block-by-Block Analysis

#### 2.1 One-time Setup
- **Overview:** Initializes the database tables, indices, and sequences required for tracking vendors, departments, requests, and audit logs, alongside pre-configured demo records.
- **Nodes Involved:** `Manual Trigger - Run Setup Once`, `Postgres - Create Approval Tables`
- **Node Details:**
  - **Manual Trigger - Run Setup Once**
    - *Type/Role:* `n8n-nodes-base.manualTrigger` — Provides the manual execution entry point to build the database schema.
    - *Configuration:* Default settings.
    - *Connections:* Outputs to `Postgres - Create Approval Tables`.
  - **Postgres - Create Approval Tables**
    - *Type/Role:* `n8n-nodes-base.postgres` — Executes a multi-statement DDL and DML query creating tables (`pa_vendors`, `pa_departments`, `pa_requests`, `pa_audit_log`), indices, a PO number sequence (`pa_po_seq`), and seeding demo records.
    - *Configuration:* Operation set to `executeQuery`.
    - *Connections:* Input from `Manual Trigger - Run Setup Once`. Output (none/terminal setup block).
    - *Edge Cases:* Database user must possess DDL privileges. Fails if connection parameters are invalid.

#### 2.2 Intake & Validation
- **Overview:** Listens for incoming POST requests, applies configuration parameters (currency rates, spend tiers, sender details), sanitizes inputs, and verifies payload integrity.
- **Nodes Involved:** `Webhook - Purchase Request`, `Set Approval Policy`, `Code - Validate & Normalize Request`, `Request Valid?`, `Respond - Validation Error`
- **Node Details:**
  - **Webhook - Purchase Request**
    - *Type/Role:* `n8n-nodes-base.webhook` — Receives external HTTP POST payloads at path `purchase-request`.
    - *Configuration:* Method: `POST`, Response Mode: `responseNode`, Webhook ID: `be10ff34-e936-51a5-9d45-0341accf0991`.
    - *Connections:* Output to `Set Approval Policy`.
    - *Edge Cases:* Missing payload or incorrect content-type headers.
  - **Set Approval Policy**
    - *Type/Role:* `n8n-nodes-base.set` — Injects global business rules including currency conversion rates, spend tier thresholds, approver email mapping, review limits, and notification configurations.
    - *Configuration:* Assignments defining JSON structures for `currencyRates`, `spendTiers`, `approvers`, and scalar thresholds (`maxRequestUsd`, `reminderHours`, etc.).
    - *Connections:* Input from `Webhook - Purchase Request`. Output to `Code - Validate & Normalize Request`.
  - **Code - Validate & Normalize Request**
    - *Type/Role:* `n8n-nodes-base.code` — Validates incoming fields (`requesterEmail`, `department`, `vendorId`, `amount`, `currency`, `description`), generates a unique request ID (`PR-[TIMESTAMP]-[RAND]`), and computes the normalized `amountUsd`.
    - *Configuration:* JavaScript execution block enforcing regex validations and currency mapping.
    - *Connections:* Input from `Set Approval Policy`. Output to `Request Valid?`.
  - **Request Valid?**
    - *Type/Role:* `n8n-nodes-base.if` — Branching node evaluating whether validation errors occurred.
    - *Configuration:* Condition checks `={{ $json.valid }}` is true.
    - *Connections:* Input from `Code - Validate & Normalize Request`. True branch connects to `Postgres - Load Vendor, Budget & History`. False branch connects to `Respond - Validation Error`.
  - **Respond - Validation Error**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` — Returns an HTTP 400 response with validation error details.
    - *Configuration:* Response Code: `400`, JSON payload containing `$json.errors`.
    - *Connections:* Input from `Request Valid?` (false branch).

#### 2.3 Policy Evaluation
- **Overview:** Queries database state regarding vendor risk, department budgets, and recent transactions, then executes comprehensive business logic to decide the routing path.
- **Nodes Involved:** `Postgres - Load Vendor, Budget & History`, `Code - Evaluate Approval Policy`, `Postgres - Save Request`, `Respond - Request Received`, `Switch - Route Policy Decision`
- **Node Details:**
  - **Postgres - Load Vendor, Budget & History**
    - *Type/Role:* `n8n-nodes-base.postgres` — Fetches vendor risk/approval status, department manager details, annual budget, current spend/commitments, and the trailing 30-day spend sum for the vendor within the department.
    - *Configuration:* Query execution using query replacements derived from validated request fields (`vendorId`, `department`).
    - *Connections:* Input from `Request Valid?`. Output to `Code - Evaluate Approval Policy`.
  - **Code - Evaluate Approval Policy**
    - *Type/Role:* `n8n-nodes-base.code` — Evaluates spend tiers (including cumulative 30-day spend anti-order-splitting rules), department freezing, budget overruns, vendor risk status (low, medium, high, blocked), and self-approval avoidance (reassigning manager approval to finance if requester is the manager).
    - *Configuration:* Complex JavaScript evaluation determining `decision` (`auto_approve`, `needs_approval`, `reject`), required approval levels, flags, and reasons.
    - *Connections:* Input from `Postgres - Load Vendor, Budget & History`. Output to `Postgres - Save Request`.
  - **Postgres - Save Request**
    - *Type/Role:* `n8n-nodes-base.postgres` — Inserts the purchase request into `pa_requests` with a `pending` status.
    - *Configuration:* SQL `INSERT` statement utilizing parameter replacements for request metadata and JSONB arrays.
    - *Connections:* Input from `Code - Evaluate Approval Policy`. Output to `Respond - Request Received`.
  - **Respond - Request Received**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` — Returns an HTTP 202 response acknowledging request ingestion, detailing status, risk level, and required approval steps.
    - *Configuration:* Response Code: `202`, JSON payload mapping evaluation outcomes.
    - *Connections:* Input from `Postgres - Save Request`. Output to `Switch - Route Policy Decision`.
  - **Switch - Route Policy Decision**
    - *Type/Role:* `n8n-nodes-base.switch` — Routes the workflow flow based on the determined decision.
    - *Configuration:* Rules evaluating `={{ $('Code - Evaluate Approval Policy').first().json.decision }}` for outputs: `auto_approve`, `needs_approval`, `reject`.
    - *Connections:* Input from `Respond - Request Received`. Outputs map to `Set Final - Auto Approved`, `Code - Build Approval Step`, and `Set Final - Policy Rejected`.

#### 2.4 Finalization, Notification & Audit
- **Overview:** Processes terminal policy decisions or completed approval chains by generating purchase order numbers, updating departmental budget commitments, logging audit events, and dispatching notifications via email and Slack.
- **Nodes Involved:** `Set Final - Auto Approved`, `Set Final - Policy Rejected`, `Postgres - Finalize Request`, `Code - Build Outcome Message`, `Email - Notify Requester`, `HTTP - Slack Audit Notification`
- **Node Details:**
  - **Set Final - Auto Approved**
    - *Type/Role:* `n8n-nodes-base.set` — Formats data variables for an auto-approved request.
    - *Configuration:* Sets `finalStatus` to `approved` and `decidedBy` to `policy`.
    - *Connections:* Input from `Switch - Route Policy Decision` (`auto_approve`). Output to `Postgres - Finalize Request`.
  - **Set Final - Policy Rejected**
    - *Type/Role:* `n8n-nodes-base.set` — Formats data variables for a policy-rejected request.
    - *Configuration:* Sets `finalStatus` to `rejected` and `decidedBy` to `policy`.
    - *Connections:* Input from `Switch - Route Policy Decision` (`reject`). Output to `Postgres - Finalize Request`.
  - **Postgres - Finalize Request**
    - *Type/Role:* `n8n-nodes-base.postgres` — Updates the request status in PostgreSQL, assigns a PO number via sequence `pa_po_seq` if approved, commits the USD amount to the department budget, and writes an audit log entry transactionally.
    - *Configuration:* CTE-based SQL query (`WITH r AS (...), b AS (...), a AS (...)`).
    - *Connections:* Input from `Set Final - Auto Approved`, `Set Final - Policy Rejected`, or approval chain switches. Output to `Code - Build Outcome Message`.
  - **Code - Build Outcome Message**
    - *Type/Role:* `n8n-nodes-base.code` — Generates formatted HTML email body templates and Slack markdown audit strings based on request outcome.
    - *Configuration:* JavaScript parsing of final status, financial metrics, and reasons.
    - *Connections:* Input from `Postgres - Finalize Request`. Outputs to `Email - Notify Requester` and `HTTP - Slack Audit Notification`.
  - **Email - Notify Requester**
    - *Type/Role:* `n8n-nodes-base.emailSend` — Sends the final status notification email to the requester.
    - *Configuration:* SMTP credentials, dynamic `toEmail`, `subject`, and HTML body. Includes retry logic (`maxTries: 3`).
    - *Connections:* Input from `Code - Build Outcome Message`.
  - **HTTP - Slack Audit Notification**
    - *Type/Role:* `n8n-nodes-base.httpRequest` — Posts audit notifications to the configured Slack webhook channel.
    - *Configuration:* Method: `POST`, URL: `={{ $('Set Approval Policy').first().json.slackWebhookUrl }}`, JSON payload containing `text`. Error handling set to `continueRegularOutput`.
    - *Connections:* Input from `Code - Build Outcome Message`.

#### 2.5 Sequential Approval Chain
- **Overview:** Iterates sequentially through assigned human approver levels, sending dynamic approval/rejection emails with embedded secure links, awaiting webhook resumption, processing responses or reminder timeouts, and recording audit events.
- **Nodes Involved:** `Code - Build Approval Step`, `Email - Send Approval Request`, `Wait - For Approver Decision`, `Code - Process Decision`, `Switch - Route Approval Outcome`, `Postgres - Record Approval Decision`
- **Node Details:**
  - **Code - Build Approval Step**
    - *Type/Role:* `n8n-nodes-base.code` — Identifies the current approval level from the sequence, generates a secure random token and resume URL pointing back to the n8n execution instance, and formats the approval request email table.
    - *Configuration:* JavaScript handling execution resume URL parsing and index tracking (`levelIndex`).
    - *Connections:* Input from `Switch - Route Policy Decision` (`needs_approval`) or `Switch - Route Approval Outcome` (`continue`). Output to `Email - Send Approval Request`.
  - **Email - Send Approval Request**
    - *Type/Role:* `n8n-nodes-base.emailSend` — Sends the approval request or reminder email to the designated approver.
    - *Configuration:* SMTP configuration, dynamic recipient and email body. Retry on fail enabled.
    - *Connections:* Input from `Code - Build Approval Step`. Output to `Wait - For Approver Decision`.
  - **Wait - For Approver Decision**
    - *Type/Role:* `n8n-nodes-base.wait` — Pauses execution awaiting an incoming webhook callback from the approver's action link or a reminder timeout.
    - *Configuration:* Resume mechanism set to `webhook`, limit wait time enabled, resume amount bound to configured `reminderHours`.
    - *Connections:* Input from `Email - Send Approval Request`. Output to `Code - Process Decision`.
  - **Code - Process Decision**
    - *Type/Role:* `n8n-nodes-base.code` — Analyzes incoming webhook query parameters (`decision`, `token`, `comment`). Validates tokens, handles reminder increments and maximum reminder limits (triggering expiry if exceeded), advances to the next approval level upon approval, or registers rejection.
    - *Configuration:* JavaScript routing and audit state construction.
    - *Connections:* Input from `Wait - For Approver Decision`. Outputs to both `Switch - Route Approval Outcome` and `Postgres - Record Approval Decision`.
  - **Switch - Route Approval Outcome**
    - *Type/Role:* `n8n-nodes-base.switch` — Routes based on decision outcome processing.
    - *Configuration:* Rules matching `$json.route` to `continue`, `approved`, or `rejected`.
    - *Connections:* Input from `Code - Process Decision`. `continue` routes back to `Code - Build Approval Step`. `approved` and `rejected` routes proceed to `Postgres - Finalize Request`.
  - **Postgres - Record Approval Decision**
    - *Type/Role:* `n8n-nodes-base.postgres` — Appends audit log entries for individual steps (reminders, approvals, rejections, invalid tokens, expirations) to `pa_audit_log`.
    - *Configuration:* SQL `INSERT` statement utilizing parameter substitution.
    - *Connections:* Input from `Code - Process Decision`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation and setup guide covering workflow logic, rules, and layout mapping. | None | None | ## Purchase Approval by Spend Limit, Department and Vendor Risk<br><br>### How it works... |
| Note: 1. One-time setup | `n8n-nodes-base.stickyNote` | Label for the setup block. | None | None | ## 1. One-time setup<br><br>Run manually once... |
| Note: 2. Intake & validation | `n8n-nodes-base.stickyNote` | Label for the intake & validation block. | None | None | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Note: 4. Acknowledge & route | `n8n-nodes-base.stickyNote` | Label for acknowledgment and routing block. | None | None | ## 4. Acknowledge & route<br><br>Answers the requester immediately... |
| Note: 5. Finalize, notify & audit | `n8n-nodes-base.stickyNote` | Label for finalization, notification & audit block. | None | None | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Note: 6. Sequential approval chain | `n8n-nodes-base.stickyNote` | Label for sequential approval chain block. | None | None | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Manual Trigger - Run Setup Once | `n8n-nodes-base.manualTrigger` | Manual trigger to execute database initialization. | None | Postgres - Create Approval Tables | ## 1. One-time setup<br><br>Run manually once... |
| Postgres - Create Approval Tables | `n8n-nodes-base.postgres` | Creates database tables, indices, sequences, and seeds demo records. | Manual Trigger - Run Setup Once | None | ## 1. One-time setup<br><br>Run manually once... |
| Webhook - Purchase Request | `n8n-nodes-base.webhook` | Receives incoming POST purchase requests. | None | Set Approval Policy | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Set Approval Policy | `n8n-nodes-base.set` | Injects global policy configuration, thresholds, and currency rates. | Webhook - Purchase Request | Code - Validate & Normalize Request | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Code - Validate & Normalize Request | `n8n-nodes-base.code` | Validates request payloads and calculates normalized USD amounts. | Set Approval Policy | Request Valid? | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Request Valid? | `n8n-nodes-base.if` | Evaluates validation success. | Code - Validate & Normalize Request | Postgres - Load Vendor, Budget & History, Respond - Validation Error | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Respond - Validation Error | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 error response for invalid requests. | Request Valid? | None | ## 2. Intake & validation<br><br>`POST /purchase-request`... |
| Postgres - Load Vendor, Budget & History | `n8n-nodes-base.postgres` | Loads vendor risk status, budget details, and 30-day vendor spend. | Request Valid? | Code - Evaluate Approval Policy | ## 3. Policy: spend, department & vendor risk<br><br>Loads the vendor and department budget... |
| Code - Evaluate Approval Policy | `n8n-nodes-base.code` | Applies spend limits, budget limits, vendor risk rules, and self-approval checks. | Postgres - Load Vendor, Budget & History | Postgres - Save Request | ## 3. Policy: spend, department & vendor risk<br><br>Loads the vendor and department budget... |
| Postgres - Save Request | `n8n-nodes-base.postgres` | Stores pending request records in PostgreSQL. | Code - Evaluate Approval Policy | Respond - Request Received | ## 4. Acknowledge & route<br><br>Answers the requester immediately... |
| Respond - Request Received | `n8n-nodes-base.respondToWebhook` | Responds immediately to webhook caller with request ID and status. | Postgres - Save Request | Switch - Route Policy Decision | ## 4. Acknowledge & route<br><br>Answers the requester immediately... |
| Switch - Route Policy Decision | `n8n-nodes-base.switch` | Routes workflow based on policy decision (`auto_approve`, `needs_approval`, `reject`). | Respond - Request Received | Set Final - Auto Approved, Code - Build Approval Step, Set Final - Policy Rejected | ## 4. Acknowledge & route<br><br>Answers the requester immediately... |
| Set Final - Auto Approved | `n8n-nodes-base.set` | Assigns auto-approval finalization parameters. | Switch - Route Policy Decision | Postgres - Finalize Request | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Set Final - Policy Rejected | `n8n-nodes-base.set` | Assigns policy rejection finalization parameters. | Switch - Route Policy Decision | Postgres - Finalize Request | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Postgres - Finalize Request | `n8n-nodes-base.postgres` | Commits decision, updates budget commitments, generates PO numbers, and logs audit entries. | Set Final - Auto Approved, Set Final - Policy Rejected, Switch - Route Approval Outcome | Code - Build Outcome Message | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Code - Build Outcome Message | `n8n-nodes-base.code` | Builds HTML email body and Slack notification payload. | Postgres - Finalize Request | Email - Notify Requester, HTTP - Slack Audit Notification | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Email - Notify Requester | `n8n-nodes-base.emailSend` | Sends outcome notification email to requester. | Code - Build Outcome Message | None | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| HTTP - Slack Audit Notification | `n8n-nodes-base.httpRequest` | Posts audit log notification to Slack webhook. | Code - Build Outcome Message | None | ## 5. Finalize, notify & audit<br><br>Commits the decision... |
| Code - Build Approval Step | `n8n-nodes-base.code` | Prepares current approval step, secure token, and dynamic approval email. | Switch - Route Policy Decision, Switch - Route Approval Outcome | Email - Send Approval Request | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Email - Send Approval Request | `n8n-nodes-base.emailSend` | Sends approval request or reminder email to current approver. | Code - Build Approval Step | Wait - For Approver Decision | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Wait - For Approver Decision | `n8n-nodes-base.wait` | Pauses execution awaiting approver response webhook or reminder timeout. | Email - Send Approval Request | Code - Process Decision | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Code - Process Decision | `n8n-nodes-base.code` | Processes approver action or timeout, checks token matching, and handles reminder limits. | Wait - For Approver Decision | Switch - Route Approval Outcome, Postgres - Record Approval Decision | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Switch - Route Approval Outcome | `n8n-nodes-base.switch` | Routes based on approval response outcome (`continue`, `approved`, `rejected`). | Code - Process Decision | Code - Build Approval Step, Postgres - Finalize Request | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Postgres - Record Approval Decision | `n8n-nodes-base.postgres` | Records approval chain step events in audit log. | Code - Process Decision | None | ## 6. Sequential approval chain<br><br>Approvers are asked one level at a time... |
| Note: 3. Policy: spend, department & vendor risk | `n8n-nodes-base.stickyNote` | Label for policy evaluation block. | None | None | ## 3. Policy: spend, department & vendor risk<br><br>Loads the vendor and department budget... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Database Setup
1. Create a **Manual Trigger** node named `Manual Trigger - Run Setup Once`.
2. Create a **Postgres** node named `Postgres - Create Approval Tables`. Set operation to `executeQuery` and paste the DDL statements for `pa_vendors`, `pa_departments`, `pa_requests`, `pa_audit_log`, sequence `pa_po_seq`, and demo inserts. Connect the Manual Trigger to this node.

#### Step 2: Configure Intake & Validation
3. Create a **Webhook** node named `Webhook - Purchase Request`. Set HTTP Method to `POST`, Path to `purchase-request`, and Response Mode to `responseNode`.
4. Create a **Set** node named `Set Approval Policy`. Define assignments for `currencyRates`, `spendTiers`, `approvers`, `maxRequestUsd`, `procurementReviewOverUsd`, `requireSecurityForHighRisk`, `reminderHours`, `maxReminders`, `fromEmail`, and `slackWebhookUrl`. Connect `Webhook - Purchase Request` to this node.
5. Create a **Code** node named `Code - Validate & Normalize Request`. Implement the validation script checking email formats, department names, positive amounts, valid currencies, and description lengths. Connect `Set Approval Policy` here.
6. Create an **If** node named `Request Valid?`. Check condition `={{ $json.valid }}` equals true. Connect `Code - Validate & Normalize Request` to it.
7. Create a **Respond to Webhook** node named `Respond - Validation Error`. Set response code to `400` and return JSON errors. Connect the *false* branch of `Request Valid?` here.

#### Step 3: Policy Evaluation & Persistence
8. Create a **Postgres** node named `Postgres - Load Vendor, Budget & History`. Use `executeQuery` with a SQL query joining `pa_vendors` and `pa_departments` and aggregating trailing 30-day spend for order-splitting checks. Connect the *true* branch of `Request Valid?` here.
9. Create a **Code** node named `Code - Evaluate Approval Policy`. Implement logic checking department freezing, vendor risk tiers, cumulative 30-day spend thresholds, max request limits, budget overruns, and self-approval avoidance. Connect `Postgres - Load Vendor, Budget & History` here.
10. Create a **Postgres** node named `Postgres - Save Request` to insert pending requests into `pa_requests`. Connect `Code - Evaluate Approval Policy` here.
11. Create a **Respond to Webhook** node named `Respond - Request Received`. Set response code to `202` returning request ID, status, and required approvals. Connect `Postgres - Save Request` here.
12. Create a **Switch** node named `Switch - Route Policy Decision`. Configure rules evaluating `={{ $('Code - Evaluate Approval Policy').first().json.decision }}` for outputs: `auto_approve`, `needs_approval`, and `reject`. Connect `Respond - Request Received` to this node.

#### Step 4: Finalization, Notification & Audit
13. Create two **Set** nodes named `Set Final - Auto Approved` and `Set Final - Policy Rejected`. Bind them to `auto_approve` and `reject` outputs of `Switch - Route Policy Decision` respectively, setting final status and decision metadata.
14. Create a **Postgres** node named `Postgres - Finalize Request`. Implement the CTE SQL query updating request status, assigning PO numbers via sequence `pa_po_seq` upon approval, updating department committed balances, and writing audit logs. Connect both final set nodes and the outcome switch nodes here.
15. Create a **Code** node named `Code - Build Outcome Message` to construct email HTML templates and Slack markdown strings. Connect `Postgres - Finalize Request` here.
16. Create an **Email Send** node named `Email - Notify Requester` using SMTP credentials, binding `toEmail`, `subject`, and HTML body. Connect `Code - Build Outcome Message` here.
17. Create an **HTTP Request** node named `HTTP - Slack Audit Notification`. Set method to `POST`, URL to the Slack webhook parameter, and send JSON body. Connect `Code - Build Outcome Message` here.

#### Step 5: Sequential Approval Chain
18. Create a **Code** node named `Code - Build Approval Step`. Implement logic resolving current approval levels, generating secure tokens and resume URLs, and formatting tables. Connect the `needs_approval` output of `Switch - Route Policy Decision` and the `continue` output of `Switch - Route Approval Outcome` here.
19. Create an **Email Send** node named `Email - Send Approval Request` to dispatch approval emails to approvers. Connect `Code - Build Approval Step` here.
20. Create a **Wait** node named `Wait - For Approver Decision`. Set resume to `webhook`, enable limit wait time, and bind `resumeAmount` to `reminderHours`. Connect `Email - Send Approval Request` here.
21. Create a **Code** node named `Code - Process Decision` to process incoming query parameters, validate tokens, handle reminder counts, trigger expirations, or advance approval indexes. Connect `Wait - For Approver Decision` here.
22. Create a **Switch** node named `Switch - Route Approval Outcome`. Configure rules matching `$json.route` to `continue`, `approved`, and `rejected`. Connect `Code - Process Decision` here.
23. Create a **Postgres** node named `Postgres - Record Approval Decision` to log audit events for reminders, approvals, rejections, and token mismatches. Connect `Code - Process Decision` here.
24. Connect `Switch - Route Approval Outcome` outputs: `continue` routes to `Code - Build Approval Step`; `approved` and `rejected` route to `Postgres - Finalize Request`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Purchase Approval by Spend Limit, Department and Vendor Risk | Workflow title describing core functional domain. |
| Slack Webhook Integration | Configured via `slackWebhookUrl` parameter placeholder in the policy set node. |
| Database Initialization | Requires a one-time manual execution of the setup trigger to seed schemas and demo records. |