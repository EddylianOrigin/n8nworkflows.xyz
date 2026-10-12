Route omnichannel customer support with Ollama and Postgres pgvector

https://n8nworkflows.xyz/workflows/route-omnichannel-customer-support-with-ollama-and-postgres-pgvector-17661


# Route omnichannel customer support with Ollama and Postgres pgvector

### 1. Workflow Overview

This workflow implements an omnichannel customer support agent powered by local AI inference (Ollama) and vector-based semantic search (PostgreSQL with pgvector). Its primary purpose is to ingest incoming customer messages from multiple channels (chat, email, and social media), maintain persistent conversation history, retrieve contextually relevant knowledge base articles, classify user intent, apply safety guardrails, and either automatically respond to the user or escalate the conversation to a human support queue.

The architecture is divided into five logical functional blocks:
- **1.1 Ingress & Validation:** Receives incoming HTTP requests across three distinct webhook entry points, validates shared-secret security headers, and normalizes payloads into a standard unified format.
- **1.2 Memory Persistence & De-duplication:** Upserts conversations in PostgreSQL, records inbound messages, fetches recent conversation history, and filters out duplicate provider event IDs.
- **1.3 Knowledge Retrieval & Augmentation:** Generates vector embeddings for incoming queries via Ollama and queries PostgreSQL (`pgvector`) to find top-matching published knowledge chunks.
- **1.4 AI Reasoning & Guardrails:** Sends user messages, history, and retrieved knowledge to Ollama for intent classification and response drafting, then enforces strict safety and citation rules via code-based guardrails.
- **1.5 Handoff & Delivery Routing:** Branches conditionally based on the guardrail decision; either creates a human support handoff record and alerts the human queue, or records the AI-generated reply and sends it back through the outbound channel adapter.

---

### 2. Block-by-Block Analysis

#### 2.1 Ingress & Validation
- **Overview:** Captures incoming customer interactions from chat, email, or social media webhooks, authenticates them using a shared secret, and normalizes disparate payloads into a uniform internal schema.
- **Nodes Involved:** `Chat inbound`, `Email inbound`, `Social inbound`, `Normalize and validate`, `Invalid request?`, `Reject inbound`
- **Node Details:**
  - **Chat inbound**
    - *Type & Role:* `n8n-nodes-base.webhook` — Acts as the HTTP entry point for live website chat messages.
    - *Configuration:* HTTP method POST, path `customer-service/chat`, response mode set to response node.
    - *Connections:* Input: None (Trigger); Output: `Normalize and validate` (main).
    - *Edge Cases:* Unhandled HTTP methods result in 404/405 errors from n8n.
  - **Email inbound**
    - *Type & Role:* `n8n-nodes-base.webhook` — Acts as the HTTP entry point for customer service emails.
    - *Configuration:* HTTP method POST, path `customer-service/email`, response mode set to response node.
    - *Connections:* Input: None (Trigger); Output: `Normalize and validate` (main).
  - **Social inbound**
    - *Type & Role:* `n8n-nodes-base.webhook` — Acts as the HTTP entry point for social media DMs.
    - *Configuration:* HTTP method POST, path `customer-service/social`, response mode set to response node.
    - *Connections:* Input: None (Trigger); Output: `Normalize and validate` (main).
  - **Normalize and validate**
    - *Type & Role:* `n8n-nodes-base.code` — Validates the `x-cs-secret` request header against environment variables and normalizes message bodies, event IDs, and metadata.
    - *Configuration:* Custom JavaScript snippet evaluating `CS_TENANT_SECRET`, trimming text inputs, checking maximum length constraints (12,000 characters), and detecting the originating communication channel.
    - *Key Expressions:* `={{$env.CS_TENANT_SECRET}}`, `={{$json.body ?? $json}}`
    - *Connections:* Input: `Chat inbound`, `Email inbound`, `Social inbound`; Output: `Invalid request?` (main).
    - *Edge Cases:* Missing required fields (`event_id`, `customer_id`, `message`) or failed secret validation marks the payload with `stop: true` and sets error status codes (401 or 400).
  - **Invalid request?**
    - *Type & Role:* `n8n-nodes-base.if` — Evaluates whether the validation step flagged the request as unauthorized or malformed.
    - *Configuration:* Checks if `{{$json.stop}}` evaluates to `true`.
    - *Connections:* Input: `Normalize and validate`; Output: `Reject inbound` (true), `Persist and fetch memory` (false).
  - **Reject inbound**
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` — Returns an immediate HTTP error response to the client.
    - *Configuration:* Dynamic response code based on input (`{{$json.status}}`), responding with JSON body `{ok: false, error: ...}`.
    - *Connections:* Input: `Invalid request?`; Output: None.

#### 2.2 Memory Persistence & De-duplication
- **Overview:** Interacts with PostgreSQL to upsert customer conversation metadata, store inbound message payloads, retrieve recent context history, and prevent processing duplicate webhook events.
- **Nodes Involved:** `Persist and fetch memory`, `Duplicate event?`, `Acknowledge duplicate`
- **Node Details:**
  - **Persist and fetch memory**
    - *Type & Role:* `n8n-nodes-base.postgres` — Executes a complex SQL CTE query to upsert into `cs_conversations`, insert into `cs_messages`, and aggregate the last 12 messages of conversation history.
    - *Configuration:* Operation: Execute Query. Uses query replacement parameters to safely bind tenant ID, customer ID, channel, event ID, message body, and metadata.
    - *Connections:* Input: `Invalid request?`; Output: `Duplicate event?` (main).
    - *Edge Cases:* Database connection timeouts or missing table schemas will cause workflow execution failures.
  - **Duplicate event?**
    - *Type & Role:* `n8n-nodes-base.if` — Determines if the inbound provider event ID has already been recorded.
    - *Configuration:* Checks if `{{$json.inserted_event}}` is empty (string operation).
    - *Connections:* Input: `Persist and fetch memory`; Output: `Acknowledge duplicate` (true), `Embed question` (false).
  - **Acknowledge duplicate**
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` — Short-circuits duplicate event processing by returning a success status without invoking the AI model.
    - *Configuration:* Response code 200, responding with JSON `{ok: true, duplicate: true}`.
    - *Connections:* Input: `Duplicate event?`; Output: None.

#### 2.3 Knowledge Retrieval & Augmentation
- **Overview:** Converts incoming customer questions into vector embeddings via Ollama and queries PostgreSQL (`pgvector`) to pull the top 5 most relevant and currently valid knowledge base chunks.
- **Nodes Involved:** `Embed question`, `Retrieve approved knowledge`
- **Node Details:**
  - **Embed question**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Calls the Ollama embeddings API to vectorize the customer's message.
    - *Configuration:* POST request to `{{ $env.CS_OLLAMA_URL + '/api/embeddings' }}`, timeout 30,000ms, raw JSON body containing model name (`nomic-embed-text` default) and prompt text.
    - *Connections:* Input: `Duplicate event?`; Output: `Retrieve approved knowledge` (main).
    - *Edge Cases:* Ollama server unavailability or model missing errors will trigger HTTP request failures.
  - **Retrieve approved knowledge**
    - *Type & Role:* `n8n-nodes-base.postgres` — Executes a cosine distance vector similarity search against the `cs_kb_chunks` table.
    - *Configuration:* Operation: Execute Query. Retrieves published, non-expired knowledge snippets matching the generated embedding vector, limited to 5 results.
    - *Connections:* Input: `Embed question`; Output: `Classify and draft` (main).

#### 2.4 AI Reasoning & Guardrails
- **Overview:** Sends the customer message, conversation history, and retrieved knowledge context to Ollama for structured intent classification and response drafting, then evaluates safety guardrails.
- **Nodes Involved:** `Classify and draft`, `Route and guardrail`
- **Node Details:**
  - **Classify and draft**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Calls the Ollama chat completion API requesting structured JSON output.
    - *Configuration:* POST request to `{{ $env.CS_OLLAMA_URL + '/api/chat' }}`, timeout 60,000ms, forces JSON formatting (`format: 'json'`), utilizing system instructions that enforce citation requirements and escalation triggers.
    - *Connections:* Input: `Retrieve approved knowledge`; Output: `Route and guardrail` (main).
    - *Edge Cases:* Hallucinated outputs or invalid JSON payloads are handled defensively in the subsequent code node.
  - **Route and guardrail**
    - *Type & Role:* `n8n-nodes-base.code` — Validates and sanitizes the LLM's JSON response, enforces deterministic guardrails (forcing human escalation for sensitive topics, low confidence, or unverified FAQ citations).
    - *Configuration:* Custom JavaScript block parsing model responses, scanning text for regex patterns matching sensitive financial/legal keywords (e.g., password, credit card, refund, cancellation, legal threats), and checking citation validity against retrieved knowledge sources.
    - *Connections:* Input: `Classify and draft`; Output: `Human handoff?` (main).

#### 2.5 Handoff & Delivery Routing
- **Overview:** Directs the conversation down either a human support escalation path or an automated resolution path, recording actions in the database and dispatching payloads to external webhooks.
- **Nodes Involved:** `Human handoff?`, `Create handoff`, `Notify human queue`, `Record outgoing reply`, `Send through channel adapter`, `Acknowledge provider`
- **Node Details:**
  - **Human handoff?**
    - *Type & Role:* `n8n-nodes-base.if` — Evaluates the guardrail decision property `needs_human`.
    - *Configuration:* Checks if `{{$json.decision.needs_human}}` equals `true`.
    - *Connections:* Input: `Route and guardrail`; Output: `Create handoff` (true), `Record outgoing reply` (false).
  - **Create handoff**
    - *Type & Role:* `n8n-nodes-base.postgres` — Records an escalation ticket in the `cs_handoffs` table and updates the conversation status to `waiting_human`.
    - *Configuration:* Operation: Execute Query. Dynamically sets priority to `urgent` if keywords match legal, safety, or payment terms.
    - *Connections:* Input: `Human handoff?`; Output: `Notify human queue` (main).
  - **Notify human queue**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Dispatches an HTTP POST webhook alerting the human support queue system.
    - *Configuration:* URL bound to `{{$env.CS_HUMAN_WEBHOOK}}`, timeout 10,000ms, with `continueOnFail: true`.
    - *Connections:* Input: `Create handoff`; Output: `Record outgoing reply` (main).
  - **Record outgoing reply**
    - *Type & Role:* `n8n-nodes-base.postgres` — Inserts the AI-generated outbound response message into `cs_messages`.
    - *Configuration:* Operation: Execute Query. Records direction as `outbound` with metadata containing intent and routing decisions.
    - *Connections:* Input: `Human handoff?` (via false branch) or `Notify human queue`; Output: `Send through channel adapter` (main).
  - **Send through channel adapter**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Delivers the finalized response payload back to the customer communication channel adapter.
    - *Configuration:* POST request to `{{$env.CS_REPLY_WEBHOOK}}`, timeout 15,000ms, with `continueOnFail: true`.
    - *Connections:* Input: `Record outgoing reply`; Output: `Acknowledge provider` (main).
  - **Acknowledge provider**
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` — Responds to the original inbound webhook caller confirming successful processing and returning conversation metadata.
    - *Configuration:* Response code 200, responding with JSON summary including conversation ID, intent, and human status.
    - *Connections:* Input: `Send through channel adapter`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Chat inbound | `n8n-nodes-base.webhook` | Inbound webhook trigger for chat | None | Normalize and validate | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Email inbound | `n8n-nodes-base.webhook` | Inbound webhook trigger for email | None | Normalize and validate | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Social inbound | `n8n-nodes-base.webhook` | Inbound webhook trigger for social DMs | None | Normalize and validate | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Normalize and validate | `n8n-nodes-base.code` | Validates shared secret and normalizes payloads | Chat inbound, Email inbound, Social inbound | Invalid request? | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Invalid request? | `n8n-nodes-base.if` | Checks if request validation failed | Normalize and validate | Reject inbound, Persist and fetch memory | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Reject inbound | `n8n-nodes-base.respondToWebhook` | Returns HTTP error for invalid requests | Invalid request? | None | Inbound setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note3 |
| Persist and fetch memory | `n8n-nodes-base.postgres` | Upserts conversation and fetches history | Invalid request? | Duplicate event? | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Duplicate event? | `n8n-nodes-base.if` | Filters out duplicate provider events | Persist and fetch memory | Acknowledge duplicate, Embed question | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Acknowledge duplicate | `n8n-nodes-base.respondToWebhook` | Returns 200 OK for duplicate events | Duplicate event? | None | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Embed question | `n8n-nodes-base.httpRequest` | Generates vector embedding via Ollama | Duplicate event? | Retrieve approved knowledge | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Retrieve approved knowledge | `n8n-nodes-base.postgres` | Queries pgvector for knowledge chunks | Embed question | Classify and draft | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Classify and draft | `n8n-nodes-base.httpRequest` | Calls Ollama chat for intent and drafting | Retrieve approved knowledge | Route and guardrail | Knowledge setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note4 |
| Route and guardrail | `n8n-nodes-base.code` | Applies safety rules and validates citations | Classify and draft | Human handoff? | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Human handoff? | `n8n-nodes-base.if` | Branches based on human escalation need | Route and guardrail | Create handoff, Record outgoing reply | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Create handoff | `n8n-nodes-base.postgres` | Creates escalation ticket in database | Human handoff? | Notify human queue | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Notify human queue | `n8n-nodes-base.httpRequest` | Alerts external human support webhook | Create handoff | Record outgoing reply | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Record outgoing reply | `n8n-nodes-base.postgres` | Saves outbound reply in database | Human handoff?, Notify human queue | Send through channel adapter | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Send through channel adapter | `n8n-nodes-base.httpRequest` | Sends reply to outbound channel webhook | Record outgoing reply | Acknowledge provider | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Acknowledge provider | `n8n-nodes-base.respondToWebhook` | Returns success response to webhook caller | Send through channel adapter | None | Output setup note<br>AI Customer Support Across Chat, Email & Social Media<br>Sticky Note5 |
| Configure template variables | `n8n-nodes-base.set` | Assigns template configuration defaults | None | None | Configuration<br>Requirements |
| About this template | `n8n-nodes-base.stickyNote` | High-level architectural overview note | None | None | About this template |
| Inbound setup note | `n8n-nodes-base.stickyNote` | Inbound configuration instructions | None | None | Inbound setup note |
| Knowledge setup note | `n8n-nodes-base.stickyNote` | Database and pgvector configuration guide | None | None | Knowledge setup note |
| Output setup note | `n8n-nodes-base.stickyNote` | Human handoff and delivery instructions | None | None | Output setup note |
| Sticky Note | `n8n-nodes-base.stickyNote` | Comprehensive template introduction | None | None | AI Customer Support Across Chat, Email & Social Media |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Required environment variables reference | None | None | Configuration |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Local AI model configuration notes | None | None | AI Model |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual container grouping for inbound nodes | None | None | Inbound setup note |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Visual container grouping for knowledge/AI nodes | None | None | Knowledge setup note |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Visual container grouping for output/handoff nodes | None | None | Output setup note |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | System requirements and setup checklist | None | None | Requirements |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Database persistence overview | None | None | Database |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Inbound Webhook Triggers:**
   - Create three **Webhook** nodes named `Chat inbound`, `Email inbound`, and `Social inbound`.
   - Configure each with HTTP Method `POST`, Response Mode set to `Response Node`, and paths `customer-service/chate`, `customer-service/email`, and `customer-service/social` respectively.

2. **Add Normalization & Validation Logic:**
   - Create a **Code** node named `Normalize and validate`.
   - Connect all three webhook nodes to this node.
   - Insert the validation JavaScript snippet checking `x-cs-secret` against `CS_TENANT_SECRET`, extracting channel types, and verifying payload constraints.

3. **Configure Request Guarding:**
   - Create an **If** node named `Invalid request?`. Connect `Normalize and validate` to it.
   - Set the condition to check if `{{$json.stop}}` equals `true`.
   - Create a **Respond to Webhook** node named `Reject inbound`, connected to the `true` branch of `Invalid request?`, configured with dynamic response codes (`{{$json.status}}`) and JSON error bodies.

4. **Setup Database Persistence & De-duplication:**
   - Create a **PostgreSQL** node named `Persist and fetch memory`, connected to the `false` branch of `Invalid request?`.
   - Configure credentials (PostgreSQL) and set operation to *Execute Query*, inserting the CTE query that manages conversations, messages, and history aggregation with query replacement parameter bindings.
   - Create an **If** node named `Duplicate event?`, connected to `Persist and fetch memory`. Check if `{{$json.inserted_event}}` is empty.
   - Create a **Respond to Webhook** node named `Acknowledge duplicate`, connected to the `true` branch of `Duplicate event?`, returning response code 200 with `{ok: true, duplicate: true}`.

5. **Configure Vector Embeddings & Knowledge Retrieval:**
   - Create an **HTTP Request** node named `Embed question`, connected to the `false` branch of `Duplicate event?`.
   - Configure POST request to `{{ $env.CS_OLLAMA_URL + '/api/embeddings' }}` with raw JSON body passing the embedding model (`nomic-embed-text`) and user message.
   - Create a **PostgreSQL** node named `Retrieve approved knowledge`, connected to `Embed question`.
   - Set operation to *Execute Query* using the cosine distance vector similarity query against `cs_kb_chunks`.

6. **Setup AI Reasoning & Guardrails:**
   - Create an **HTTP Request** node named `Classify and draft`, connected to `Retrieve approved knowledge`.
   - Configure POST request to `{{ $env.CS_OLLAMA_URL + '/api/chat' }}` with timeout 60,000ms, forcing JSON output and supplying system/user prompts incorporating retrieved knowledge and conversation history.
   - Create a **Code** node named `Route and guardrail`, connected to `Classify and draft`.
   - Insert the JavaScript guardrail logic that sanitizes JSON parsing, checks for sensitive keyword regexes, verifies citation sources against approved knowledge, and determines `needs_human` status.

7. **Configure Handoff & Delivery Branching:**
   - Create an **If** node named `Human handoff?`, connected to `Route and guardrail`. Set condition to check if `{{$json.decision.needs_human}}` is true.
   - **Escalation Path (True Branch):**
     - Create a **PostgreSQL** node named `Create handoff`, inserting into `cs_handoffs` and updating conversation status to `waiting_human`.
     - Create an **HTTP Request** node named `Notify human queue`, connected to `Create handoff`, pointing to `{{$env.CS_HUMAN_WEBHOOK}}` with `continueOnFail: true`.
   - **Automated Path (False Branch & Convergence):**
     - Create a **PostgreSQL** node named `Record outgoing reply`, connected both to the `false` branch of `Human handoff?` and the output of `Notify human queue`. Inserts the outbound message into `cs_messages`.
     - Create an **HTTP Request** node named `Send through channel adapter`, connected to `Record outgoing reply`, posting to `{{$env.CS_REPLY_WEBHOOK}}` with `continueOnFail: true`.
     - Create a **Respond to Webhook** node named `Acknowledge provider`, connected to `Send through channel adapter`, returning HTTP 200 with conversation metadata.

8. **Add Configuration Metadata Node:**
   - Create a **Set (Edit Fields)** node named `Configure template variables` (unconnected in the graph layout) to document required environment variables (`CS_OLLAMA_URL`, `CS_OLLAMA_MODEL`, `CS_OLLAMA_EMBED_MODEL`, `CS_TENANT_SECRET`, `CS_HUMAN_WEBHOOK`, `CS_REPLY_WEBHOOK`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostgreSQL Database Setup | Requires PostgreSQL with the `pgvector` extension enabled and tables `cs_conversations`, `cs_messages`, `cs_kb_chunks`, and `cs_handoffs` created. |
| Ollama Local Inference | Requires a running Ollama server instance accessible via HTTP, serving `/api/chat` and `/api/embeddings` with models `llama3.1:8b` and `nomic-embed-text` installed. |
| Security & Credentials | Provider authentication and signature verification must be handled in upstream channel adapters before requests reach the n8n webhook entry points. |