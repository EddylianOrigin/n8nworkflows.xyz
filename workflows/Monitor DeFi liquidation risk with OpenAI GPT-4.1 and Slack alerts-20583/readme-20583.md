Monitor DeFi liquidation risk with OpenAI GPT-4.1 and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-defi-liquidation-risk-with-openai-gpt-4-1-and-slack-alerts-20583


# Monitor DeFi liquidation risk with OpenAI GPT-4.1 and Slack alerts

### 1. Workflow Overview

This workflow automates the regular monitoring of DeFi lending positions to detect liquidation risks. Running every 10 minutes, it fetches position data and live market prices via HTTP, calculates critical risk metrics (such as health factor, estimated liquidation price, and distance to liquidation), and evaluates whether the risk status has escalated. Every inspection cycle is recorded in a Google Sheets audit log. When risk thresholds are breached or escalate, the system utilizes OpenAI GPT-4.1 to generate a summarized warning, posts an alert to a dedicated Slack channel, and logs the alert details into a secondary tracking sheet.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule and Configuration Intake:** Triggers the evaluation cycle periodically and injects base runtime settings (API endpoints and risk thresholds).
- **1.2 Lending Position Retrieval & Validation:** Fetches current user position data from a target lending protocol and verifies its numerical integrity.
- **1.3 Market Price Retrieval & Validation:** Fetches live asset pricing data and confirms valid numerical returns.
- **1.4 Risk Metrics Calculation & Classification:** Computes core financial indicators, categorizes the risk level, and checks if an alert state requires distribution.
- **1.5 Monitoring History Logging & Alert Gate:** Persists all operational metrics into a main spreadsheet audit log and conditionally routes execution based on alert flags.
- **1.6 AI Alert Generation & Normalization:** Calls OpenAI GPT-4.1 to synthesize risk metrics into a human-readable summary and parses the response into uniform data structures.
- **1.7 Alert Delivery & Audit:** Broadcasts the notification to Slack and records the final alert status in a separate audit ledger.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule and Configuration Intake
- **Overview:** This block initializes the workflow execution cadence and sets up structural runtime variables, including target API endpoints and predefined health factor sensitivity bounds.
- **Nodes Involved:** 
  - `Schedule Monitoring Cycle`
  - `Load Monitor Configuration`
- **Node Details:**
  - **Schedule Monitoring Cycle**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger node). Executes the workflow at a fixed time interval.
    - *Configuration:* Interval configured to trigger every 10 minutes.
    - *Expressions:* None.
    - *Connections:* Input: None; Output: `Load Monitor Configuration`.
    - *Edge Cases / Failures:* Missed triggers due to system downtime or n8n instance sleep states.
  - **Load Monitor Configuration**
    - *Type & Role:* `n8n-nodes-base.set` (Data transformation/setting node). Defines variables used downstream.
    - *Configuration:* Sets position and price API URLs, along with warning (`1.5`), critical (`1.2`), and urgent (`1.1`) numerical thresholds.
    - *Key Expressions:* Static string and number assignments.
    - *Connections:* Input: `Schedule Monitoring Cycle`; Output: `Fetch Lending Position`.
    - *Edge Cases / Failures:* Typographical errors in URI definitions.

#### 1.2 Lending Position Retrieval & Validation
- **Overview:** Fetches the active lending position from a protocol endpoint and checks that returned collateral, debt, and liquidation threshold fields contain positive values.
- **Nodes Involved:**
  - `Fetch Lending Position`
  - `Validate Lending Position`
- **Node Details:**
  - **Fetch Lending Position**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API integration node). Retrieves position data.
    - *Configuration:* GET request to `{{ $json.position_api_url }}` with a 10,000ms timeout limit.
    - *Key Expressions:* URL: `={{ $json.position_api_url }}`.
    - *Connections:* Input: `Load Monitor Configuration`; Output: `Validate Lending Position`.
    - *Edge Cases / Failures:* HTTP 5xx/4xx errors, connection timeouts, or malformed JSON payloads from the API.
  - **Validate Lending Position**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional routing node). Filters out incomplete or invalid positions.
    - *Configuration:* Evaluates if `collateral_amount`, `debt_amount`, and `liquidation_threshold` are all strictly greater than `0`.
    - *Key Expressions:* `={{ Number($json.collateral_amount) > 0 }}` (and corresponding checks for debt and threshold).
    - *Connections:* Input: `Fetch Lending Position`; Output (True): `Fetch Live Asset Price`.
    - *Edge Cases / Failures:* Type casting failures if strings cannot be parsed cleanly into numbers.

#### 1.3 Market Price Retrieval & Validation
- **Overview:** Obtains the current USD market value of the collateral asset and verifies that a positive price is returned.
- **Nodes Involved:**
  - `Fetch Live Asset Price`
  - `Validate Market Price`
- **Node Details:**
  - **Fetch Live Asset Price**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API integration node). Fetches live market asset quotes.
    - *Configuration:* GET request to `{{ $('Load Monitor Configuration').item.json.price_api_url }}` with a 10,000ms timeout.
    - *Key Expressions:* URL: `={{ $('Load Monitor Configuration').item.json.price_api_url }}`.
    - *Connections:* Input: `Validate Lending Position`; Output: `Validate Market Price`.
    - *Edge Cases / Failures:* Rate limiting, pricing oracle downtime, or network latency.
  - **Validate Market Price**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional routing node). Ensures the fetched price is a valid, non-zero number.
    - *Configuration:* Evaluates if `price_usd` > `0`.
    - *Key Expressions:* `={{ Number($json.price_usd) > 0 }}`.
    - *Connections:* Input: `Fetch Live Asset Price`; Output (True): `Calculate Risk Metrics`.
    - *Edge Cases / Failures:* Missing pricing attributes in payload leading to evaluation errors.

#### 1.4 Risk Metrics Calculation & Classification
- **Overview:** Combines position values and market prices to compute core risk equations (health factor, liquidation price, distance percent), maps the risk level against state variables, and determines if an alert is warranted.
- **Nodes Involved:**
  - `Calculate Risk Metrics`
  - `Classify Liquidation Risk`
- **Node Details:**
  - **Calculate Risk Metrics**
    - *Type & Role:* `n8n-nodes-base.code` (Custom JavaScript execution node). Computes financial derivations.
    - *Configuration:* Executes custom JavaScript processing inputs from both upstream HTTP nodes, outputting normalized metrics rounded to standard decimal places.
    - *Key Expressions:* Custom JS parsing `collateral_amount`, `debt_amount`, and `price_usd`.
    - *Connections:* Input: `Validate Market Price`; Output: `Classify Liquidation Risk`.
    - *Edge Cases / Failures:* Division by zero if debt values compute unexpectedly; runtime errors on missing keys.
  - **Classify Liquidation Risk**
    - *Type & Role:* `n8n-nodes-base.code` (Custom JavaScript execution node). Categorizes the risk level and tracks historical states.
    - *Configuration:* Assigns risk tiers (`SAFE`, `WARNING`, `CRITICAL`, `URGENT`, `LIQUIDATION_BREACHED`) based on health factor bands, updating workflow static state memory (`global`).
    - *Key Expressions:* Uses `$getWorkflowStaticData('global')` to track state changes and severity escalations.
    - *Connections:* Input: `Calculate Risk Metrics`; Output: `Log Monitoring Result`.
    - *Edge Cases / Failures:* State persistence synchronization anomalies across heavy workloads.

#### 1.5 Monitoring History Logging & Alert Gate
- **Overview:** Writes the comprehensive monitoring dataset to a Google Sheets audit log and gates alert generation to only trigger when the risk state changes or escalates.
- **Nodes Involved:**
  - `Log Monitoring Result`
  - `Check Alert Required`
- **Node Details:**
  - **Log Monitoring Result**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Spreadsheet logging node). Appends audit entries.
    - *Configuration:* Append operation targeting spreadsheet `19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`, sheet `Monitoring History`. Auto-maps input data.
    - *Key Expressions:* Column values explicitly map `timestamp`.
    - *Connections:* Input: `Classify Liquidation Risk`; Output: `Check Alert Required`.
    - *Edge Cases / Failures:* Google API authentication expiration, quota exhaustion, or mismatched column schemas.
  - **Check Alert Required**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional routing node). Prevents alert spamming.
    - *Configuration:* Evaluates whether `alert_required` equals `true`.
    - *Key Expressions:* `={{ $json.alert_required }}` == `true`.
    - *Connections:* Input: `Log Monitoring Result`; Output (True): `Generate Risk Alert Message`.
    - *Edge Cases / Failures:* Boolean type mismatches.

#### 1.6 AI Alert Generation & Normalization
- **Overview:** Sends the risk data payload to OpenAI GPT-4.1 to generate an intelligent summary message, which is subsequently parsed into structured metadata.
- **Nodes Involved:**
  - `Generate Risk Alert Message`
  - `Normalize AI Risk Alert`
- **Node Details:**
  - **Generate Risk Alert Message**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI language model node). Generates text content.
    - *Configuration:* Uses model `gpt-4.1-nano` with a temperature setting of `0.2`. Prompt instructs the model to summarize risk factors using *only* supplied JSON parameters without providing financial advice.
    - *Credentials:* OpenAI (PAID) (`openAiApi`).
    - *Key Expressions:* Prompt injects `{{ JSON.stringify($json) }}`.
    - *Connections:* Input: `Check Alert Required`; Output: `Normalize AI Risk Alert`.
    - *Edge Cases / Failures:* API token limits, model unavailability, or rate limiting (HTTP 429).
  - **Normalize AI Risk Alert**
    - *Type & Role:* `n8n-nodes-base.code` (Custom JavaScript execution node). Normalizes AI response outputs.
    - *Configuration:* Regex-based parsing script extracting specific line metrics (collateral details, price levels, risk classification, summary notes) from raw LLM text blocks.
    - *Key Expressions:* Custom JS string matching against AI message content.
    - *Connections:* Input: `Generate Risk Alert Message`; Output: Branching to `Send Liquidation Alert` and `Log Alert History`.
    - *Edge Cases / Failures:* LLM semantic drift causing regex pattern matching failures.

#### 1.7 Alert Delivery & Audit
- **Overview:** Dispatches the final alert message to a designated Slack channel and records the notification event in a secondary Google Sheets audit table.
- **Nodes Involved:**
  - `Send Liquidation Alert`
  - `Log Alert History`
- **Node Details:**
  - **Send Liquidation Alert**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging integration node). Sends team notifications.
    - *Configuration:* Sends message to Slack channel `C0B1LNY15GW` using structured markdown incorporating data references from upstream classification and AI nodes.
    - *Key Expressions:* Dynamic text templates pulling position identifiers, health factors, liquidation prices, and generated AI summaries.
    - *Connections:* Input: `Normalize AI Risk Alert`; Output: None (Terminal node).
    - *Edge Cases / Failures:* Slack API token revocation, invalid channel IDs, or missing bot scopes (`chat:write`).
  - **Log Alert History**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Spreadsheet logging node). Records alert audits.
    - *Configuration:* Append operation targeting spreadsheet `19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`, sheet `Alert History`.
    - *Key Expressions:* Auto-maps input data from normalized AI schema.
    - *Connections:* Input: `Normalize AI Risk Alert`; Output: None (Terminal node).
    - *Edge Cases / Failures:* Schema mismatches between normalized payload and Google Sheet headers.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Monitoring Cycle | `n8n-nodes-base.scheduleTrigger` | Triggers workflow every 10 minutes | None | Load Monitor Configuration | Schedule and Configuration Intake<br>Workflow Documentation |
| Load Monitor Configuration | `n8n-nodes-base.set` | Sets API URLs and health factor thresholds | Schedule Monitoring Cycle | Fetch Lending Position | Schedule and Configuration Intake<br>Workflow Documentation |
| Fetch Lending Position | `n8n-nodes-base.httpRequest` | Retrieves DeFi lending position details | Load Monitor Configuration | Validate Lending Position | Lending Position Retrieval<br>Workflow Documentation |
| Validate Lending Position | `n8n-nodes-base.if` | Validates position values > 0 | Fetch Lending Position | Fetch Live Asset Price | Lending Position Retrieval<br>Workflow Documentation |
| Fetch Live Asset Price | `n8n-nodes-base.httpRequest` | Fetches live collateral asset price | Validate Lending Position | Validate Market Price | Market Price Retrieval<br>Workflow Documentation |
| Validate Market Price | `n8n-nodes-base.if` | Validates asset price > 0 | Fetch Live Asset Price | Calculate Risk Metrics | Market Price Retrieval<br>Workflow Documentation |
| Calculate Risk Metrics | `n8n-nodes-base.code` | Calculates health factor and liquidation distance | Validate Market Price | Classify Liquidation Risk | Risk Metrics Calculation<br>Workflow Documentation |
| Classify Liquidation Risk | `n8n-nodes-base.code` | Classifies risk tier and change state | Calculate Risk Metrics | Log Monitoring Result | Risk Metrics Calculation<br>Workflow Documentation |
| Log Monitoring Result | `n8n-nodes-base.googleSheets` | Logs cycle output to Monitoring History sheet | Classify Liquidation Risk | Check Alert Required | Monitoring History and Alert Gate<br>Workflow Documentation |
| Check Alert Required | `n8n-nodes-base.if` | Evaluates if alert is required | Log Monitoring Result | Generate Risk Alert Message | Monitoring History and Alert Gate<br>Workflow Documentation |
| Generate Risk Alert Message | `@n8n/n8n-nodes-langchain.openAi` | Generates AI risk alert summary | Check Alert Required | Normalize AI Risk Alert | AI Alert Generation<br>Workflow Documentation |
| Normalize AI Risk Alert | `n8n-nodes-base.code` | Parses AI output into structured format | Generate Risk Alert Message | Send Liquidation Alert, Log Alert History | AI Alert Generation<br>Workflow Documentation |
| Send Liquidation Alert | `n8n-nodes-base.slack` | Sends notification to Slack channel | Normalize AI Risk Alert | None | Alert Delivery and Audit<br>Workflow Documentation |
| Log Alert History | `n8n-nodes-base.googleSheets` | Logs alert entry to Alert History sheet | Normalize AI Risk Alert | None | Alert Delivery and Audit<br>Workflow Documentation |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Schedule Monitoring Cycle`.
   - Set interval type to **Minutes**, set value to `10`.

2. **Setup Configuration Initialization:**
   - Add a **Set (Edit Fields)** node named `Load Monitor Configuration`.
   - Add string assignments: `position_api_url` (`http://...`), `price_api_url` (`http://...`).
   - Add number assignments: `warning_threshold` (`1.5`), `critical_threshold` (`1.2`), `urgent_threshold` (`1.1`).
   - Connect `Schedule Monitoring Cycle` output to `Load Monitor Configuration`.

3. **Fetch Position Data:**
   - Add an **HTTP Request** node named `Fetch Lending Position`.
   - Set Method to `GET`, URL to `={{ $json.position_api_url }}`, and options timeout to `10000`.
   - Connect `Load Monitor Configuration` to `Fetch Lending Position`.

4. **Validate Position Response:**
   - Add an **If** node named `Validate Lending Position`.
   - Configure three rules: `Number($json.collateral_amount)` > `0`, `Number($json.debt_amount)` > `0`, and `Number($json.liquidation_threshold)` > `0`.
   - Connect `Fetch Lending Position` to `Validate Lending Position`.

5. **Fetch Market Price:**
   - Add an **HTTP Request** node named `Fetch Live Asset Price`.
   - Set Method to `GET`, URL to `={{ $('Load Monitor Configuration').item.json.price_api_url }}`, options timeout to `10000`.
   - Connect the **True** branch of `Validate Lending Position` to `Fetch Live Asset Price`.

6. **Validate Market Price:**
   - Add an **If** node named `Validate Market Price`.
   - Set rule: `Number($json.price_usd)` > `0`.
   - Connect `Fetch Live Asset Price` to `Validate Market Price`.

7. **Calculate Risk Metrics:**
   - Add a **Code** node named `Calculate Risk Metrics`.
   - Insert JavaScript to pull items from `Fetch Lending Position` and `Fetch Live Asset Price`, compute collateral value, health factor, liquidation price, and distance metrics. Return array with JSON payload.
   - Connect the **True** branch of `Validate Market Price` to `Calculate Risk Metrics`.

8. **Classify Risk States:**
   - Add a **Code** node named `Classify Liquidation Risk`.
   - Insert JavaScript to evaluate health factor thresholds (`SAFE`, `WARNING`, `CRITICAL`, `URGENT`, `LIQUIDATION_BREACHED`), compare with workflow static global data (`$getWorkflowStaticData('global')`), and output flags for `risk_state_changed`, `risk_escalated`, and `alert_required`.
   - Connect `Calculate Risk Metrics` to `Classify Liquidation Risk`.

9. **Log Monitoring Audit Data:**
   - Add a **Google Sheets** node named `Log Monitoring Result`.
   - Configure credentials, operation to `Append`, document ID, and select the `Monitoring History` worksheet. Map auto-input data.
   - Connect `Classify Liquidation Risk` to `Log Monitoring Result`.

10. **Gate Alert Trigger:**
    - Add an **If** node named `Check Alert Required`.
    - Set condition: `={{ $json.alert_required }}` equals boolean `true`.
    - Connect `Log Monitoring Result` to `Check Alert Required`.

11. **Generate AI Risk Summary:**
    - Add an **OpenAI** node named `Generate Risk Alert Message`.
    - Select model `gpt-4.1-nano` (or equivalent), set temperature to `0.2`.
    - Set prompt message content to parse stringified JSON (`{{ JSON.stringify($json) }}`). Configure OpenAI API credentials.
    - Connect the **True** branch of `Check Alert Required` to `Generate Risk Alert Message`.

12. **Normalize AI Output:**
    - Add a **Code** node named `Normalize AI Risk Alert`.
    - Insert JavaScript containing regular expression extraction patterns to normalize protocol fields, health factors, risk levels, and summary texts.
    - Connect `Generate Risk Alert Message` to `Normalize AI Risk Alert`.

13. **Dispatch Slack Notification:**
    - Add a **Slack** node named `Send Liquidation Alert`.
    - Configure Slack credentials, set resource to `message`, operation to `post`.
    - Set channel to ID mode and select target channel (`C0B1LNY15GW`). Populate text body with dynamic references to classification nodes and normalized AI text summaries.
    - Connect `Normalize AI Risk Alert` to `Send Liquidation Alert`.

14. **Record Alert Audit History:**
    - Add a **Google Sheets** node named `Log Alert History`.
    - Configure credentials, operation to `Append`, document ID, and select the `Alert History` worksheet. Map auto-input data.
    - Connect `Normalize AI Risk Alert` to `Log Alert History`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n automation workflows development and configuration support | [WeblineIndia n8n Services](https://www.weblineindia.com/n8n-automation/) |