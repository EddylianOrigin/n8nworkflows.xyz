Deduplicate and send server-side conversions to Meta Conversions API

https://n8nworkflows.xyz/workflows/deduplicate-and-send-server-side-conversions-to-meta-conversions-api-17674


# Deduplicate and send server-side conversions to Meta Conversions API

### 1. Workflow Overview

This workflow is designed to ingest server-side conversion events (such as qualified leads, purchases, or bookings) via a webhook, sanitize and normalize user data, apply SHA-256 hashing to private identifiers, and prevent duplicate processing via an n8n Data Table before successfully dispatching the payload to the Meta Conversions API (Graph API). Finally, it inspects the response to confirm that Meta has validated and accepted the event.

The logic is systematically grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingestion of incoming raw JSON webhook payloads and establishment of core metadata constants (Pixel ID, API version, and tracking parameters).
- **1.2 Parsing & Normalization:** Sanitization of fields, conversion of event timestamps into Unix epoch format (with safety clamps for stale or future events), repair of Meta click IDs (`fbc`), and computation of a deterministic FNV-1a deduplication key.
- **1.3 Deduplication Control:** Querying an n8n Data Table to identify and drop duplicate events before storage and recording new unique events to prevent race-condition loops.
- **1.4 Cryptographic Hashing:** Sequential SHA-256 transformation of sensitive user attributes (email, phone, first name, last name, city, zip, and country) while keeping public parameters (IP addresses, user agents, and external IDs) unhashed.
- **1.5 Payload Assembly & Transmission:** Compilation of the final structured schema expected by the Meta Graph API endpoint, execution of the HTTP POST request using Query Auth, and response validation to ensure receipt confirmation.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

- **Overview:** This block captures incoming tracking requests through an HTTP webhook endpoint and injects configuration parameters required to communicate with Meta's Graph API and the local tracking state tables.
- **Nodes Involved:** 
  - `Conversion Webhook`
  - `Settings`

- **Node Details:**
  - **Conversion Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration Choices:* Listens for incoming `POST` requests on the route path `meta-capi-conversion`.
    - *Key Expressions or Variables:* Operates on the path `/webhook/meta-capi-conversion`.
    - *Input and Output Connections:* Input: None (Trigger); Output: Connects to `Settings`.
    - *Version-Specific Requirements:* Version 2.1.
    - *Edge Cases or Potential Failure Types:* Malformed JSON payloads or network drops before execution trigger failure. Ensure external callers use the `POST` method.
  
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Constant Injector)
    - *Configuration Choices:* Manually sets configuration variables including Meta Pixel ID (`1234567890123456`), Meta API version (`v22.0`), default source URL (`https://example.com`), and the tracking table name (`Meta CAPI sent events`).
    - *Key Expressions or Variables:* Assigns static variables accessible downstream via `$('Settings').first().json`.
    - *Input and Output Connections:* Input: `Conversion Webhook`; Output: Connects to `Parse and Normalize`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases or Potential Failure Types:* Ensure configuration values match your active Meta business asset settings.

---

#### Block 1.2: Parsing & Normalization

- **Overview:** This block extracts incoming parameters, normalizes user data according to Meta rules, repairs bare `fbclid` strings into valid `fbc` formats, handles timestamp clamping, and generates a unique deduplication hash.
- **Nodes Involved:** 
  - `Parse and Normalize`

- **Node Details:**
  - **Parse and Normalize**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration Choices:* Executes once for all items using a custom JavaScript function that implements string sanitization, FNV-1a hashing for deduplication, and date validation rules.
    - *Key Expressions or Variables:* Reads configurations from `$('Settings')` and incoming data from `$input.first().json`.
    - *Input and Output Connections:* Input: `Settings`; Output: Connects to `Skip If Already Sent`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Throws a JavaScript error if `event_name` is missing or if no valid user identifiers are present. Future-dated timestamps are safely clamped to the current server time, and events older than 6.5 days are clamped to Meta’s 7-day ingestion limit.

---

#### Block 1.3: Deduplication Control

- **Overview:** This block guards against duplicate execution by checking an n8n Data Table to see if the computed deduplication key already exists, halting processing if found, or recording the new key to prevent replay attacks.
- **Nodes Involved:** 
  - `Skip If Already Sent`
  - `Record Event Key`

- **Node Details:**
  - **Skip If Already Sent**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Database Query/Filter)
    - *Configuration Choices:* Operates on rows, executing an existence check (`rowNotExists`) comparing the table column `dedupe_key` with `{{ $json.dedupeKey }}`.
    - *Key Expressions or Variables:* `={{ $json.dedupeKey }}` and Data Table name derived from `={{ $('Settings').first().json.dedupeTableName }}`.
    - *Input and Output Connections:* Input: `Parse and Normalize`; Output: Connects to `Record Event Key`.
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases or Potential Failure Types:* Fails if the underlying n8n Data Table has not been created or lacks the `dedupe_key` schema column.
  
  - **Record Event Key**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Database Insertion)
    - *Configuration Choices:* Inserts a new row containing `dedupe_key`, `event_name`, `order_id`, and `sent_at`.
    - *Key Expressions or Variables:* Populates fields using `$now.toISO()` and `$json` parameters.
    - *Input and Output Connections:* Input: `Skip If Already Sent`; Output: Connects to `Hash Email`.
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases or Potential Failure Types:* Database write constraints or permission restrictions will disrupt execution.

---

#### Block 1.4: Cryptographic Hashing

- **Overview:** This block applies sequential SHA-256 encryption to individual personal identification attributes to comply with Meta privacy guidelines. Empty or unset attributes are skipped safely.
- **Nodes Involved:** 
  - `Hash Email`
  - `Hash Phone`
  - `Hash First Name`
  - `Hash Last Name`
  - `Hash City`
  - `Hash Zip`
  - `Hash Country`

- **Node Details:**
  - **Hash Email**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedEmail`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedEmail || 'unset' }}`
    - *Input and Output Connections:* Input: `Record Event Key`; Output: Connects to `Hash Phone`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Fails if input is malformed.
  
  - **Hash Phone**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedPhone`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedPhone || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash Email`; Output: Connects to `Hash First Name`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Hash First Name**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedFirstName`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedFirstName || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash Phone`; Output: Connects to `Hash Last Name`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Hash Last Name**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedLastName`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedLastName || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash First Name`; Output: Connects to `Hash City`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Hash City**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedCity`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedCity || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash Last Name`; Output: Connects to `Hash Zip`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Hash Zip**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedZip`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedZip || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash City`; Output: Connects to `Hash Country`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Hash Country**
    - *Type and Technical Role:* `n8n-nodes-base.crypto` (Cryptographic Transformation)
    - *Configuration Choices:* Action: Hash, Algorithm: SHA256, Encoding: Hex, Target Property: `hashedCountry`.
    - *Key Expressions or Variables:* `={{ $('Parse and Normalize').item.json.normalizedCountry || 'unset' }}`
    - *Input and Output Connections:* Input: `Hash Zip`; Output: Connects to `Build Meta Payload`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* None.

---

#### Block 1.5: Payload Assembly & Transmission

- **Overview:** This final functional block structures the hashed and normalized attributes into the required Meta payload wrapper, posts it securely to the Graph API, and validates the response body to confirm event acceptance.
- **Nodes Involved:** 
  - `Build Meta Payload`
  - `Send Event to Meta CAPI`
  - `Validate Meta Response`

- **Node Details:**
  - **Build Meta Payload**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration Choices:* Consolidates outputs from the parsing and cryptographic hashing nodes into an array object nested under `data`.
    - *Key Expressions or Variables:* Merges references from `$('Parse and Normalize')` and the cryptographic hash stream.
    - *Input and Output Connections:* Input: `Hash Country`; Output: Connects to `Send Event to Meta CAPI`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Throws an error if no usable identifiers (`fbc`, `fbp`, `em`, `ph`, or `external_id`) are present.
  
  - **Send Event to Meta CAPI**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request Dispatcher)
    - *Configuration Choices:* Method: `POST`, URL: `https://graph.facebook.com/{{ $('Settings').first().json.metaApiVersion }}/{{ $('Settings').first().json.metaPixelId }}/events`, Content-Type: `Raw` (`application/json`), Authentication: Generic Credential Type (HTTP Query Auth).
    - *Key Expressions or Variables:* Dynamic interpolation of API version, Pixel ID, and serialized request body `={{ JSON.stringify($json.requestBody) }}`.
    - *Input and Output Connections:* Input: `Build Meta Payload`; Output: Connects to `Validate Meta Response`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Meta responses default to HTTP 200 even for dropped events; configuration requires `neverError: true` and `fullResponse: true` to pass execution safely to the validator node. Requires a configured `access_token` query credential.
  
  - **Validate Meta Response**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Validator)
    - *Configuration Choices:* Inspects JSON response attributes for API errors or empty `events_received` counts.
    - *Key Expressions or Variables:* Evaluates `$input.first().json.body`.
    - *Input and Output Connections:* Input: `Send Event to Meta CAPI`; Output: None (Terminal node).
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Throws an explicit error if Meta returns error objects or if `events_received` is less than 1.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note a5aa8e` | `n8n-nodes-base.stickyNote` | Documentation & Overview | None | None | Send server-side conversions to the Meta Conversions API<br><br>Post any conversion — a qualified lead, a purchase, a booking — to this webhook and it is normalised, hashed and forwarded to Meta with the identifiers Meta actually accepts. Works with any CRM, backend or form: the webhook takes plain JSON.<br><br>### How it works<br>1. **Parse and Normalize** applies Meta's rules to every identifier you send and builds a valid `fbc`. Meta expects `fb.1.<timestamp>.<fbclid>`; most senders pass the bare `fbclid`, which Meta silently ignores.<br>2. A Data Table records every event key, so retries and duplicate CRM webhooks are dropped before they reach Meta. An `event_id` is sent as a second guard.<br>3. Seven **Crypto** nodes hash email, phone, name, city, zip and country with SHA-256. Doing it here avoids `require('crypto')`, which is unavailable inside n8n Code nodes. `external_id`, IP and user agent go unhashed, as Meta expects. Anything you do not send is simply skipped.<br>4. The payload is posted to the Graph API and the response is checked — Meta answers HTTP 200 even when it rejects an event.<br><br>### Setup<br>1. Create a Data Table with a `dedupe_key` text column and put its name in **Settings**.<br>2. Fill in your pixel (dataset) ID and the API version in **Settings**.<br>3. Add your Meta access token as a **Query Auth** credential named `access_token` and attach it to the send node. Never paste the token into the URL.<br>4. Point your CRM or backend at the production webhook URL.<br><br>### Customization tips<br>- Send `test_event_code` in the payload to route events to Meta's Test Events tool.<br>- The output reports which identifiers were attached, so you can measure coverage. |
| `Sticky Note c9d72d` | `n8n-nodes-base.stickyNote` | Documentation Block 1 | None | None | ## 1. Receive and normalise<br>Accepts plain JSON from any source and repairs the fbc format. |
| `Sticky Note 6e17f3` | `n8n-nodes-base.stickyNote` | Documentation Block 2 | None | None | ## 2. Drop duplicates<br>A Data Table key stops repeat deliveries before they reach Meta. |
| `Sticky Note 8cc3c1` | `n8n-nodes-base.stickyNote` | Documentation Block 3 | None | None | ## 3. Hash identifiers<br>SHA-256 via Crypto nodes. Empty values are skipped, never sent as a hash of nothing. |
| `Sticky Note 31ff69` | `n8n-nodes-base.stickyNote` | Documentation Block 4 | None | None | ## 4. Send and verify<br>Posts to the Graph API and checks the response body. |
| `Sticky Note 3acc65` | `n8n-nodes-base.stickyNote` | Warning Documentation | None | None | ## Do not send the wrong IP<br><br>If your CRM calls this webhook server to server, the request headers belong to your CRM, not to the customer.<br><br>Never map `client_ip_address` or `client_user_agent` from the incoming headers — it lowers match quality instead of raising it. Send them in the payload only if you captured them in the browser. |
| `Conversion Webhook` | `n8n-nodes-base.webhook` | Ingests conversion data via HTTP POST | None | `Settings` | |
| `Settings` | `n8n-nodes-base.set` | Injects global configuration variables | `Conversion Webhook` | `Parse and Normalize` | |
| `Parse and Normalize` | `n8n-nodes-base.code` | Normalizes payload fields and computes deduplication key | `Settings` | `Skip If Already Sent` | |
| `Skip If Already Sent` | `n8n-nodes-base.dataTable` | Filters duplicate events using Data Table lookups | `Parse and Normalize` | `Record Event Key` | |
| `Record Event Key` | `n8n-nodes-base.dataTable` | Records processed deduplication keys | `Skip If Already Sent` | `Hash Email` | |
| `Hash Email` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for email attributes | `Record Event Key` | `Hash Phone` | |
| `Hash Phone` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for phone attributes | `Hash Email` | `Hash First Name` | |
| `Hash First Name` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for first name attributes | `Hash Phone` | `Hash Last Name` | |
| `Hash Last Name` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for last name attributes | `Hash First Name` | `Hash City` | |
| `Hash City` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for city attributes | `Hash Last Name` | `Hash Zip` | |
| `Hash Zip` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for zip attributes | `Hash City` | `Hash Country` | |
| `Hash Country` | `n8n-nodes-base.crypto` | Computes SHA-256 hash for country attributes | `Hash Zip` | `Build Meta Payload` | |
| `Build Meta Payload` | `n8n-nodes-base.code` | Compiles validated user identifiers and event parameters into Meta Graph API schema | `Hash Country` | `Send Event to Meta CAPI` | |
| `Send Event to Meta CAPI` | `n8n-nodes-base.httpRequest` | Posts conversion payload securely to Meta Graph API | `Build Meta Payload` | `Validate Meta Response` | |
| `Validate Meta Response` | `n8n-nodes-base.code` | Validates API response body to confirm event receipt | `Send Event to Meta CAPI` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually inside n8n:

1. **Create the Data Table:**
   - In your n8n instance, create a new Data Table named `Meta CAPI sent events`.
   - Add a text column with the exact name `dedupe_key`. Additional recommended columns include `event_name`, `order_id`, and `sent_at`.

2. **Set up Node 1 (`Conversion Webhook`):**
   - Add a **Webhook** trigger node.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `meta-capi-conversion`.

3. **Set up Node 2 (`Settings`):**
   - Add an **Edit Fields (Set)** node connected downstream from the Webhook.
   - Configure assignments to include string variables:
     - `metaPixelId`: `1234567890123456` (replace with your active Pixel/Dataset ID)
     - `metaApiVersion`: `v22.0`
     - `defaultEventSourceUrl`: `https://example.com`
     - `dedupeTableName`: `Meta CAPI sent events`

4. **Set up Node 3 (`Parse and Normalize`):**
   - Add a **Code** node.
   - Set mode to **Run Once for All Items**.
   - Paste the normalization script parsing incoming payloads, cleaning identifiers, creating safe timestamps, and calculating the FNV-1a deduplication hash (`dedupeKey`).

5. **Set up Node 4 (`Skip If Already Sent`):**
   - Add a **Data Table** node.
   - Set **Resource** to `Row` and **Operation** to `Row Does Not Exist`.
   - Link the Data Table ID lookup to `={{ $('Settings').first().json.dedupeTableName }}`.
   - Add a condition where table column `dedupe_key` equals `={{ $json.dedupeKey }}`.

6. **Set up Node 5 (`Record Event Key`):**
   - Add a **Data Table** node.
   - Set **Resource** to `Row` and **Operation** to `Insert`.
   - Map columns to `dedupe_key` (`={{ $json.dedupeKey }}`), `event_name` (`={{ $json.eventName }}`), `order_id` (`={{ $json.orderId }}`), and `sent_at` (`={{ $now.toISO() }}`).

7. **Set up Nodes 6–12 (`Crypto Nodes` for Hashing):**
   - Create seven sequential **Crypto** nodes named respectively: `Hash Email`, `Hash Phone`, `Hash First Name`, `Hash Last Name`, `Hash City`, `Hash Zip`, and `Hash Country`.
   - For each node:
     - Set **Action** to `Hash`.
     - Set **Type** to `SHA256`.
     - Set **Encoding** to `Hex`.
     - Set the target data property name to match (`hashedEmail`, `hashedPhone`, etc.).
     - Map the input value expression to pull from `$('Parse and Normalize').item.json.normalized[Field] || 'unset'`.
   - Chain them together sequentially from Email down to Country.

8. **Set up Node 13 (`Build Meta Payload`):**
   - Add a **Code** node.
   - Set mode to **Run Once for All Items**.
   - Paste the aggregation logic mapping parsed properties and processed cryptographic hash values into the Graph API `data` array format.

9. **Set up Node 14 (`Send Event to Meta CAPI`):**
   - Add an **HTTP Request** node.
   - Set **Method** to `POST`.
   - Set **URL** to `=https://graph.facebook.com/{{ $('Settings').first().json.metaApiVersion }}/{{ $('Settings').first().json.metaPixelId }}/events`.
   - Set **Authentication** to `Generic Credential Type` -> `HTTP Query Auth`. Create or assign a credential parameter where the query parameter key is named `access_token` and populated with your Meta system user access token.
   - Set **Body Content Type** to `Raw` / `Application/Json`.
   - Set **Body** value to `={{ JSON.stringify($json.requestBody) }}`.
   - Under node options, enable **Never Error** and **Full Response** under response settings.

10. **Set up Node 15 (`Validate Meta Response`):**
    - Add a **Code** node.
    - Set mode to **Run Once for All Items**.
    - Paste validation logic to inspect response error payloads and verify that `events_received` is greater than or equal to 1.

11. **Finalize Connections:**
    - Wire the nodes sequentially in the exact topological order specified in Section 2 and Section 3.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Server-side IP and User Agent Warning | Never map client IP addresses or user agents directly from server-to-server webhook request headers unless captured natively in the end-user's browser. Doing so degrades Meta match quality. |
| Meta Conversions API Documentation | Refer to the official Meta Graph API specifications for payload standards and parameter requirements: [Meta Conversions API Reference](https://developers.facebook.com/docs/marketing-api/conversions-api) |