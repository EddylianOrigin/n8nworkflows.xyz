Create faceless vertical Shorts with OpenAI Sora 2, PostWire, and Sheets

https://n8nworkflows.xyz/workflows/create-faceless-vertical-shorts-with-openai-sora-2--postwire--and-sheets-20581


# Create faceless vertical Shorts with OpenAI Sora 2, PostWire, and Sheets

### 1. Workflow Overview

This workflow automates the creation and multi-platform publishing of faceless vertical video Shorts. Operating on a daily schedule, it prevents duplicate content by tracking previously published topics in Google Sheets, generates a tailored script and visual prompt via OpenAI, produces a vertical video using OpenAI’s Sora 2 model, and publishes the result across YouTube Shorts, TikTok, and Instagram Reels using PostWire. Finally, it logs execution results and sends automated failure notifications via Gmail if any platform upload fails.

The system logic is organized into three sequential functional blocks:
- **1.1 Initialization and Topic History:** Triggers daily, loads baseline channel configurations, and aggregates historical post data from Google Sheets to prevent repetitive topics.
- **1.2 AI Ideation and Video Generation:** Employs an LLM chain with structured output to select a unique topic and write a Sora 2 prompt, subsequently rendering the vertical video clip.
- **1.3 Multi-Platform Publishing, Logging, and Error Handling:** Distributes the media via PostWire to multiple networks, branches conditionally based on API response success, logs individual outcomes to Google Sheets, and alerts the administrator via Gmail on failure.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Initialization and Topic History
- **Overview:** This block establishes the execution schedule, injects core channel metadata (niche, visual style, video length, alert recipient), and fetches past entries from storage to compile a exclusion list for the content generator.
- **Nodes Involved:** 
  - `Every day at 9:00`
  - `Channel settings`
  - `Read past videos`
  - `List past topics`

- **Node Details:**
  - **Every day at 9:00**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) — Acts as the time-based entry point.
    - *Configuration Choices:* Configured to trigger daily at hour 09:00.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Output connects to `Channel settings`.
    - *Edge Cases / Failures:* Missed executions if the n8n instance is offline during the scheduled hour.
  - **Channel settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) — Defines static parameters governing the content generation run.
    - *Configuration Choices:* Assigns string and numeric fields: `niche`, `style`, `seconds`, and `email`.
    - *Key Expressions:* Hardcoded string/numeric values.
    - *Input/Output Connections:* Input from `Every day at 9:00`; output connects to `Read past videos`.
    - *Edge Cases / Failures:* Type mismatch if downstream nodes expect alternate data types for duration (`seconds`).
  - **Read past videos**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Fetches existing rows from the tracking spreadsheet.
    - *Configuration Choices:* Operation set to read/retrieve rows (sheet and document IDs must be bound via UI list selectors). `alwaysOutputData` enabled.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Input from `Channel settings`; output connects to `List past topics`.
    - *Edge Cases / Failures:* API authentication failure, missing spreadsheet permissions, or rate limits.
  - **List past topics**
    - *Type and Technical Role:* `n8n-nodes-base.aggregate` (v1) — Consolidates rows into a single structured array containing historical topics.
    - *Configuration Choices:* Field to aggregate set to `topic`.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Input from `Read past videos`; output connects to `Write today's Short`.
    - *Edge Cases / Failures:* Empty sheet datasets resulting in empty arrays (handled gracefully by downstream filtering).

---

#### Block 1.2: AI Ideation and Video Generation
- **Overview:** This block uses the aggregated topic history and channel parameters to prompt an OpenAI chat model, extracts a validated JSON content plan, and commands the Sora 2 model to render the targeted vertical video file.
- **Nodes Involved:**
  - `Write today's Short`
  - `OpenAI Chat Model`
  - `Short plan format`
  - `Generate the video with Sora 2`

- **Node Details:**
  - **Write today's Short**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.7) — LangChain orchestration node driving the ideation prompt and enforcing structure.
    - *Configuration Choices:* Prompt type defined inline; output parser enabled.
    - *Key Expressions:* 
      - Input text expression dynamically injects niche, style, length, and slices the last 60 unique historical topics: `={{ 'Niche: ' + $('Channel settings').first().json.niche + '\nVisual style: ' + $('Channel settings').first().json.style + '\nLength: ' + $('Channel settings').first().json.seconds + ' seconds\nTopics already posted (never repeat or paraphrase them):\n' + ($json.topic || []).filter(Boolean).slice(-60).join('\n') }}`
    - *Input/Output Connections:* Input from `List past topics`; linked to `OpenAI Chat Model` (AI language model) and `Short plan format` (AI output parser); output connects to `Generate the video with Sora 2`.
    - *Edge Cases / Failures:* LLM quota exhaustion, invalid JSON output failing parser validation, or API timeouts.
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) — Supplies the underlying language model configuration for the chain.
    - *Configuration Choices:* Model selected as `gpt-5-mini`.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Connected to `Write today's Short` via `ai_languageModel`.
    - *Edge Cases / Failures:* Invalid API credentials.
  - **Short plan format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — Enforces structural output compliance on the LLM response.
    - *Configuration Choices:* Schema example defined with properties: `topic`, `sora_prompt`, `caption_idea`, and `youtube_title`.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Connected to `Write today's Short` via `ai_outputParser`.
    - *Edge Cases / Failures:* LLM failing to return exact schema keys.
  - **Generate the video with Sora 2**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (v2.1) — Interacts with OpenAI's video generation capabilities.
    - *Configuration Choices:* Resource set to `video`, operation set to `generate`, model ID set to `sora-2`, dimensions set to `720x1280`.
    - *Key Expressions:* 
      - Prompt: `={{ $json.output.sora_prompt }}`
      - Seconds: `={{ $('Channel settings').first().json.seconds }}`
    - *Input/Output Connections:* Input from `Write today's Short`; output connects to `PostWatch, TikTok and Reels`.
    - *Edge Cases / Failures:* High latency generation timeouts, credit balance exhaustion for Sora 2 billing tiers, or content policy blocks on generated prompts.

---

#### Block 1.3: Multi-Platform Publishing, Logging, and Error Handling
- **Overview:** This block manages distribution across multiple social networks via PostWire, evaluates success via conditional branching, records outcomes to Google Sheets, and dispatches email alerts if execution fails.
- **Nodes Involved:**
  - `Post to Shorts, TikTok and Reels`
  - `Published?`
  - `Log the video`
  - `Log the failure`
  - `Email me the fix`

- **Node Details:**
  - **Post to Shorts, TikTok and Reels**
    - *Type and Technical Role:* `n8n-nodes-postwire.postWire` (v2) — External multi-platform publishing wrapper.
    - *Configuration Choices:* Resource set to `post`, operation set to `publish`, platforms target `youtube`, `tiktok`, and `instagram`. Content mode set to `smart`, media source set to `binary`, binary property name set to `data`. `onError` set to `continueRegularOutput`.
    - *Key Expressions:*
      - Prompt: `={{ $('Write today\'s Short').item.json.output.caption_idea }}`
      - Options Title: `={{ $('Write today\'s Short').item.json.output.youtube_title }}`
      - Idempotency Key: `={{ 'short-' + $now.toFormat('yyyy-MM-dd') + '-' + $('Write today\'s Short').item.json.output.topic.slice(0, 80) }}`
    - *Input/Output Connections:* Input from `Generate the video with Sora 2`; output connects to `Published?`.
    - *Edge Cases / Failures:* Plan limits reached (free plan restricts network counts), expired platform tokens in PostWire dashboard, or network upload errors (caught by `continueRegularOutput`).
  - **Published?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) — Evaluates success status of downstream posting items.
    - *Configuration Choices:* Condition checks if `$json.ok` evaluates to true.
    - *Key Expressions:* 
      - Left Value: `={{ $json.ok }}` (Operator: Boolean true)
    - *Input/Output Connections:* Input from `Post to Shorts, TikTok and Reels`; True branch connects to `Log the video`, False branch connects to `Log the failure`.
    - *Edge Cases / Failures:* Malformed response payloads missing the `ok` property.
  - **Log the video**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Appends success logs to tracking sheet.
    - *Configuration Choices:* Operation set to `append`. Mapping mode set to `defineBelow`.
    - *Key Expressions:*
      - date: `={{ $now.toFormat('yyyy-MM-dd') }}`
      - link: `={{ $json.url || 'processing: the link appears in PostWire' }}`
      - note: (empty string)
      - topic: `={{ $('Write today\'s Short').item.json.output.topic }}`
      - status: `published`
      - network: `={{ $json.platform }}`
    - *Input/Output Connections:* Input from `Published?` (True branch).
    - *Edge Cases / Failures:* Spreadsheet API write errors or schema mismatches.
  - **Log the failure**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Appends error diagnostics to tracking sheet.
    - *Configuration Choices:* Operation set to `append`. Mapping mode set to `defineBelow`.
    - *Key Expressions:*
      - date: `={{ $now.toFormat('yyyy-MM-dd') }}`
      - link: (empty string)
      - note: `={{ $json.error }} {{ $json.hint || $json.description || '' }}`
      - topic: `={{ $('Write today\'s Short').item.json.output.topic }}`
      - status: `failed`
      - network: `={{ $json.platform || 'all' }}`
    - *Input/Output Connections:* Input from `Published?` (False branch); output connects to `Email me the fix`.
    - *Edge Cases / Failures:* Spreadsheet write failures compounding error states.
  - **Email me the fix**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.1) — Dispatches administrative failure notifications.
    - *Configuration Choices:* Email type set to text, options disable attribution.
    - *Key Expressions:*
      - Send To: `={{ $('Channel settings').first().json.email }}`
      - Subject: `={{ ($json.network === 'all' ? 'The Short' : $json.network) + ' was not posted: ' + $json.topic }}`
      - Message: `={{ 'Topic: ' + $json.topic + '\n\nWhat happened and how to fix it:\n' + $json.note }}`
    - *Input/Output Connections:* Input from `Log the failure`.
    - *Edge Cases / Failures:* Gmail authentication errors, invalid recipient address configuration, or SMTP rate limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Visual documentation and setup guide. | None | None | ## Faceless YouTube Shorts on autopilot with Sora 2<br>Every morning an AI picks a new idea in your niche, Sora 2 renders a vertical video with sound, and PostWire posts it as a YouTube Short, a TikTok and an Instagram Reel, each with its own caption. Every video is logged, so the AI never repeats a topic.<br><br>### How it works<br>1. **Every day at 9:00** starts the run; **Channel settings** holds your niche, style and length.<br>2. **Read past videos** and **List past topics** load what you already posted.<br>3. **Write today's Short** (OpenAI) returns a new topic, a Sora 2 prompt and a caption idea.<br>4. **Generate the video with Sora 2** renders a 9:16 clip with audio.<br>5. **Post to Shorts, TikTok and Reels** uploads it once; PostWire writes a native caption, YouTube title and tags per network.<br>6. **Published?** logs each network's link in Google Sheets, or logs the error and emails you the fix.<br><br>### Setup<br>1. Install the verified **PostWire** node (search "PostWire" in the nodes panel), create an account at postwire.io, connect your networks (one click each: no developer app, no Meta or TikTok review) and paste your API key into a **PostWire API** credential.<br>2. Add an **OpenAI** credential to the chat model and to the Sora 2 node (Sora 2 is billed by OpenAI per second of video).<br>3. Create a Google Sheet with the columns `date, topic, network, status, link, note`, select it in the three Google Sheets nodes and add a Gmail credential.<br>4. Edit **Channel settings**: niche, style, seconds (4, 8 or 12) and your email.<br><br>Free PostWire plan: 20 posts a month and 2 networks per post, no card; 3 or more networks per post need a paid plan. |
| **Every day at 9:00** | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow execution daily at 9 AM. | None | Channel settings | |
| **Channel settings** | `n8n-nodes-base.set` | Sets global configuration variables (niche, style, duration, email). | Every day at 9:00 | Read past videos | |
| **Read past videos** | `n8n-nodes-base.googleSheets` | Reads existing video history records from Google Sheets. | Channel settings | List past topics | |
| **List past topics** | `n8n-nodes-base.aggregate` | Aggregates previous topics into a single array to prevent duplication. | Read past videos | Write today's Short | |
| **Write today's Short** | `@n8n/n8n-nodes-langchain.chainLlm` | AI chain that generates a unique topic, Sora prompt, and caption. | List past topics | Generate the video with Sora 2 | ## 1. Pick a new idea<br>Your niche and the topics already posted, so nothing repeats. |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Supplies the language model (gpt-5-mini) for the ideation chain. | None | Write today's Short | ## 1. Pick a new idea<br>Your niche and the topics already posted, so nothing repeats. |
| **Short plan format** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Ensures the AI response follows a strict JSON schema. | None | Write today's Short | ## 1. Pick a new idea<br>Your niche and the topics already posted, so nothing repeats. |
| **Generate the video with Sora 2** | `@n8n/n8n-nodes-langchain.openAi` | Generates a 9:16 vertical video clip using OpenAI Sora 2 based on prompt. | Write today's Short | Post to Shorts, TikTok and Reels | ## 2. Script and render<br>OpenAI writes the Sora 2 prompt; Sora 2 renders 9:16 video with sound. |
| **Post to Shorts, TikTok and Reels** | `n8n-nodes-postwire.postWire` | Publishes the generated video across YouTube, TikTok, and Instagram. | Generate the video with Sora 2 | Published? | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |
| **Published?** | `n8n-nodes-base.if` | Evaluates whether the PostWire publishing action succeeded. | Post to Shorts, TikTok and Reels | Log the video, Log the failure | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |
| **Log the video** | `n8n-nodes-base.googleSheets` | Logs successful publish events and URLs to Google Sheets. | Published? | None | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |
| **Log the failure** | `n8n-nodes-base.googleSheets` | Logs error details to Google Sheets when publishing fails. | Published? | Email me the fix | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |
| **Email me the fix** | `n8n-nodes-base.gmail` | Sends a notification email detailing posting errors to the administrator. | Log the failure | None | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |
| **Section: 1. Pick a new idea** | `n8n-nodes-base.stickyNote` | Visual grouping container for initial ideation steps. | None | None | ## 1. Pick a new idea<br>Your niche and the topics already posted, so nothing repeats. |
| **Section: 2. Script and render** | `n8n-nodes-base.stickyNote` | Visual grouping container for scriptwriting and video rendering. | None | None | ## 2. Script and render<br>OpenAI writes the Sora 2 prompt; Sora 2 renders 9:16 video with sound. |
| **Section: 3. Post, log, alert** | `n8n-nodes-base.stickyNote` | Visual grouping container for multi-platform publishing and logging. | None | None | ## 3. Post, log, alert<br>One upload, a native caption per network, each link logged, failures emailed with the fix. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every day at 9:00`.
   - Set interval rule to trigger daily at hour `9`.

2. **Configure Channel Settings:**
   - Add a **Set** node named `Channel settings`.
   - Create string assignments: `niche` (e.g., `"Little-known facts about the ocean"`), `style` (e.g., `"Cinematic, vertical 9:16..."`), `email` (e.g., `"you@example.com"`).
   - Create a number assignment: `seconds` (value: `12`).
   - *Connection:* Connect `Every day at 9:00` output to `Channel settings`.

3. **Fetch Historical Data:**
   - Add a **Google Sheets** node named `Read past videos`.
   - Select your target document and sheet (ensure columns include `date, topic, network, status, link, note`). Enable `alwaysOutputData`.
   - *Connection:* Connect `Channel settings` to `Read past videos`.

4. **Aggregate Historical Topics:**
   - Add an **Aggregate** node named `List past topics`.
   - Set fields to aggregate: field name `topic`.
   - *Connection:* Connect `Read past videos` to `List past topics`.

5. **Configure AI Ideation Chain:**
   - Add an **Advanced AI (Chain LLM)** node named `Write today's Short`.
   - Set prompt type to define inline, and enable output parser.
   - Insert prompt text combining niche, style, seconds, and historical topics sliced from the aggregate node:
     `={{ 'Niche: ' + $('Channel settings').first().json.niche + '\nVisual style: ' + $('Channel settings').first().json.style + '\nLength: ' + $('Channel settings').first().json.seconds + ' seconds\nTopics already posted (never repeat or paraphrase them):\n' + ($json.topic || []).filter(Boolean).slice(-60).join('\n') }}`
   - Add an **OpenAI Chat Model** node named `OpenAI Chat Model`, configure model to `gpt-5-mini`, and attach it to the chain's language model input. Configure OpenAI credentials.
   - Add a **Structured Output Parser** node named `Short plan format`. Define its JSON schema example with keys: `topic`, `sora_prompt`, `caption_idea`, `youtube_title`. Attach it to the chain's output parser input.
   - *Connection:* Connect `List past topics` to `Write today's Short`.

6. **Generate Video via Sora 2:**
   - Add an **OpenAI** node named `Generate the video with Sora 2`.
   - Set resource to `video`, operation to `generate`, model ID to `sora-2`, and size to `720x1280`.
   - Set Prompt expression to `={{ $json.output.sora_prompt }}` and Seconds expression to `={{ $('Channel settings').first().json.seconds }}`. Configure OpenAI credentials.
   - *Connection:* Connect `Write today's Short` to `Generate the video with Sora 2`.

7. **Publish via PostWire:**
   - Add a **PostWire** node named `Post to Shorts, TikTok and Reels`.
   - Set resource to `post`, operation to `publish`, content mode to `smart`, media source to `binary`, binary property name to `data`.
   - Select platforms: `youtube`, `tiktok`, `instagram`. Set `onError` behavior to `Continue (Using Regular Output)`.
   - Set Prompt expression to `={{ $('Write today\'s Short').item.json.output.caption_idea }}`, title option to `={{ $('Write today\'s Short').item.json.output.youtube_title }}`, and idempotencyKey option to `={{ 'short-' + $now.toFormat('yyyy-MM-dd') + '-' + $('Write today\'s Short').item.json.output.topic.slice(0, 80) }}`. Configure PostWire API credentials.
   - *Connection:* Connect `Generate the video with Sora 2` to `Post to Shorts, TikTok and Reels`.

8. **Implement Conditional Routing:**
   - Add an **If** node named `Published?`.
   - Set condition check: left value `={{ $json.ok }}` equals boolean true.
   - *Connection:* Connect `Post to Shorts, TikTok and Reels` to `Published?`.

9. **Configure Success Logging:**
   - Add a **Google Sheets** node named `Log the video`.
   - Set operation to `append` and map columns (`date`, `topic`, `network`, `status`, `link`, `note`) using expressions:
     - `date`: `={{ $now.toFormat('yyyy-MM-dd') }}`
     - `topic`: `={{ $('Write today\'s Short').item.json.output.topic }}`
     - `network`: `={{ $json.platform }}`
     - `status`: `published`
     - `link`: `={{ $json.url || 'processing: the link appears in PostWire' }}`
     - `note`: (leave blank)
   - *Connection:* Connect `Published?` True output to `Log the video`.

10. **Configure Failure Logging and Alerting:**
    - Add a **Google Sheets** node named `Log the failure`.
    - Set operation to `append` and map columns using expressions:
      - `date`: `={{ $now.toFormat('yyyy-MM-dd') }}`
      - `topic`: `={{ $('Write today\'s Short').item.json.output.topic }}`
      - `network`: `={{ $json.platform || 'all' }}`
      - `status`: `failed`
      - `link`: (leave blank)
      - `note`: `={{ $json.error }} {{ $json.hint || $json.description || '' }}`
    - *Connection:* Connect `Published?` False output to `Log the failure`.
    - Add a **Gmail** node named `Email me the fix`.
    - Set recipient expression to `={{ $('Channel settings').first().json.email }}`, subject expression to `={{ ($json.network === 'all' ? 'The Short' : $json.network) + ' was not posted: ' + $json.topic }}`, and message expression to `={{ 'Topic: ' + $json.topic + '\n\nWhat happened and how to fix it:\n' + $json.note }}`. Configure Gmail credentials and disable attribution.
    - *Connection:* Connect `Log the failure` output to `Email me the fix`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostWire Platform Setup & Free Plan Limits | PostWire accounts can be registered at [postwire.io](https://postwire.io). The free plan includes 20 posts per month and restricts usage to 2 networks per post without requiring a credit card. Paid plans are required for 3 or more networks per post. |
| OpenAI Sora 2 Billing Notice | Sora 2 video generation is billed by OpenAI per second of rendered video. Ensure your OpenAI account is appropriately funded and possesses proper video generation access tiers. |