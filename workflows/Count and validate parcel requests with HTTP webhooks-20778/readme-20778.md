Count and validate parcel requests with HTTP webhooks

https://n8nworkflows.xyz/workflows/count-and-validate-parcel-requests-with-http-webhooks-20778


# Count and validate parcel requests with HTTP webhooks

### 1. Workflow Overview

This workflow serves as a starter template for **Challenge 05: The Parcel Desk** (Season 2 of the n8n Community Weekly Challenge). Its primary purpose is to fetch a personal dataset of 30 parcel requests from an external API, parse the data into individual items, and provide isolated processing tracks to calculate three specific business metrics: requests needing correction, express-ready requests, and standard-ready requests. 

The logic is organized into the following functional blocks:
- **1.1 Initialization and Ingestion:** Triggers the workflow, supplies user credentials and base URL configurations, retrieves the remote dataset via an HTTP GET request, and fans out the resulting rows into individual execution items.
- **1.2 Stage 1 Processing (Needs Correction):** Placeholder branch intended to filter, count, and format requests that require correction, with an optional validation check endpoint.
- **1.3 Stage 2 Processing (Express Ready):** Placeholder branch intended to filter, count, and format valid requests destined for express delivery, with an optional validation check endpoint.
- **1.4 Stage 3 Processing (Standard Ready):** Placeholder branch intended to filter, count, and format valid requests destined for standard delivery, with an optional validation check endpoint.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Ingestion
This block starts execution, injects participant configuration data, retrieves the 30 parcel requests from the challenge endpoint, and splits the payload into individual rows for downstream analysis.

- **Start**
  - **Type and Technical Role:** `n8n-nodes-base.manualTrigger` (Trigger Node). Initiates the workflow execution manually.
  - **Configuration Choices:** Uses default manual trigger settings with no parameters.
  - **Key Expressions / Variables:** None.
  - **Input / Output Connections:** Input: None (Entry point) | Output: Connected to `Settings`.
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases / Potential Failure Types:** None.

- **Settings**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation / Setup Node). Initializes variables required for API calls.
  - **Configuration Choices:** Manual assignment mode, defining three string variables: `username`, `key`, and `base`.
  - **Key Expressions / Variables:** 
    - `username`: Empty string (intended for user input).
    - `key`: Empty string (participant authentication key).
    - `base`: `https://marketing.app.n8n.cloud/webhook/cwc/challenge-05`
  - **Input / Output Connections:** Input: `Start` | Output: Connected to `Get requests`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failure Types:** Authentication failures if `username` or `key` are left empty when making remote calls.

- **Get requests**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Fetches the participant's challenge dataset.
  - **Configuration Choices:** HTTP GET method, JSON response format, sends query parameters (`file=requests`, `username`) and header parameters (`x-participant-key`). Authentication is set to None (key is passed via headers).
  - **Key Expressions / Variables:** 
    - URL: `={{ $('Settings').first().json.base }}/data`
    - Query Parameter `username`: `={{ $('Settings').first().json.username }}`
    - Header `x-participant-key`: `={{ $('Settings').first().json.key }}`
  - **Input / Output Connections:** Input: `Settings` | Output: Connected to `Request rows`.
  - **Version-Specific Requirements:** Version 4.2.
  - **Edge Cases / Potential Failure Types:** HTTP 401/403 errors due to invalid credentials; network timeouts.

- **Request rows**
  - **Type and Technical Role:** `n8n-nodes-base.splitOut` (Data Transformation Node). Splits the array of parcel requests into 30 distinct items.
  - **Configuration Choices:** Splits out the `rows` array and strips extraneous root fields (`include: noOtherFields`).
  - **Key Expressions / Variables:** Field to split out: `rows`.
  - **Input / Output Connections:** Input: `Get requests` | Output: Unconnected in starter template (intended to feed custom validation logic for the three stages).
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases / Potential Failure Types:** Expression failure if the API response does not contain a valid `rows` array.

---

#### 2.2 Stage 1 Processing (Needs Correction)
This block captures the calculated answer for requests needing correction and provides a disabled endpoint to validate the result against the challenge server.

- **Stage 1 answer**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Assigns the calculated count to variable `b1`.
  - **Configuration Choices:** Manual assignment mode, defining a single string variable `b1`.
  - **Key Expressions / Variables:** `b1`: Empty string (expects final numeric count).
  - **Input / Output Connections:** Input: Unconnected in starter template (expects input from custom calculation logic) | Output: Connected to `Check stage 1`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failure Types:** Type mismatch if data passed is not recognized correctly as a number/string.

- **Check stage 1**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Submits and validates Stage 1 answers.
  - **Configuration Choices:** HTTP POST method, JSON response format, JSON body specified via key-value pairs (`username`, `stage=1`, `answer`), header parameter `x-participant-key`. Node is **disabled** by default.
  - **Key Expressions / Variables:**
    - URL: `={{ $('Settings').first().json.base }}/check`
    - Body `username`: `={{ $('Settings').first().json.username }}`
    - Body `answer`: `={{ $('Stage 1 answer').first().json.b1 }}`
    - Header `x-participant-key`: `={{ $('Settings').first().json.key }}`
  - **Input / Output Connections:** Input: `Stage 1 answer` | Output: None.
  - **Version-Specific Requirements:** Version 4.2.
  - **Edge Cases / Potential Failure Types:** API rejection if credentials are blank or the answer format is incorrect.

---

#### 2.3 Stage 2 Processing (Express Ready)
This block captures the calculated answer for express-ready requests and provides a disabled endpoint to validate the result.

- **Stage 2 answer**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Assigns the calculated count to variable `b2`.
  - **Configuration Choices:** Manual assignment mode, defining a single string variable `b2`.
  - **Key Expressions / Variables:** `b2`: Empty string (expects final numeric count).
  - **Input / Output Connections:** Input: Unconnected in starter template | Output: Connected to `Check stage 2`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failure Types:** Type mismatch if data passed is incorrect.

- **Check stage 2**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Submits and validates Stage 2 answers.
  - **Configuration Choices:** HTTP POST method, JSON response format, JSON body specified via key-value pairs (`username`, `stage=2`, `answer`), header parameter `x-participant-key`. Node is **disabled** by default.
  - **Key Expressions / Variables:**
    - URL: `={{ $('Settings').first().json.base }}/check`
    - Body `username`: `={{ $('Settings').first().json.username }}`
    - Body `answer`: `={{ $('Stage 2 answer').first().json.b2 }}`
    - Header `x-participant-key`: `={{ $('Settings').first().json.key }}`
  - **Input / Output Connections:** Input: `Stage 2 answer` | Output: None.
  - **Version-Specific Requirements:** Version 4.2.
  - **Edge Cases / Potential Failure Types:** API rejection if credentials are blank or the answer format is incorrect.

---

#### 2.4 Stage 3 Processing (Standard Ready)
This block captures the calculated answer for standard-ready requests and provides a disabled endpoint to validate the result.

- **Stage 3 answer**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Assigns the calculated count to variable `b3`.
  - **Configuration Choices:** Manual assignment mode, defining a single string variable `b3`.
  - **Key Expressions / Variables:** `b3`: Empty string (expects final numeric count).
  - **Input / Output Connections:** Input: Unconnected in starter template | Output: Connected to `Check stage 3`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failure Types:** Type mismatch if data passed is incorrect.

- **Check stage 3**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Submits and validates Stage 3 answers.
  - **Configuration Choices:** HTTP POST method, JSON response format, JSON body specified via key-value pairs (`username`, `stage=3`, `answer`), header parameter `x-participant-key`. Node is **disabled** by default.
  - **Key Expressions / Variables:**
    - URL: `={{ $('Settings').first().json.base }}/check`
    - Body `username`: `={{ $('Settings').first().json.username }}`
    - Body `answer`: `={{ $('Stage 3 answer').first().json.b3 }}`
    - Header `x-participant-key`: `={{ $('Settings').first().json.key }}`
  - **Input / Output Connections:** Input: `Stage 3 answer` | Output: None.
  - **Version-Specific Requirements:** Version 4.2.
  - **Edge Cases / Potential Failure Types:** API rejection if credentials are blank or the answer format is incorrect.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start | n8n-nodes-base.manualTrigger | Triggers workflow manually | None | Settings | ## Challenge 05 · The Parcel Desk<br><br>**Community Weekly Challenge · Season 2**<br><br>A shop has **30 parcel requests**. Find the requests that need a correction, then the ones ready for express or standard delivery.<br><br>**Starter, not a finished solution.** Build the missing native routing and counting nodes in the three work areas on the right. All three questions use your one personal batch.<br><br>### Your dataset<br><br>`request_id` · `contact_email` · `parcel_count` · `delivery`<br><br>Count **requests, not parcels**. Data and checks follow Challenge 05's participation window.<br><br>### Challenge links<br><br>[Read Challenge 05](https://marketing.app.n8n.cloud/webhook/cwc/challenge-05) · [Register](https://marketing.app.n8n.cloud/webhook/cwc/register) · [Your account](https://marketing.app.n8n.cloud/webhook/cwc/status)<br><br>[Submit](https://marketing.app.n8n.cloud/webhook/cwc/submit) · [Results](https://marketing.app.n8n.cloud/webhook/cwc/results) · [FAQ](https://marketing.app.n8n.cloud/webhook/cwc/faq)<br><br>[Learn and practise](https://marketing.app.n8n.cloud/webhook/cwc/learn) · [n8n documentation](https://docs.n8n.io/) |
| Settings | n8n-nodes-base.set | Stores user config variables | Start | Get requests | ## Challenge 05 · The Parcel Desk<br>...[Same Challenge 05 link block]...<br><br>## 1 · Fetch your personal data<br><br>[Read Challenge 05](https://marketing.app.n8n.cloud/webhook/cwc/challenge-05) and [register](https://marketing.app.n8n.cloud/webhook/cwc/register), or reuse your existing participant key. Enter your forum username and participant key in **Settings**. Keep the supplied base URL and HTTP Authentication **None**. Your key is sent in the `x-participant-key` header, never in the URL.<br><br>Run **Start** when Challenge 05 is open. The first successful personal data fetch starts the timer; re-fetching does not reset it. **Request rows** gives 30 items. Fetch your personal requests first. |
| Get requests | n8n-nodes-base.httpRequest | Fetches challenge dataset via GET | Settings | Request rows | ## Challenge 05 · The Parcel Desk<br>...[Same Challenge 05 link block]...<br><br>## 1 · Fetch your personal data<br>...[Same Fetch data description]... |
| Request rows | n8n-nodes-base.splitOut | Splits API response into 30 rows | Get requests | None (Starter endpoint) | ## Challenge 05 · The Parcel Desk<br>...[Same Challenge 05 link block]...<br><br>## 1 · Fetch your personal data<br>...[Same Fetch data description]...<br><br>## 4 · Finish & fair play<br><br>1. **Map your calculated counts** to `b1`, `b2` and `b3` as **Number**. Use a direct native Summarize count of `request_id`; no arithmetic, typed counts, grouped summaries or loops.<br>2. **Check when ready.** Checks start disabled, are optional and never stop the timer.<br>3. **Submit all three counts, your workflow JSON and your explanation.** Correct eligible answers enter the leaderboard immediately; the first accepted evidence stays locked.<br><br>**Timing:** first successful personal fetch → first accepted final. Weekly points are **15 / 12 / 12 / 11** by competition rank.<br><br>**No AI assistance of any kind**, including interpreting, planning, generating, debugging, solving or explaining this challenge. Chatbots, copilots, workflow builders and agents are included. No Code, Function, AI, Agent, MCP or Tool nodes.<br><br>**Before export:** blank Settings.key, set Authentication to None and remove all credential references and pinned data.<br><br>Workflows are reviewed at season end before rewards. |
| Stage 1 answer | n8n-nodes-base.set | Assigns Stage 1 answer variable `b1` | None | Check stage 1 | ## Question 1 · Needs correction<br><br>Count requests with an empty contact email OR zero parcels. If both are wrong, count that request once. Delivery does not rescue an invalid request.<br><br>**Your work area:** add native validation/routing and **Summarize → Count `request_id`**.<br><br>Map the calculated field directly to **Stage 1 answer → b1** as **Number**. Keep enabled forward paths from Request rows through your calculation to this answer. The Check node is optional.<br><br>**Build your calculation in the space below.** |
| Check stage 1 | n8n-nodes-base.httpRequest | Validates Stage 1 answer via POST | Stage 1 answer | None | ## Question 1 · Needs correction<br>...[Same Question 1 description]... |
| Stage 2 answer | n8n-nodes-base.set | Assigns Stage 2 answer variable `b2` | None | Check stage 2 | ## Question 2 · Express ready<br><br>Count valid requests whose delivery is EXPRESS. Validate before choosing the delivery queue.<br><br>**Your work area:** add native validation/routing and **Summarize → Count `request_id`**.<br><br>Map the calculated field directly to **Stage 2 answer → b2** as **Number**. Keep enabled forward paths from Request rows through your calculation to this answer. The Check node is optional.<br><br>**Build your calculation in the space below.** |
| Check stage 2 | n8n-nodes-base.httpRequest | Validates Stage 2 answer via POST | Stage 2 answer | None | ## Question 2 · Express ready<br>...[Same Question 2 description]... |
| Stage 3 answer | n8n-nodes-base.set | Assigns Stage 3 answer variable `b3` | None | Check stage 3 | ## Question 3 · Standard ready<br><br>Count valid requests whose delivery is STANDARD. The three calculated queue counts should add up to 30.<br><br>**Your work area:** add native validation/routing and **Summarize → Count `request_id`**.<br><br>Map the calculated field directly to **Stage 3 answer → b3** as **Number**. Keep enabled forward paths from Request rows through your calculation to this answer. The Check node is optional.<br><br>**Build your calculation in the space below.** |
| Check stage 3 | n8n-nodes-base.httpRequest | Validates Stage 3 answer via POST | Stage 3 answer | None | ## Question 3 · Standard ready<br>...[Same Question 3 description]... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger:** Add a `Manual Trigger` node (`n8n-nodes-base.manualTrigger`) named `Start`.
2. **Add Settings Node:** Create a `Set` node (`n8n-nodes-base.set`) named `Settings`. Set mode to `Manual`. Configure three string assignments:
   - `username`: `""`
   - `key`: `""`
   - `base`: `https://marketing.app.n8n.cloud/webhook/cwc/challenge-05`
   - Connect `Start` output to `Settings`.
3. **Add HTTP Request (Get requests):** Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Get requests`.
   - Method: `GET`
   - URL: `={{ $('Settings').first().json.base }}/data`
   - Query Parameters: `file` = `requests`, `username` = `={{ $('Settings').first().json.username }}`
   - Headers: `x-participant-key` = `={{ $('Settings').first().json.key }}`
   - Authentication: None
   - Connect `Settings` output to `Get requests`.
4. **Add Split Out Node:** Create a `Split Out` node (`n8n-nodes-base.splitOut`) named `Request rows`.
   - Field to Split Out: `rows`
   - Include: `noOtherFields`
   - Connect `Get requests` output to `Request rows`.
5. **Set up Stage 1 Branch:**
   - Create a `Set` node named `Stage 1 answer` (`n8n-nodes-base.set`) with assignment `b1` (string type, value `""`).
   - Create an `HTTP Request` node named `Check stage 1` (`n8n-nodes-base.httpRequest`), set to **Disabled** by default.
     - Method: `POST`
     - URL: `={{ $('Settings').first().json.base }}/check`
     - Send Body: JSON (Key-Pair format) -> `username`: `={{ $('Settings').first().json.username }}`, `stage`: `1`, `answer`: `={{ $('Stage 1 answer').first().json.b1 }}`
     - Headers: `x-participant-key` = `={{ $('Settings').first().json.key }}`
     - Authentication: None
   - Connect `Stage 1 answer` to `Check stage 1`.
6. **Set up Stage 2 Branch:**
   - Create a `Set` node named `Stage 2 answer` (`n8n-nodes-base.set`) with assignment `b2` (string type, value `""`).
   - Create an `HTTP Request` node named `Check stage 2` (`n8n-nodes-base.httpRequest`), set to **Disabled** by default.
     - Method: `POST`
     - URL: `={{ $('Settings').first().json.base }}/check`
     - Send Body: JSON (Key-Pair format) -> `username`: `={{ $('Settings').first().json.username }}`, `stage`: `2`, `answer`: `={{ $('Stage 2 answer').first().json.b2 }}`
     - Headers: `x-participant-key` = `={{ $('Settings').first().json.key }}`
     - Authentication: None
   - Connect `Stage 2 answer` to `Check stage 2`.
7. **Set up Stage 3 Branch:**
   - Create a `Set` node named `Stage 3 answer` (`n8n-nodes-base.set`) with assignment `b3` (string type, value `""`).
   - Create an `HTTP Request` node named `Check stage 3` (`n8n-nodes-base.httpRequest`), set to **Disabled** by default.
     - Method: `POST`
     - URL: `={{ $('Settings').first().json.base }}/check`
     - Send Body: JSON (Key-Pair format) -> `username`: `={{ $('Settings').first().json.username }}`, `stage`: `3`, `answer`: `={{ $('Stage 3 answer').first().json.b3 }}`
     - Headers: `x-participant-key` = `={{ $('Settings').first().json.key }}`
     - Authentication: None
   - Connect `Stage 3 answer` to `Check stage 3`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Read Challenge 05 | [Challenge 05 Overview](https://marketing.app.n8n.cloud/webhook/cwc/challenge-05) |
| Register for the challenge | [Registration Link](https://marketing.app.n8n.cloud/webhook/cwc/register) |
| Check account status | [Account Status](https://marketing.app.n8n.cloud/webhook/cwc/status) |
| Submit challenge results | [Submission Link](https://marketing.app.n8n.cloud/webhook/cwc/submit) |
| View leaderboard results | [Results Page](https://marketing.app.n8n.cloud/webhook/cwc/results) |
| Frequently Asked Questions | [FAQ Link](https://marketing.app.n8n.cloud/webhook/cwc/faq) |
| Learn and practise | [Practice Hub](https://marketing.app.n8n.cloud/webhook/cwc/learn) |
| Official documentation | [n8n Documentation](https://docs.n8n.io/) |