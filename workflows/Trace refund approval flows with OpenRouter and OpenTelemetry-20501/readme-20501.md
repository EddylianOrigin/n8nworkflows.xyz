Trace refund approval flows with OpenRouter and OpenTelemetry

https://n8nworkflows.xyz/workflows/trace-refund-approval-flows-with-openrouter-and-opentelemetry-20501


# Trace refund approval flows with OpenRouter and OpenTelemetry

### 1. Workflow Overview

This workflow automates the processing and human-in-the-loop review of refund requests, integrating AI-driven proposal generation with OpenTelemetry tracing. Its primary purpose is to take an incoming refund claim, validate and enrich its context via a sub-workflow, pass it through an instrumented policy check, generate an AI-backed recommendation using OpenRouter, and pause execution safely until a human reviewer accepts or rejects the proposal via a unique resume URL. 

The execution logic is divided into three functional blocks:
- **1.1 Input Reception & Enrichment:** Captures the initial HTTP POST webhook and forwards the request to an external sub-workflow to attach simulated customer tier data and history.
- **1.2 Policy Evaluation & AI Proposal Generation:** Executes an HTTP call carrying telemetry context, invokes an advanced OpenRouter chat model via an AI agent node to draft a contextual recommendation, and responds to the initial webhook caller with the proposal and a secure resume endpoint.
- **1.3 Human Review & Validation:** Pauses the workflow until an authorized reviewer submits a decision, validates the response structure, and either records the approval state or terminates with an error.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Enrichment
- **Overview:** This block captures the initial incoming webhook request containing refund metrics and pushes it to an enrichment sub-workflow to retrieve contextual customer data.
- **Nodes Involved:** 
  - `Webhook - Approval Request`
  - `Enrich Request Sub-workflow`
- **Node Details:**
  - `Webhook - Approval Request`:
    - *Type & Role:* `n8n-nodes-base.webhook` (Trigger). Listens for incoming HTTP POST requests at the `audit-trace-approval` path.
    - *Configuration:* HTTP Method: POST, Response Mode: Response Node.
    - *Expressions:* None (raw payload ingestion).
    - *Connections:* Output connects to `Enrich Request Sub-workflow`.
    - *Edge Cases:* Missing payload parameters, malformed JSON, or lack of expected `traceparent` headers for OpenTelemetry.
  - `Enrich Request Sub-workflow`:
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Action). Calls a modular child workflow to validate and append customer tier and prior refund metrics.
    - *Configuration:* Execution option `waitForSubWorkflow` set to true. Workflow ID targets the Exercise 5 enrichment sub-workflow.
    - *Expressions:* Inherits and passes upstream webhook payload.
    - *Input/Output:* Input from `Webhook - Approval Request`; Output connects to `Policy Service Call`.
    - *Edge Cases:* Unlinked or unpublished target sub-workflow throwing missing ID errors.
    - *Sub-workflow Reference:* Requires the Exercise 5 enrichment sub-workflow to be imported, saved, and published.

#### 2.2 Policy Evaluation & AI Proposal Generation
- **Overview:** This block performs a telemetry-tracked policy service call, drafts an AI-powered evaluation using OpenRouter, and returns the preliminary proposal and resume link back to the calling client before pausing.
- **Nodes Involved:**
  - `Policy Service Call`
  - `Approval Proposal Agent`
  - `OpenRouter - Approval`
  - `Return Approval Request`
- **Node Details:**
  - `Policy Service Call`:
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Action). Sends an HTTP request to an echo service to propagate and inspect trace context headers.
    - *Configuration:* URL targets `https://postman-echo.com/get`.
    - *Expressions:* None.
    - *Input/Output:* Input from `Enrich Request Sub-workflow`; Output connects to `Approval Proposal Agent`.
    - *Edge Cases:* Network timeouts, DNS resolution failures, or external echo service unreliability.
  - `Approval Proposal Agent`:
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Node). Acts as an intermediary refund policy assistant synthesizing customer metrics into a review proposal.
    - *Configuration:* Prompt type defined inline with custom system messages directing the model to act strictly as a recommender.
    - *Expressions:* Dynamic text injection using upstream properties: `{{ $('Enrich Request Sub-workflow').item.json.requestId }}`, `amount`, `customerTier`, `priorRefunds12mo`, and `reason`.
    - *Input/Output:* Main input from `Policy Service Call`; AI Model input from `OpenRouter - Approval`; Output connects to `Return Approval Request`.
    - *Edge Cases:* Token limit exhaustion, API rate limits, or unexpected output parsing failures.
  - `OpenRouter - Approval`:
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (AI Model Sub-node). Configures the underlying large language model provider.
    - *Configuration:* Model specified as `openai/gpt-4.1-mini`.
    - *Credentials:* Requires an active OpenRouter API credential.
    - *Input/Output:* Connects exclusively as an AI language model provider to `Approval Proposal Agent`.
    - *Edge Cases:* Invalid API keys, depleted account credits, or service outages on OpenRouter.
  - `Return Approval Request`:
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` (Action). Returns an immediate HTTP response to the original webhook caller containing the proposal, execution ID, and unique resume URL.
    - *Configuration:* Respond with JSON.
    - *Expressions:* `={{ JSON.stringify({ requestId: $("Enrich Request Sub-workflow").first().json.requestId, proposal: $json.output, executionId: $execution.id, resumeUrl: $execution.resumeUrl }) }}`
    - *Input/Output:* Input from `Approval Proposal Agent`; Output connects to `Wait for Human Approval`.
    - *Edge Cases:* Expression evaluation failures if AI output is empty or malformed.

#### 2.3 Human Review & Validation
- **Overview:** This block pauses execution until a reviewer posts a decision to the resume endpoint, validates the response structure, and either archives a successful review or halts execution due to invalid input.
- **Nodes Involved:**
  - `Wait for Human Approval`
  - `Validate Approval`
  - `Reject Approval`
  - `Record Approval`
- **Node Details:**
  - `Wait for Human Approval`:
    - *Type & Role:* `n8n-nodes-base.wait` (Flow Control). Halts the workflow execution until an external POST request hits the generated resume URL.
    - *Configuration:* Resume mode set to Webhook. Response data set to `firstEntryJson`, response mode set to `lastNode`. Webhook ID configured.
    - *Input/Output:* Input from `Return Approval Request`; Output connects to `Validate Approval`.
    - *Edge Cases:* Webhook expiration, incorrect resume URL invocation, or unauthenticated external access in production.
  - `Validate Approval`:
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Evaluates whether the incoming approval payload conforms to required data types and structural criteria.
    - *Configuration:* Loose type validation enabled. Combinator set to `and`.
    - *Expressions:* `={{ typeof $json.body.approved === 'boolean' && typeof $json.body.approver === 'string' && $json.body.approver.trim() !== '' }}`
    - *Input/Output:* Input from `Wait for Human Approval`; True branch connects to `Record Approval`, False branch connects to `Reject Approval`.
    - *Edge Cases:* Missing body parameters resulting in runtime evaluation errors if not safely typed.
  - `Reject Approval`:
    - *Type & Role:* `n8n-nodes-base.stopAndError` (Action). Terminates the execution path with a defined error message when validation fails.
    - *Configuration:* Error message states: `approved must be a JSON boolean and approver is required for this demo`.
    - *Input/Output:* Input from the false branch of `Validate Approval`. No outputs.
  - `Record Approval`:
    - *Type & Role:* `n8n-nodes-base.set` (Action). Aggregates and structures the final audit record parameters upon successful review validation.
    - *Configuration:* Field assignments configured for audit properties (`requestId`, `proposal`, `submittedApprover`, `approved`, `decision`, `recordedAt`, `paymentProcessed`).
    - *Expressions:* Pulls values via expressions targeting `Enrich Request Sub-workflow`, `Approval Proposal Agent`, and the incoming webhook body (`$json.body`). Uses `{{ $now.toISO() }}` for timestamp generation.
    - *Input/Output:* Input from the true branch of `Validate Approval`. No further primary downstream outputs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook - Approval Request` | `n8n-nodes-base.webhook` | Receives incoming POST requests with refund details | None | `Enrich Request Sub-workflow` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Enrich Request Sub-workflow` | `n8n-nodes-base.executeWorkflow` | Appends customer context via a sub-workflow | `Webhook - Approval Request` | `Policy Service Call` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Policy Service Call` | `n8n-nodes-base.httpRequest` | Sends an HTTP request carrying trace context | `Enrich Request Sub-workflow` | `Approval Proposal Agent` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Approval Proposal Agent` | `@n8n/n8n-nodes-langchain.agent` | Drafts a review proposal using an LLM | `Policy Service Call`, `OpenRouter - Approval` | `Return Approval Request` | Trace a human approval across a wait with OpenTelemetry... |
| `OpenRouter - Approval` | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Provides OpenRouter model config to the AI agent | None | `Approval Proposal Agent` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Return Approval Request` | `n8n-nodes-base.respondToWebhook` | Responds to caller with proposal and resume URL | `Approval Proposal Agent` | `Wait for Human Approval` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Wait for Human Approval` | `n8n-nodes-base.wait` | Pauses execution until reviewer posts a decision | `Return Approval Request` | `Validate Approval` | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Template Notes` | `n8n-nodes-base.stickyNote` | Documents overview, steps, and setup guidelines | None | None | Trace a human approval across a wait with OpenRouter and OpenTelemetry... |
| `Validate Approval` | `n8n-nodes-base.if` | Verifies decision structure and required fields | `Wait for Human Approval` | `Record Approval`, `Reject Approval` | |
| `Reject Approval` | `n8n-nodes-base.stopAndError` | Fails execution when approval payload is invalid | `Validate Approval` | None | |
| `Record Approval` | `n8n-nodes-base.set` | Formats final audit record and decision metadata | `Validate Approval` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually inside an n8n instance:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Webhook - Approval Request`.
   - Set **HTTP Method** to `POST` and **Path** to `audit-trace-approval`. Set **Response Mode** to `Response Node`.
2. **Add the Enrichment Sub-Workflow Node:**
   - Add an **Execute Workflow** node named `Enrich Request Sub-workflow`.
   - Enable **Wait for Sub-Workflow** under options.
   - Select and link the pre-imported Exercise 5 enrichment sub-workflow in the workflow ID picker.
   - Connect `Webhook - Approval Request` output to this node.
3. **Configure the Policy Service Call:**
   - Add an **HTTP Request** node named `Policy Service Call`.
   - Set the method to `GET` and **URL** to `https://postman-echo.com/get`.
   - Connect `Enrich Request Sub-workflow` output to this node.
4. **Setup the AI Agent and Language Model:**
   - Add an **OpenAI Chat Model** (OpenRouter integration variant) named `OpenRouter - Approval`.
   - Configure the model field to `openai/gpt-4.1-mini`.
   - Configure credentials to use your valid OpenRouter API key.
   - Add an **AI Agent** node named `Approval Proposal Agent`.
   - Set **Prompt Type** to `Define` and paste the text expression: `=Request {{ $('Enrich Request Sub-workflow').item.json.requestId }}: amount {{ $('Enrich Request Sub-workflow').item.json.amount }} cents, tier {{ $('Enrich Request Sub-workflow').item.json.customerTier }}, prior refunds {{ $('Enrich Request Sub-workflow').item.json.priorRefunds12mo }}, reason {{ $('Enrich Request Sub-workflow').item.json.reason }}`
   - Open options, add a **System Message**: `You are a refund policy assistant. You propose a decision for a human approver, you never decide.\n\nGiven the enriched request, reply with one short paragraph recommending approve or decline, citing the amount, the customer tier, and the number of prior refunds in the last 12 months.`
   - Connect `OpenRouter - Approval` to the `ai_languageModel` input of `Approval Proposal Agent`.
   - Connect `Policy Service Call` main output to `Approval Proposal Agent`.
5. **Configure the Webhook Response:**
   - Add a **Respond to Webhook** node named `Return Approval Request`.
   - Set **Respond With** to `JSON`.
   - Set **Response Body** expression to: `={{ JSON.stringify({ requestId: $("Enrich Request Sub-workflow").first().json.requestId, proposal: $json.output, executionId: $execution.id, resumeUrl: $execution.resumeUrl }) }}`
   - Connect `Approval Proposal Agent` main output to this node.
6. **Configure the Human Review Wait State:**
   - Add a **Wait** node named `Wait for Human Approval`.
   - Set **Resume** parameter to `Webhook`.
   - Connect `Return Approval Request` main output to this node.
7. **Add Decision Validation and Branching:**
   - Add an **If** node named `Validate Approval`.
   - Set condition logic using type validation (loose) with a custom expression checking data types: `={{ typeof $json.body.approved === 'boolean' && typeof $json.body.approver === 'string' && $json.body.approver.trim() !== '' }}`
   - Connect `Wait for Human Approval` main output to `Validate Approval`.
8. **Configure Error Handling and Success Recording:**
   - Add a **Stop and Error** node named `Reject Approval`. Set the error message to `approved must be a JSON boolean and approver is required for this demo`. Connect the **false** output of `Validate Approval` here.
   - Add a **Set (Edit Fields)** node named `Record Approval`. Configure assignments for properties: `requestId`, `proposal`, `submittedApprover`, `approved`, `decision`, `recordedAt`, and `paymentProcessed` referencing appropriate upstream expressions and `$now.toISO()`. Connect the **true** output of `Validate Approval` here.
9. **Add Visual Documentation:**
   - Add a **Sticky Note** node containing the comprehensive template overview, instructions, and setup steps aligned beside the canvas layout.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Exercise 5 from the n8n Production AI Playbook on Audit & Trace | Reference context for architectural patterns involving OpenTelemetry tracing across asynchronous human-in-the-loop wait states. |
| No payment processing disclaimer | Demonstrative workflow setup; no real financial transactions occur. Reviewer fields and identities should be securely mapped via authenticated logins in production environments. |