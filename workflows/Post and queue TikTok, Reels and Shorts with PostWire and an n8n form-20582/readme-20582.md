Post and queue TikTok, Reels and Shorts with PostWire and an n8n form

https://n8nworkflows.xyz/workflows/post-and-queue-tiktok--reels-and-shorts-with-postwire-and-an-n8n-form-20582


# Post and queue TikTok, Reels and Shorts with PostWire and an n8n form

### 1. Workflow Overview

This workflow automates the multi-platform distribution of vertical videos to TikTok, Instagram Reels, and YouTube Shorts. It provides a centralized interface for team members or clients to upload video assets, generate AI-assisted platform-specific captions, review and edit drafts, and selectively publish them either immediately or via automated scheduling queues.

The workflow is logically structured into four sequential blocks:

- **1.1 Upload and Host:** Receives user inputs (video file, description, target platforms, and timing preferences) via an interactive n8n Form trigger and uploads the binary video file to PostWire to obtain a public media URL.
- **1.2 Captions and Review:** Normalizes form parameters, leverages PostWire's generation engine to produce tailored drafts for each specified social network, and presents a secondary interactive form for user validation, editing, and final decision-making (Publish vs. Cancel).
- **1.3 Post Now or Queue:** Handles conditional user decisions, routing cancelled requests to a termination page, while processing approved edits to execute either an immediate publication or a queued schedule via PostWire.
- **1.4 Log and Show the Links:** Aggregates multi-platform API responses, formats a structured execution summary, logs the transaction details to a Google Sheets document, and displays a final success completion page containing the generated URLs or status alerts.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Upload and Host
- **Overview:** Captures initial metadata and binary video files from an external uploader via an n8n Form, then transmits the media asset to PostWire to generate a publicly accessible download URL required by social platforms.
- **Nodes Involved:** `Upload form`, `Host the video`, `Read the form`
- **Node Details:**
  - **Upload form** (`n8n-nodes-base.formTrigger`, v2.5)
    - *Technical Role:* Acts as the primary workflow entry point by rendering an interactive submission form.
    - *Configuration:* Collects a video file (`.mp4`, `.mov`, required), a textual description (`about`, required), a checkbox selection of target networks (`TikTok`, `Instagram Reels`, `YouTube Shorts`), and a scheduling preference radio option (`Now`, `Next free slot in my queue`).
    - *Key Expressions:* None (triggers execution context).
    - *Input / Output Connections:* Input: Webhook trigger. Output: `Host the video`.
    - *Edge Cases / Failures:* Unsupported file formats or oversized files exceeding the 1 GB PostWire threshold will fail processing.
  - **Host the video** (`n8n-nodes-postwire.postWire`, v2)
    - *Technical Role:* Uploads the binary video file to the PostWire media hosting service.
    - *Configuration:* Resource: `media`, Operation: `upload`. Dynamically detects the binary property name.
    - *Key Expressions:* `={{ Object.keys($binary || {})[0] || 'video' }}`
    - *Input / Output Connections:* Input: `Upload form`. Output: `Read the form`.
    - *Edge Cases / Failures:* Authentication errors due to invalid PostWire API credentials or network timeouts on large file transfers.
  - **Read the form** (`n8n-nodes-base.set`, v3.4)
    - *Technical Role:* Normalizes data structures received from both the initial form trigger and the media hosting node.
    - *Configuration:* Maps raw display network names to internal system identifiers (`TikTok` -> `tiktok`, `Instagram Reels` -> `instagram`, `YouTube Shorts` -> `youtube`).
    - *Key Expressions:* 
      - `media_url`: `={{ $json.media_url }}`
      - `about`: `={{ $('Upload form').item.json.about }}`
      - `networks`: `={{ [].concat($('Upload form').item.json.networks || []).map(n => ({ 'TikTok': 'tiktok', 'Instagram Reels': 'instagram', 'YouTube Shorts': 'youtube' })[n]).filter(Boolean) }}`
      - `when`: `={{ $('Upload form').item.json.when }}`
    - *Input / Output Connections:* Input: `Host the video`. Output: `Write a caption for each network`.

---

#### Block 1.2: Captions and Review
- **Overview:** Generates contextual draft captions and titles via PostWire based on the video description, and displays an editable review form for user validation.
- **Nodes Involved:** `Write a caption for each network`, `Review the captions`, `Cancel?`
- **Node Details:**
  - **Write a caption for each network** (`n8n-nodes-postwire.postWire`, v2)
    - *Technical Role:* Generates platform-optimized text drafts using integrated AI writing models.
    - *Configuration:* Resource: `post`, Operation: `generate`. Uses URL-based media source with video media type.
    - *Key Expressions:*
      - Prompt: `={{ $json.about }}`
      - Media URL: `={{ $json.media_url }}`
      - Platforms: `={{ $json.networks }}`
    - *Input / Output Connections:* Input: `Read the form`. Output: `Review the captions`.
    - *Edge Cases / Failures:* AI generation failures or missing platform permissions in PostWire.
  - **Review the captions** (`n8n-nodes-base.form`, v2.5)
    - *Technical Role:* Renders an interactive review form containing prepopulated platform drafts and a final decision radio button.
    - *Configuration:* Form fields for TikTok, Instagram, YouTube descriptions, YouTube titles, and a required decision field (`Publish` vs. `Cancel`).
    - *Key Expressions:* Default text values dynamically map to nested JSON elements from the draft generation step (e.g., `={{ ($json.drafts.tiktok || {}).text || '' }}`).
    - *Input / Output Connections:* Input: `Write a caption for each network`. Output: `Cancel?`.
  - **Cancel?** (`n8n-nodes-base.if`, v2.2)
    - *Technical Role:* Evaluates the user's decision from the review form.
    - *Configuration:* Checks if the decision field strictly equals "Cancel".
    - *Key Expressions:* Left value: `={{ $json.decision }}`, Right value: `Cancel`.
    - *Input / Output Connections:* Input: `Review the captions`. Output (True): `Cancelled`. Output (False): `Use my edits`.

---

#### Block 1.3: Post Now or Queue
- **Overview:** Processes approved edits and routes payloads based on the selected publishing timeframe (immediate execution vs. queue scheduling).
- **Nodes Involved:** `Use my edits`, `Now or later?`, `Publish now`, `Add to my queue`
- **Node Details:**
  - **Use my edits** (`n8n-nodes-base.set`, v3.4)
    - *Technical Role:* Rebuilds the drafts payload by merging user modifications from the review form with original contextual properties.
    - *Configuration:* Overrides standard draft text properties with user-submitted form data.
    - *Key Expressions:* Executes an inline JavaScript function to iterate through selected networks and re-assign modified captions and YouTube titles.
    - *Input / Output Connections:* Input: `Cancel?` (False branch). Output: `Now or later?`.
  - **Now or later?** (`n8n-nodes-base.if`, v2.2)
    - *Technical Role:* Determines whether to publish immediately or schedule the post.
    - *Configuration:* Compares the timing preference variable against the string "Now".
    - *Key Expressions:* Left value: `={{ $json.when }}`, Right value: `Now`.
    - *Input / Output Connections:* Input: `Use my edits`. Output (True): `Publish now`. Output (False): `Add to my queue`.
  - **Publish now** (`n8n-nodes-postwire.postWire`, v2)
    - *Technical Role:* Immediately pushes the drafted posts to connected social channels.
    - *Configuration:* Resource: `post`, Operation: `publish`, Content Mode: `drafts`. Error handling set to `continueRegularOutput`.
    - *Key Expressions:* Drafts, media URL, and platforms mapped from upstream nodes.
    - *Input / Output Connections:* Input: `Now or later?` (True branch). Output: `Collect the results`.
    - *Edge Cases / Failures:* Platform API token expirations or rate limits.
  - **Add to my queue** (`n8n-nodes-postwire.postWire`, v2)
    - *Technical Role:* Schedules the posts into the next available queue slot within PostWire.
    - *Configuration:* Resource: `post`, Operation: `schedule`, `when: nextSlot`. Error handling set to `continueRegularOutput`.
    - *Key Expressions:* Label truncated from the `about` field: `={{ $('Read the form').item.json.about.slice(0, 60) }}`.
    - *Input / Output Connections:* Input: `Now or later?` (False branch). Output: `Collect the results`.

---

#### Block 1.4: Log and Show the Links
- **Overview:** Consolidates multi-platform execution outputs, writes audit logs to Google Sheets, and presents final completion links to the user.
- **Nodes Involved:** `Collect the results`, `Summarize`, `Log it`, `Show the links`, `Cancelled`
- **Node Details:**
  - **Collect the results** (`n8n-nodes-base.aggregate`, v1)
    - *Technical Role:* Collects execution items from parallel or sequential publishing paths into a single array data structure.
    - *Configuration:* Aggregate mode: `aggregateAllItemData`, Destination field name: `results`.
    - *Input / Output Connections:* Input: `Publish now`, `Add to my queue`. Output: `Summarize`.
  - **Summarize** (`n8n-nodes-base.set`, v3.4)
    - *Technical Role:* Transforms aggregated response objects into a human-readable multi-line summary string.
    - *Configuration:* Maps execution statuses, queue timestamps, and error messages into a formatted string separated by newlines (`\n`).
    - *Key Expressions:* Complex JavaScript mapping over `$json.results`.
    - *Input / Output Connections:* Input: `Collect the results`. Output: `Log it`.
  - **Log it** (`n8n-nodes-base.googleSheets`, v4.7)
    - *Technical Role:* Appends a structured audit record of the publishing run to a Google Sheets document.
    - *Configuration:* Operation: `append`. Schema columns mapped: `date`, `about`, `networks`, `when`, `result`.
    - *Key Expressions:* Timestamp generation via `={{ $now.toFormat('yyyy-MM-dd HH:mm') }}`.
    - *Input / Output Connections:* Input: `Summarize`. Output: `Show the links`.
    - *Edge Cases / Failures:* Google Sheets API write errors or mismatched column mappings.
  - **Show the links** (`n8n-nodes-base.form`, v2.5)
    - *Technical Role:* Renders the final completion page for successful submissions.
    - *Configuration:* Operation: `completion`, Respond with: `text`.
    - *Key Expressions:* Completion message set to summary: `={{ $('Summarize').item.json.summary }}`.
    - *Input / Output Connections:* Input: `Log it`. Output: None (Terminal node).
  - **Cancelled** (`n8n-nodes-base.form`, v2.5)
    - *Technical Role:* Renders a completion message if the user cancels during the caption review phase.
    - *Configuration:* Operation: `completion`, Respond with: `text`.
    - *Input / Output Connections:* Input: `Cancel?` (True branch). Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | n8n-nodes-base.stickyNote | Workflow documentation and configuration instructions | — | — | ## One upload form for TikTok, Reels and Shorts... |
| **Upload form** | n8n-nodes-base.formTrigger | Captures video and form metadata | (Webhook) | Host the video | ## 1. Upload and host<br>The form takes the file; PostWire gives it a public link. |
| **Host the video** | n8n-nodes-postwire.postWire | Uploads video binary to PostWire | Upload form | Read the form | ## 1. Upload and host<br>The form takes the file; PostWire gives it a public link. |
| **Read the form** | n8n-nodes-base.set | Normalizes form data and maps networks | Host the video | Write a caption for each network | ## 1. Upload and host<br>The form takes the file; PostWire gives it a public link. |
| **Write a caption for each network** | n8n-nodes-postwire.postWire | Generates AI draft captions | Read the form | Review the captions | ## 2. Captions and review<br>A native caption per network, edited by the uploader before anything goes out. |
| **Review the captions** | n8n-nodes-base.form | Interactive review and edit form | Write a caption for each network | Cancel? | ## 2. Captions and review<br>A native caption per network, edited by the uploader before anything goes out. |
| **Cancel?** | n8n-nodes-base.if | Evaluates user review decision | Review the captions | Cancelled, Use my edits | ## 2. Captions and review<br>A native caption per network, edited by the uploader before anything goes out. |
| **Use my edits** | n8n-nodes-base.set | Applies user edits to draft objects | Cancel? | Now or later? | ## 3. Post now or queue<br>Publish immediately, or in the brand's next free queue slot. |
| **Now or later?** | n8n-nodes-base.if | Routes by scheduling preference | Use my edits | Publish now, Add to my queue | ## 3. Post now or queue<br>Publish immediately, or in the brand's next free queue slot. |
| **Publish now** | n8n-nodes-postwire.postWire | Publishes posts immediately | Now or later? | Collect the results | ## 3. Post now or queue<br>Publish immediately, or in the brand's next free queue slot. |
| **Add to my queue** | n8n-nodes-postwire.postWire | Schedules posts into queue | Now or later? | Collect the results | ## 3. Post now or queue<br>Publish immediately, or in the brand's next free queue slot. |
| **Collect the results** | n8n-nodes-base.aggregate | Aggregates publishing responses | Publish now, Add to my queue | Summarize | ## 4. Log and show the links<br>One row per upload; the last page shows each link or the fix. |
| **Summarize** | n8n-nodes-base.set | Formats execution results string | Collect the results | Log it | ## 4. Log and show the links<br>One row per upload; the last page shows each link or the fix. |
| **Log it** | n8n-nodes-base.googleSheets | Appends audit row to spreadsheet | Summarize | Show the links | ## 4. Log and show the links<br>One row per upload; the last page shows each link or the fix. |
| **Show the links** | n8n-nodes-base.form | Renders completion success page | Log it | — | ## 4. Log and show the links<br>One row per upload; the last page shows each link or the fix. |
| **Cancelled** | n8n-nodes-base.form | Renders cancellation exit page | Cancel? | — | ## 2. Captions and review<br>A native caption per network, edited by the uploader before anything goes out. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger (`Upload form`)**
   - Type: `n8n-nodes-base.formTrigger` (v2.5)
   - Configuration: Form Title: "Post a video to TikTok, Reels and Shorts". Add fields: `video` (File, required, `.mp4,.mov`), `about` (Textarea, required), `networks` (Checkbox with options: TikTok, Instagram Reels, YouTube Shorts, required), `when` (Radio with options: Now, Next free slot in my queue, required).
2. **Add Video Hosting (`Host the video`)**
   - Type: `n8n-nodes-postwire.postWire` (v2)
   - Configuration: Resource: `media`, Operation: `upload`. Binary property: `={{ Object.keys($binary || {})[0] || 'video' }}`. Credential: PostWire API.
3. **Normalize Data (`Read the form`)**
   - Type: `n8n-nodes-base.set` (v3.4)
   - Configuration: Assign `media_url` (`={{ $json.media_url }}`), `about` (`={{ $('Upload form').item.json.about }}`), `networks` (mapped array), and `when` (`={{ $('Upload form').item.json.when }}`).
4. **Generate AI Drafts (`Write a caption for each network`)**
   - Type: `n8n-nodes-postwire.postWire` (v2)
   - Configuration: Resource: `post`, Operation: `generate`. Prompt: `={{ $json.about }}`, Media URL: `={{ $json.media_url }}`, Platforms: `={{ $json.networks }}`.
5. **Create Review Form (`Review the captions`)**
   - Type: `n8n-nodes-base.form` (v2.5)
   - Configuration: Add fields for `tiktok`, `instagram`, `youtube`, `youtube_title`, and `decision` (Radio: Publish, Cancel, required). Set default values referencing the generated drafts JSON.
6. **Evaluate Cancellation (`Cancel?`)**
   - Type: `n8n-nodes-base.if` (v2.2)
   - Configuration: Condition: `={{ $json.decision }}` equals `Cancel`.
   - Connect true branch to **Cancelled** (`n8n-nodes-base.form`, completion page).
7. **Process Edits (`Use my edits`)**
   - Type: `n8n-nodes-base.set` (v3.4)
   - Configuration: Use inline JavaScript to update draft objects with user review inputs.
8. **Evaluate Timing (`Now or later?`)**
   - Type: `n8n-nodes-base.if` (v2.2)
   - Configuration: Condition: `={{ $json.when }}` equals `Now`.
9. **Publish or Schedule (`Publish now` & `Add to my queue`)**
   - **Publish now:** `n8n-nodes-postwire.postWire` (v2), Resource: `post`, Operation: `publish`. Enable `Continue on Fail`.
   - **Add to my queue:** `n8n-nodes-postwire.postWire` (v2), Resource: `post`, Operation: `schedule`, `when: nextSlot`. Enable `Continue on Fail`.
10. **Aggregate and Summarize (`Collect the results` & `Summarize`)**
    - **Collect the results:** `n8n-nodes-base.aggregate` (v1), Aggregate All Item Data.
    - **Summarize:** `n8n-nodes-base.set` (v3.4), map multi-line summary string.
11. **Log and Complete (`Log it` & `Show the links`)**
    - **Log it:** `n8n-nodes-base.googleSheets` (v4.7), Operation: `append`. Configure Google Sheets OAuth2 credential and map schema columns (`date`, `about`, `networks`, `when`, `result`).
    - **Show the links:** `n8n-nodes-base.form` (v2.5), Operation: `completion`, display summary message.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostWire Community Node & Platform Setup | Install the verified PostWire node, create an account at [postwire.io](https://postwire.io), and connect social channels via OAuth. |
| Video Constraints & Specifications | Recommended format: MP4 (H.264), aspect ratio 9:16, duration 3-90 seconds, maximum file size 1 GB. |
| Free Plan Limitations | PostWire free plan supports up to 20 posts per month and 2 networks per post without requiring a credit card. |