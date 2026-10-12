Identify and fix low-quality Meta ads with OpenAI and Gmail reports

https://n8nworkflows.xyz/workflows/identify-and-fix-low-quality-meta-ads-with-openai-and-gmail-reports-17610


# Identify and fix low-quality Meta ads with OpenAI and Gmail reports

### 1. Workflow Overview

This workflow automates the detection, analysis, and reporting of underperforming Meta ads based on quality, engagement, and conversion rankings. Its primary purpose is to identify ads that Meta is quietly penalizing due to below-average metrics, generate actionable optimization steps using OpenAI, and deliver a formatted digest via email.

The workflow logic is grouped into the following functional blocks:
- **1.1 Initialization & Configuration:** Handles manual execution triggers and sets runtime configuration variables such as date ranges and minimum spend floors.
- **1.2 Data Retrieval & Filtering:** Queries the Meta Marketing API for ad-level performance insights and filters out ads that do not meet the minimum spend threshold or performance criteria.
- **1.3 AI Processing:** Constructs tailored prompts and invokes OpenAI's GPT-4o-mini model to analyze flagged ads and suggest specific corrective levers.
- **1.4 Reporting & Delivery:** Formats the AI assessment and raw metrics into a readable report and transmits it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Configuration
- **Overview:** Initializes the execution flow manually and defines the operational parameters for the Meta ads analysis.
- **Nodes Involved:** `Run manually`, `Set config: quality digest`
- **Node Details:**
  - **Run manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Version 1). Acts as the primary entry point for manual testing and on-demand execution.
    - *Configuration Choices:* Default configuration.
    - *Input/Output:* No inputs; outputs to `Set config: quality digest`.
    - *Edge Cases:* None.
  - **Set config: quality digest**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2). Sets global configuration parameters (`DATE_PRESET` and `MIN_SPEND`).
    - *Configuration Choices:* JavaScript block returning a JSON object with `DATE_PRESET: 'last_7d'` and `MIN_SPEND: 20`.
    - *Input/Output:* Input from `Run manually`; output to `Fetch ad quality rankings (Meta)`.
    - *Edge Cases:* Short windows are required because Meta returns `UNKNOWN` for quality rankings over a 30-day window.

#### 2.2 Data Retrieval & Filtering
- **Overview:** Fetches ad-level diagnostic metrics from the Meta Marketing API and isolates ads exhibiting below-average performance rankings.
- **Nodes Involved:** `Fetch ad quality rankings (Meta)`, `Flag below-average ads`
- **Node Details:**
  - **Fetch ad quality rankings (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Makes a GET request to the Meta Graph API to retrieve ad insights.
    - *Configuration Choices:* Uses query parameters for `access_token`, `level=ad`, `date_preset` (from the config node), requested `fields` (`ad_name`, `campaign_name`, `adset_name`, `spend`, `impressions`, `quality_ranking`, `engagement_rate_ranking`, `conversion_rate_ranking`), and a limit of 500.
    - *Key Expressions:* `={{ $env.META_API_VERSION || 'v21.0' }}`, `={{ $env.META_AD_ACCOUNT_ID }}`, `={{ $('Set config: quality digest').first().json.DATE_PRESET }}`
    - *Input/Output:* Input from `Set config: quality digest`; output to `Flag below-average ads`.
    - *Edge Cases:* Authentication errors if `META_ACCESS_TOKEN` is invalid or expired; API rate limits if executing frequently.
  - **Flag below-average ads**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2). Filters raw Meta API output against the minimum spend floor and identifies attributes flagged as `BELOW_AVERAGE`.
    - *Configuration Choices:* JavaScript loop evaluating spend and ranking strings, sorting results by spend in descending order.
    - *Input/Output:* Input from `Fetch ad quality rankings (Meta)`; output to `Build AI quality prompt`.
    - *Edge Cases:* Empty data sets returned from Meta will result in a `flagged_count` of 0.

#### 2.3 AI Processing
- **Overview:** Prepares structured prompt payloads and queries OpenAI to generate strategic fixes for each flagged ad.
- **Nodes Involved:** `Build AI quality prompt`, `Explain quality fixes with OpenAI`
- **Node Details:**
  - **Build AI quality prompt**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2). Formats system and user prompts for OpenAI, restricting the output to valid JSON.
    - *Configuration Choices:* Prepares a payload using the `gpt-4o-mini` model with JSON response formatting enabled.
    - *Input/Output:* Input from `Flag below-average ads`; output to `Explain quality fixes with OpenAI`.
    - *Edge Cases:* Large payloads could exceed token limits, mitigated here by slicing the top 20 flagged ads.
  - **Explain quality fixes with OpenAI**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Sends a POST request to OpenAI’s chat completions endpoint.
    - *Configuration Choices:* Uses Bearer token authentication via environment variables and passes the JSON body generated by the preceding code node.
    - *Key Expressions:* `={{ $json.body }}`, `=Bearer {{ $env.OPENAI_API_KEY }}`
    - *Input/Output:* Input from `Build AI quality prompt`; output to `Print quality digest report`.
    - *Edge Cases:* API timeouts, insufficient OpenAI account credits, or malformed JSON responses from the model.

#### 2.4 Reporting & Delivery
- **Overview:** Parses the AI response, formats a clean text-based digest report, and sends it via email.
- **Nodes Involved:** `Print quality digest report`, `Send a message`
- **Node Details:**
  - **Print quality digest report**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2). Parses the AI JSON response, extracts summaries and fixes, and compiles a readable text report.
    - *Configuration Choices:* Includes robust regex fallback logic to extract JSON objects from the model's response text.
    - *Input/Output:* Input from `Explain quality fixes with OpenAI`; output to `Send a message`.
    - *Edge Cases:* Handling unparseable AI responses by providing a default fallback structure.
  - **Send a message**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2). Sends an email containing the formatted quality digest report.
    - *Configuration Choices:* Configured to send a message using the generated report text.
    - *Key Expressions:* `={{ $json.report }}`
    - *Input/Output:* Input from `Print quality digest report`; no downstream output.
    - *Edge Cases:* Gmail authentication failures or missing recipient configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run manually` | `manualTrigger` | Initiates workflow execution manually. | None | `Set config: quality digest` | Meta Below-Average Quality-Ranking Digest<br><br>Surfaces the ads Meta is quietly penalising. Meta scores every ad below/average/above average on quality, engagement-rate and conversion-rate ranking; below average means higher costs and less reach. This finds them and tells you which lever to pull.<br><br>### How it works<br>- Pulls ad-level insights with the three Meta ranking diagnostics from the Marketing API.<br>- Flags ads spending over a floor that are below average on any ranking.<br>- Maps each ranking to its lever: quality to creative, engagement to hook, conversion to offer or landing page.<br>- OpenAI gives a specific fix per ad, worst-spend first.<br><br>### Setup<br>1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, OPENAI_API_KEY.<br>2. Run with the manual trigger, or attach a Schedule trigger for a weekly digest.<br><br>### Customization<br>In the Config node, tune MIN_SPEND. Swap the report node for a Slack or email node. (Keep DATE_PRESET short. Meta only populates the three rankings on a recent rolling window, so last_30d returns UNKNOWN for every ad regardless of spend or impressions. last_7d is the safe default.)<br><br>Built by **nocode.expert** - done-for-you automation & tracking.<br><br>Full walkthrough of this workflow: https://nocode.expert/resources/n8n-meta-ads-quality-ranking-automation |
| `Set config: quality digest` | `code` | Sets configuration variables for date preset and minimum spend. | `Run manually` | `Fetch ad quality rankings (Meta)` | Meta Below-Average Quality-Ranking Digest<br><br>Surfaces the ads Meta is quietly penalising. Meta scores every ad below/average/above average on quality, engagement-rate and conversion-rate ranking; below average means higher costs and less reach. This finds them and tells you which lever to pull.<br><br>### How it works<br>- Pulls ad-level insights with the three Meta ranking diagnostics from the Marketing API.<br>- Flags ads spending over a floor that are below average on any ranking.<br>- Maps each ranking to its lever: quality to creative, engagement to hook, conversion to offer or landing page.<br>- OpenAI gives a specific fix per ad, worst-spend first.<br><br>### Setup<br>1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, OPENAI_API_KEY.<br>2. Run with the manual trigger, or attach a Schedule trigger for a weekly digest.<br><br>### Customization<br>In the Config node, tune MIN_SPEND. Swap the report node for a Slack or email node. (Keep DATE_PRESET short. Meta only populates the three rankings on a recent rolling window, so last_30d returns UNKNOWN for every ad regardless of spend or impressions. last_7d is the safe default.)<br><br>Built by **nocode.expert** - done-for-you automation & tracking.<br><br>Full walkthrough of this workflow: https://nocode.expert/resources/n8n-meta-ads-quality-ranking-automation |
| `Fetch ad quality rankings (Meta)` | `httpRequest` | Fetches ad insights and rankings from the Meta Graph API. | `Set config: quality digest` | `Flag below-average ads` | ## 1. Fetch rankings<br>Pull ad-level quality, engagement and conversion rankings. |
| `Flag below-average ads` | `code` | Filters ads by minimum spend and extracts below-average ratings. | `Fetch ad quality rankings (Meta)` | `Build AI quality prompt` | ## 2. Flag below-average<br>Find ads below average, then OpenAI gives a fix per ad. |
| `Build AI quality prompt` | `code` | Constructs the system and user prompts for OpenAI. | `Flag below-average ads` | `Explain quality fixes with OpenAI` | ## 2. Flag below-average<br>Find ads below average, then OpenAI gives a fix per ad. |
| `Explain quality fixes with OpenAI` | `httpRequest` | Sends prompt to OpenAI and receives AI analysis. | `Build AI quality prompt` | `Print quality digest report` | ## 2. Flag below-average<br>Find ads below average, then OpenAI gives a fix per ad. |
| `Print quality digest report` | `code` | Parses AI output and formats the text report. | `Explain quality fixes with OpenAI` | `Send a message` | ## 3. Report<br>Print the digest (swap for Slack/email). |
| `Optional: run this weekly` | `scheduleTrigger` | Optional weekly trigger (disabled by default). | None | None | ## Optional add-ons (not wired up)<br><br>Left disconnected so you can plug in only what you need.<br><br>- **Run this weekly** - quality ranking moves slowly and the default window is 30 days, so weekly is the right cadence. Swap this in for the manual trigger.<br>- **Log flagged ads to Google Sheets** - connect after the report node. This is the one that pays off: a ranking is only useful once you can see whether last week's creative fix actually moved it.<br>- **Email the digest via Outlook** - connect after the report node to send rather than read in the editor.<br><br>None are required. The workflow runs end to end from the manual trigger with all three disconnected. |
| `Optional: log flagged ads to Google Sheets` | `googleSheets` | Optional logging node (disabled by default). | None | None | ## Optional add-ons (not wired up)<br><br>Left disconnected so you can plug in only what you need.<br><br>- **Run this weekly** - quality ranking moves slowly and the default window is 30 days, so weekly is the right cadence. Swap this in for the manual trigger.<br>- **Log flagged ads to Google Sheets** - connect after the report node. This is the one that pays off: a ranking is only useful once you can see whether last week's creative fix actually moved it.<br>- **Email the digest via Outlook** - connect after the report node to send rather than read in the editor.<br><br>None are required. The workflow runs end to end from the manual trigger with all three disconnected. |
| `Send a message` | `gmail` | Sends the formatted quality digest report via Gmail. | `Print quality digest report` | None | ## 3. Report<br>Print the digest (swap for Slack/email). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`).
2. **Configure Settings Code Node:**
   - Add a **Code** node named `Set config: quality digest`.
   - Set the mode to JavaScript and insert configuration return logic for `DATE_PRESET` (e.g., `'last_7d'`) and `MIN_SPEND` (e.g., `20`).
   - Connect `Run manually` to `Set config: quality digest`.
3. **Configure Meta API HTTP Request Node:**
   - Add an **HTTP Request** node named `Fetch ad quality rankings (Meta)`.
   - Set method to `GET`, URL to `https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`.
   - Add query parameters: `access_token` (`={{ $env.META_ACCESS_TOKEN }}`), `level` (`ad`), `date_preset` (`={{ $('Set config: quality digest').first().json.DATE_PRESET }}`), `fields` (`ad_name,campaign_name,adset_name,spend,impressions,quality_ranking,engagement_rate_ranking,conversion_rate_ranking`), and `limit` (`500`).
   - Connect `Set config: quality digest` to this node.
4. **Configure Filtering Code Node:**
   - Add a **Code** node named `Flag below-average ads`.
   - Insert logic to filter rows where spend is below `MIN_SPEND` and isolate fields starting with `BELOW_AVERAGE`.
   - Connect `Fetch ad quality rankings (Meta)` to this node.
5. **Configure AI Prompt Construction Node:**
   - Add a **Code** node named `Build AI quality prompt`.
   - Insert JavaScript code to structure system and user prompts instructing OpenAI (`gpt-4o-mini`) to return a JSON response containing ad optimization fixes.
   - Connect `Flag below-average ads` to this node.
6. **Configure OpenAI HTTP Request Node:**
   - Add an **HTTP Request** node named `Explain quality fixes with OpenAI`.
   - Set method to `POST`, URL to `https://api.openai.com/v1/chat/completions`.
   - Set Headers: `Authorization` (`=Bearer {{ $env.OPENAI_API_KEY }}`) and `content-type` (`application/json`).
   - Set Body to JSON (`={{ $json.body }}`).
   - Connect `Build AI quality prompt` to this node.
7. **Configure Report Generation Node:**
   - Add a **Code** node named `Print quality digest report`.
   - Insert parsing logic to extract the JSON response from OpenAI and compile a plaintext summary report.
   - Connect `Explain quality fixes with OpenAI` to this node.
8. **Configure Gmail Node:**
   - Add a **Gmail** node named `Send a message`.
   - Set resource/operation to send a message.
   - Configure the message parameter to `={{ $json.report }}`.
   - Authenticate with your Gmail OAuth2 credentials.
   - Connect `Print quality digest report` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by nocode.expert - done-for-you automation & tracking. | Project credits / Branding |
| Full walkthrough of this workflow | https://nocode.expert/resources/n8n-meta-ads-quality-ranking-automation |