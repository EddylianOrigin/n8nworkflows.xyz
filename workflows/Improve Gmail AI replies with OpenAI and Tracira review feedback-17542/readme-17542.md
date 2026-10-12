Improve Gmail AI replies with OpenAI and Tracira review feedback

https://n8nworkflows.xyz/workflows/improve-gmail-ai-replies-with-openai-and-tracira-review-feedback-17542


# Improve Gmail AI replies with OpenAI and Tracira review feedback

### 1. Workflow Overview

This workflow automates the drafting, human review, and dispatching of Gmail responses using OpenAI combined with Tracira's instruction and feedback tracking system. Its primary target use case is customer support teams aiming to maintain human oversight over AI-generated outbound communications while continuously improving prompt guidelines based on reviewer adjustments.

The workflow is organized into two primary logical blocks:
- **1.1 Input Reception & AI Drafting Block:** Monitors incoming emails, extracts relevant metadata, fetches dynamic system instructions from Tracira, generates an initial AI response via OpenAI, and logs the draft in Tracira for review without automatically sending it.
- **1.2 Human Review & Feedback Loop Block:** Listens for review events from Tracira, filters them by project and task, handles branching logic based on the human reviewer's decision (approved, manually rewritten, or sent back with revision comments), and either dispatches the email via Gmail or triggers an automated AI revision cycle.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & AI Drafting Block
**Overview:** This block captures inbound Gmail messages, filters out outbound replies, normalizes the email structure, fetches current behavioral guidelines from Tracira, generates an AI draft response, and securely routes the draft for human authorization.

**Nodes Involved:**
- `Gmail` (Gmail Trigger)
- `Filter` (Filter)
- `Email` (Set)
- `Get instructions` (Tracira)
- `Draft with the instructions` (OpenAI)
- `Send to Tracira` (Tracira)

**Node Details:**

- **Gmail**
  - *Type & Role:* `n8n-nodes-base.gmailTrigger` — Triggers the workflow execution when new messages arrive.
  - *Configuration:* Polls every minute (`everyMinute`), using default options and raw filters disabled (`simple: false`).
  - *Expressions:* None.
  - *Connections:* Output connects to `Filter`.
  - *Edge Cases:* API rate limits, authentication token expiration, or polling delays.

- **Filter**
  - *Type & Role:* `n8n-nodes-base.filter` — Prevents infinite loops by ignoring messages already marked as sent.
  - *Configuration:* Validates array contents on the `labelIds` property.
  - *Expressions:* `={{ $json.labelIds }}` must not contain `SENT`.
  - *Connections:* Input from `Gmail`; output connects to `Email`.
  - *Edge Cases:* Missing label arrays on incoming message objects.

- **Email**
  - *Type & Role:* `n8n-nodes-base.set` — Normalizes email payload structures into clean, standardized parameters.
  - *Configuration:* Maps properties for payload processing.
  - *Key Expressions:*
    - `sender`: `={{ $json.from?.value?.[0]?.address ?? $json.from?.text ?? $json.from }}`
    - `subject`: `={{ $json.subject }}`
    - `body`: `={{ $json.text }}`
    - `bodyHtml`: `={{ $json.html }}`
    - `threadId`: `={{ $json.threadId }}`
    - `messageId`: `={{ $json.id }}`
  - *Connections:* Input from `Filter`; output connects to `Get instructions`.
  - *Edge Cases:* Non-standard `from` header formats resulting in null senders.

- **Get instructions**
  - *Type & Role:* `@deepidealab/n8n-nodes-tracira.tracira` — Fetches active playbooks and instructions.
  - *Configuration:* Resource set to `instructions`. Project: `Customer Support`, Task: `Email reply and improve - n8n`. Includes starter instructions text for cold bootstrapping.
  - *Expressions:* Static configuration parameters.
  - *Connections:* Input from `Email`; output connects to `Draft with the instructions`.
  - *Edge Cases:* Workspace connectivity drops or missing project/task definitions.

- **Draft with the instructions**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` — Generates contextual response text using an LLM.
  - *Configuration:* Uses model `GPT-5.6-TERRA`. Simplified output disabled.
  - *Key Expressions:*
    - System message: `={{ $json.content }}`
    - User prompt: `=Customer email from {{ $('Email').first().json.sender }} - subject: {{ $('Email').first().json.subject }}\n\n{{ $('Email').first().json.body }}`
  - *Connections:* Input from `Get instructions`; output connects to `Send to Tracira`.
  - *Edge Cases:* OpenAI API rate limits, model hallucinations, or token context length overflows.

- **Send to Tracira**
  - *Type & Role:* `@deepidealab/n8n-nodes-tracira.tracira` — Asynchronously logs the generated draft for human inspection.
  - *Configuration:* Mode set to `async`. Project: `Customer Support`, Task: `Email reply and improve - n8n`.
  - *Key Expressions:*
    - Input: `={{ $('Email').first().json.bodyHtml }}`
    - Output: `={{ $json.choices[0].message.content }}`
    - Actor ID: `={{ $('Email').first().json.sender }}`
    - Metadata subject: `={{ $('Email').first().json.subject }}`
    - Session ID: `={{ $('Email').first().json.threadId }}`
    - Subject ID: `={{ $('Email').first().json.messageId }}`
    - Instructions Version: `={{ $('Get instructions').first().json.version }}`
  - *Connections:* Input from `Draft with the instructions`; terminal node for this branch.
  - *Edge Cases:* Payload serialization issues or mismatched tracking IDs.

---

#### 1.2 Human Review & Feedback Loop Block
**Overview:** This block listens to review outcomes coming from Tracira, routes execution paths depending on whether the human approved, edited, or rejected the draft, and handles automated instruction learning loops.

**Nodes Involved:**
- `Tracira Trigger` (Tracira Trigger)
- `For this task?` (Filter)
- `What did the reviewer do?` (Switch)
- `Send the reply` (Gmail)
- `Get latest instructions` (Tracira)
- `Revise the draft` (OpenAI)
- `Re-check the revised draft` (Tracira)

**Node Details:**

- **Tracira Trigger**
  - *Type & Role:* `@deepidealab/n8n-nodes-tracira.traciraTrigger` — Webhook entry point responding to review actions.
  - *Configuration:* Listens to events: `approved` and `changed`.
  - *Expressions:* None.
  - *Connections:* Output connects to `For this task?`.
  - *Edge Cases:* Webhook signature mismatches or unreachable endpoint configurations.

- **For this task?**
  - *Type & Role:* `n8n-nodes-base.filter` — Isolates events intended for the active workflow project and task.
  - *Configuration:* Combinator set to `and`.
  - *Key Expressions:*
    - Project validation: `={{ $json.projectName }}` equals `Customer Support`
    - Task validation: `={{ $json.taskName }}` equals `Email reply and improve - n8n`
  - *Connections:* Input from `Tracira Trigger`; output connects to `What did the reviewer do?`.
  - *Edge Cases:* Project name drift between Tracira setup and node parameters.

- **What did the reviewer do?**
  - *Type & Role:* `n8n-nodes-base.switch` — Routes execution paths based on human decision metadata.
  - *Configuration:* Evaluates conditional rule matches.
  - *Key Expressions:*
    - Output 0 (Approved): `={{ $json.decision }}` equals `approved`
    - Output 1 (Rewritten): `={{ $json.correctedOutput || '' }}` is not empty
    - Output 2 (Sent back): `={{ $json.comment || '' }}` is not empty
  - *Connections:* Input from `For this task?`; outputs connect to `Send the reply` (outputs 0 & 1) and `Get latest instructions` (output 2).
  - *Edge Cases:* Ambiguous payloads where multiple conditions overlap.

- **Send the reply**
  - *Type & Role:* `n8n-nodes-base.gmail` — Dispatches the finalized email via Gmail reply operation.
  - *Configuration:* Operation set to `reply`. Appends attribution disabled.
  - *Key Expressions:*
    - Message body: `={{ $json.output.replaceAll('\n', '<br/>') }}`
    - Message ID: `={{ $json.subjectId }}`
  - *Connections:* Inputs from `What did the reviewer do?` (Outputs 0 and 1); terminal node.
  - *Edge Cases:* Thread ID mismatches or expired Gmail OAuth tokens.

- **Get latest instructions**
  - *Type & Role:* `@deepidealab/n8n-nodes-tracira.tracira` — Fetches updated instruction sets containing reviewer corrections.
  - *Configuration:* Resource set to `instructions`.
  - *Key Expressions:*
    - Task Name: `={{ $json.taskName }}`
    - Project Name: `={{ $json.projectName }}`
  - *Connections:* Input from `What did the reviewer do?` (Output 2); output connects to `Revise the draft`.
  - *Edge Cases:* Missing dynamic project or task names in event payloads.

- **Revise the draft**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` — Re-processes text using new manager constraints.
  - *Configuration:* Uses model `GPT-5.6-TERRA`. Simplified output disabled.
  - *Key Expressions:*
    - System message: `={{ $('Get latest instructions').first().json.content }}\n\n--\n\nYou revise draft replies...`
    - User prompt: `=Input:\n{{ $('What did the reviewer do?').item.json.input }}\n\nDraft reply:\n{{ $('What did the reviewer do?').item.json.output }}\n\nManager's instruction:\n{{ $('What did the reviewer do?').item.json.comment }}`
  - *Connections:* Input from `Get latest instructions`; output connects to `Re-check the revised draft`.
  - *Edge Cases:* Token limits exceeded during revision injection.

- **Re-check the revised draft**
  - *Type & Role:* `@deepidealab/n8n-nodes-tracira.tracira` — Submits the AI-revised response back to Tracira for re-evaluation.
  - *Configuration:* Mode set to `async`.
  - *Key Expressions:*
    - Input: `={{ $('Tracira Trigger').item.json.input }}`
    - Output: `={{ $json.choices[0].message.content }}`
    - Actor ID: `={{ $('Tracira Trigger').item.json.actorId }}`
    - Metadata subject: `={{ $('Tracira Trigger').item.json.metadata.subject }}`
    - Session ID: `={{ $('Tracira Trigger').item.json.sessionId }}`
    - Subject ID: `={{ $('Tracira Trigger').item.json.subjectId }}`
    - Revision Of: `={{ $('Tracira Trigger').item.json.logId }}`
    - Instructions Version: `={{ $('Get latest instructions').first().json.version }}`
    - Task Name: `={{ $('Tracira Trigger').item.json.taskName }}`
    - Project Name: `={{ $('Tracira Trigger').item.json.projectName }}`
  - *Connections:* Input from `Revise the draft`; terminal node.
  - *Edge Cases:* Missing log reference IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note 0 | n8n-nodes-base.stickyNote | Workflow overview documentation and onboarding instructions. | None | None | ## Self-improving email replies<br><br>Correct your AI once and it never makes that mistake again. **Nothing reaches a customer until you approve it** from any device.<br><br>Two branches, one automation, joined through Tracira (not by a wire). Activate the workflow once and both run.<br><br>### How it works<br>- **Draft branch:** every new Gmail message is flattened into clean fields (sender, subject, body). Get instructions serves the playbook stored in Tracira, the AI drafts with it, and Send to Tracira logs the draft. Nothing is sent yet.<br>- **Decision branch:** the Tracira Trigger fires the moment you decide. Approve and Gmail sends the reply. Rewrite it yourself and Gmail sends your version. Send it back with a note instead and the AI redrafts and returns it for a second look. Either correction teaches the stored instructions, so the next draft already follows it.<br><br>### Setup<br>1. Create a free account at [tracira.com](https://tracira.com/?utm_source=n8n&utm_medium=template&utm_campaign=self-improving-email-replies) if you don't have one yet.<br>2. In n8n, search the nodes panel for **Tracira** and add it.<br>3. Connect your **Gmail**, **OpenAI**, and **Tracira** credentials.<br>4. Open **Get instructions** and **Send to Tracira** and set your workspace and task name.<br>5. Activate the workflow, then approve or edit each draft in Tracira.<br><br>### Customization tips<br>- Edit the starter text in **Get instructions**. After the first run the instructions live in Tracira and you change them by reviewing.<br>- Narrow the **Filter** node to certain senders or labels. |
| Sticky Note A1 | n8n-nodes-base.stickyNote | Explains the email ingestion step. | None | None | ### 1. Receive new email<br><br>**Email** flattens the message into clean fields (sender, subject, body). |
| Sticky Note A | n8n-nodes-base.stickyNote | Explains the AI drafting phase and Tracira logging. | None | None | ### 2. Draft with the stored instructions<br><br>**Get instructions** serves the playbook hosted in Tracira (the first run seeds it).<br>The AI drafts with it, then **Send to Tracira** logs the draft and the version it followed. Nothing is sent yet. |
| Sticky Note B | n8n-nodes-base.stickyNote | Explains the human decision routing and handling. | None | None | ### 3. Act on the human decision<br><br>**Approved** sends the AI's reply. **Rewritten by the reviewer** sends theirs, already fixed.<br>**Sent back to the AI** redrafts it and resubmits it for another look.<br>Either correction teaches the stored instructions, so there is no prompt editing here. |
| Gmail | n8n-nodes-base.gmailTrigger | Triggers execution on new Gmail message arrival. | None | Filter | |
| Filter | n8n-nodes-base.filter | Filters out messages with the SENT label. | Gmail | Email | |
| Email | n8n-nodes-base.set | Flattens raw email data into clean parameters. | Filter | Get instructions | |
| Get instructions | @deepidealab/n8n-nodes-tracira.tracira | Fetches playbook instructions from Tracira. | Email | Draft with the instructions | |
| Draft with the instructions | @n8n/n8n-nodes-langchain.openAi | Generates email draft response using OpenAI. | Get instructions | Send to Tracira | |
| Send to Tracira | @deepidealab/n8n-nodes-tracira.tracira | Logs generated draft for asynchronous review. | Draft with the instructions | None | |
| Tracira Trigger | @deepidealab/n8n-nodes-tracira.traciraTrigger | Listens for Tracira review decisions. | None | For this task? | |
| For this task? | n8n-nodes-base.filter | Validates project and task name matching. | Tracira Trigger | What did the reviewer do? | |
| What did the reviewer do? | n8n-nodes-base.switch | Routes workflow based on reviewer actions. | For this task? | Send the reply, Get latest instructions | |
| Send the reply | n8n-nodes-base.gmail | Sends the approved or rewritten email via Gmail. | What did the reviewer do? | None | |
| Get latest instructions | @deepidealab/n8n-nodes-tracira.tracira | Fetches updated instructions after reviewer edits. | What did the reviewer do? | Revise the draft | |
| Revise the draft | @n8n/n8n-nodes-langchain.openAi | Revises the draft based on reviewer comments. | Get latest instructions | Re-check the revised draft | |
| Re-check the revised draft | @deepidealab/n8n-nodes-tracira.tracira | Submits revised draft back to Tracira. | Revise the draft | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in n8n:

1. **Create Node:** Add a **Gmail Trigger** node (`Gmail`). Set poll times to every minute and disable simplified outputs. Configure Gmail OAuth2 credentials.
2. **Create Node:** Add a **Filter** node (`Filter`). Set the condition to check that `{{ $json.labelIds }}` does not contain `SENT`. Connect `Gmail` output to `Filter`.
3. **Create Node:** Add a **Set** node (`Email`). Configure assignments for `sender`, `subject`, `body`, `bodyHtml`, `threadId`, and `messageId` using expressions matching:
   - `sender`: `={{ $json.from?.value?.[0]?.address ?? $json.from?.text ?? $json.from }}`
   - `subject`: `={{ $json.subject }}`
   - `body`: `={{ $json.text }}`
   - `bodyHtml`: `={{ $json.html }}`
   - `threadId`: `={{ $json.threadId }}`
   - `messageId`: `={{ $json.id }}`
   Connect `Filter` output to `Email`.
4. **Create Node:** Add a **Tracira** node (`Get instructions`). Set resource to `instructions`, project name to `Customer Support`, task name to `Email reply and improve - n8n`, and provide starter instructions in the parameter field. Connect `Email` output to `Get instructions`. Requires Tracira API credentials.
5. **Create Node:** Add an **OpenAI Chat Model** node (`Draft with the instructions`). Configure model selection to `GPT-5.6-TERRA` using OpenAI credentials. Add system message `={{ $json.content }}` and user prompt referencing sender, subject, and body from the `Email` node. Connect `Get instructions` output to `Draft with the instructions`.
6. **Create Node:** Add a **Tracira** node (`Send to Tracira`). Set mode to `async`, project to `Customer Support`, task name to `Email reply and improve - n8n`, input to `={{ $('Email').first().json.bodyHtml }}`, and output to `={{ $json.choices[0].message.content }}`. Map metadata, actor ID, session ID, subject ID, and instruction versions. Connect `Draft with the instructions` output to `Send to Tracira`.
7. **Create Node:** Add a **Tracira Trigger** node (`Tracira Trigger`). Subscribe to events `approved` and `changed`.
8. **Create Node:** Add a **Filter** node (`For this task?`). Configure conditions to verify `{{ $json.projectName }}` equals `Customer Support` and `{{ $json.taskName }}` equals `Email reply and improve - n8n`. Connect `Tracira Trigger` output to `For this task?`.
9. **Create Node:** Add a **Switch** node (`What did the reviewer do?`). Set up three output rules:
   - Rule 1: `={{ $json.decision }}` equals `approved`
   - Rule 2: `={{ $json.correctedOutput || '' }}` is not empty
   - Rule 3: `={{ $json.comment || '' }}` is not empty
   Connect `For this task?` output to `What did the reviewer do?`.
10. **Create Node:** Add a **Gmail** node (`Send the reply`). Set operation to `reply`, message ID to `={{ $json.subjectId }}`, and message body to `={{ $json.output.replaceAll('\n', '<br/>') }}`. Connect output routes 0 and 1 of `What did the reviewer do?` to `Send the reply`.
11. **Create Node:** Add a **Tracira** node (`Get latest instructions`). Set resource to `instructions`, project name to `={{ $json.projectName }}`, and task name to `={{ $json.taskName }}`. Connect output route 2 of `What did the reviewer do?` to `Get latest instructions`.
12. **Create Node:** Add an **OpenAI Chat Model** node (`Revise the draft`). Configure model to `GPT-5.6-TERRA`. Set system prompt combining fetched instructions with revision directives, and user prompt mapping input, draft reply, and manager's comment. Connect `Get latest instructions` output to `Revise the draft`.
13. **Create Node:** Add a **Tracira** node (`Re-check the revised draft`). Set mode to `async`, task name to `={{ $('Tracira Trigger').item.json.taskName }}`, project name to `={{ $('Tracira Trigger').item.json.projectName }}`, input to `={{ $('Tracira Trigger').item.json.input }}`, output to `={{ $json.choices[0].message.content }}`, and map contextual identifiers, revision IDs, and instruction versions. Connect `Revise the draft` output to `Re-check the revised draft`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a free account at Tracira if needed. | [Tracira Website](https://tracira.com/?utm_source=n8n&utm_medium=template&utm_campaign=self-improving-email-replies) |