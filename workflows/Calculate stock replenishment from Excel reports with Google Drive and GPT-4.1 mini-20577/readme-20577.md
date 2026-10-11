Calculate stock replenishment from Excel reports with Google Drive and GPT-4.1 mini

https://n8nworkflows.xyz/workflows/calculate-stock-replenishment-from-excel-reports-with-google-drive-and-gpt-4-1-mini-20577


# Calculate stock replenishment from Excel reports with Google Drive and GPT-4.1 mini

### 1. Workflow Overview

This workflow automates inventory replenishment calculations by processing newly uploaded Excel inventory sheets from Google Drive. It extracts product data, leverages OpenAI (GPT-4.1 mini) to analyze sales trends and stock levels in batches, writes the resulting reorder quantities and justifications back into the data structure, and updates the original spreadsheet before archiving it in a designated output folder.

The logic is grouped into three distinct functional blocks:
- **1.1 Input Reception & Parsing:** Triggers upon new file creation in a target Google Drive folder, downloads the XLSX file, parses its contents, and chunks the dataset into manageable batches of 15 rows.
- **1.2 AI Processing & Mapping:** Iteratively loops through each batch, builds a compact JSON payload representing sales, purchases, and multi-location stock metrics, queries the OpenAI model for replenishment decisions, and maps the responses back to the original row schema.
- **1.3 Reassembly & Archiving:** Consolidates all processed batches, sorts them back into their original sequence, converts the dataset into an XLSX file, overwrites the original file on Google Drive, and relocates it to an output folder.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Parsing

##### Overview
This block monitors a designated Google Drive directory for incoming inventory reports, downloads the target file, extracts tabular data from the XLSX format, and splits the rows into indexed arrays to prepare them for batch processing.

##### Nodes Involved
- `Trigger_XLSX_Uploaded_to_Drive`
- `Download XLSX`
- `Extract from XLSX`
- `Code: ChunkToBatches`

##### Node Details

- **Trigger_XLSX_Uploaded_to_Drive**
  - **Type & Technical Role:** Google Drive Trigger (`n8n-nodes-base.googleDriveTrigger`). Monitors a specific folder for newly created files.
  - **Configuration Choices:** Configured to trigger on the `fileCreated` event within a specific folder ID (`1zgN_2uUYjzYtJedS7okl9VqJzUyCtAvq`, named "Forbest AI"), polling every minute.
  - **Key Expressions:** None.
  - **Input / Output Connections:** Input: None (Trigger). Output: Connects to `Download XLSX`.
  - **Version Requirements:** Type version 1.
  - **Edge Cases & Failure Types:** Polling delays if Google Drive API rate limits are hit; ensure proper OAuth2 scopes (`drive` or `drive.readonly`).

- **Download XLSX**
  - **Type & Technical Role:** Google Drive (`n8n-nodes-base.googleDrive`). Downloads file binary data using the file ID from the trigger.
  - **Configuration Choices:** Operation set to `download`.
  - **Key Expressions:** File ID: `={{ $json.id }}`
  - **Input / Output Connections:** Input: `Trigger_XLSX_Uploaded_to_Drive`. Output: Connects to `Extract from XLSX`.
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Missing file ID or transient network errors during download.

- **Extract from XLSX**
  - **Type & Technical Role:** Extract from File (`n8n-nodes-base.extractFromFile`). Parses binary XLSX data into JSON objects representing rows.
  - **Configuration Choices:** Operation set to `xlsx`.
  - **Key Expressions:** None.
  - **Input / Output Connections:** Input: `Download XLSX`. Output: Connects to `Code: ChunkToBatches`.
  - **Version Requirements:** Type version 1.
  - **Edge Cases & Failure Types:** Corrupted binary stream or unexpected spreadsheet formatting/headers.

- **Code: ChunkToBatches**
  - **Type & Technical Role:** Code (`n8n-nodes-base.code`). Splits incoming row items into fixed-size chunks and assigns global index identifiers.
  - **Configuration Choices:** JavaScript execution mode (`runOnceForAllItems`). Batch size is configured to `15` items per chunk.
  - **Key Expressions:** Implements custom JavaScript logic mapping `_globalIndex` to every row item.
  - **Input / Output Connections:** Input: `Extract from XLSX`. Output: Connects to `Loop Over Items`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases & Failure Types:** Out-of-memory errors if processing excessively large spreadsheets without adequate batching.

---

#### Block 1.2: AI Processing & Mapping

##### Overview
This block iterates through each row batch, constructs a lightweight JSON payload containing essential inventory metrics per SKU, queries OpenAI’s GPT-4.1 mini model with structured system prompts, and parses the model response to map replenishment recommendations back onto the raw dataset.

##### Nodes Involved
- `Loop Over Items`
- `Code: BuildPrompt`
- `Message a model`
- `Code: MapBack`

##### Node Details

- **Loop Over Items**
  - **Type & Technical Role:** Split In Batches (`n8n-nodes-base.splitInBatches`). Controls iteration over the batch arrays generated in Block 1.1.
  - **Configuration Choices:** Standard loop options.
  - **Key Expressions:** None.
  - **Input / Output Connections:** Input: `Code: ChunkToBatches` (and `Code: MapBack` for loop continuation). Output: Branch 0 connects to `Code: Reassemble` (when loop finishes); Branch 1 connects to `Code: BuildPrompt`.
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Infinite loops if batch indexing or loop reset logic becomes misaligned.

- **Code: BuildPrompt**
  - **Type & Technical Role:** Code (`n8n-nodes-base.code`). Transforms raw row objects into a slimmed-down JSON representation tailored for the LLM.
  - **Configuration Choices:** Mode set to `runOnceForEachItem`.
  - **Key Expressions:** JavaScript mapping function `slim(r)` extracting columns (`KOD`, sales amounts, purchase amounts, warehouse stocks, and multi-store metrics like `SRN`, `BZK`, `HTY`, etc.).
  - **Input / Output Connections:** Input: `Loop Over Items`. Output: Connects to `Message a model`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases & Failure Types:** Missing column keys in the spreadsheet resulting in `NaN` or `0` values passed to the model.

- **Message a model**
  - **Type & Technical Role:** OpenAI LangChain Node (`@n8n/n8n-nodes-langchain.openAi`). Interfaces with OpenAI to generate replenishment recommendations.
  - **Configuration Choices:** Model ID: `gpt-4.1-mini`. Retry on fail enabled with a 5000ms wait between tries. Always output data enabled.
  - **Key Expressions:** 
    - System Prompt: Defines role as a Turkish-speaking inventory analyst for retail brand `"MARKA ADI"`, instructing the model to output a strict JSON array containing integer `need` and single-sentence `reason` attributes.
    - User Prompt: `={{ $json.prompt }}`
  - **Input / Output Connections:** Input: `Code: BuildPrompt`. Output: Connects to `Code: MapBack`.
  - **Version Requirements:** Type version 1.8.
  - **Edge Cases & Failure Types:** API rate limits, authentication failures, or malformed JSON responses from the LLM.

- **Code: MapBack**
  - **Type & Technical Role:** Code (`n8n-nodes-base.code`). Parses OpenAI text responses, strips markdown wrappers if present, and merges AI recommendations back into the original row structures.
  - **Configuration Choices:** JavaScript execution mode (`runOnceForAllItems`). Adds columns `AB` (need) and `AC` (reason).
  - **Key Expressions:** Accesses upstream data via `$items("Code: BuildPrompt", 0)`.
  - **Input / Output Connections:** Input: `Message a model`. Output: Connects back to `Loop Over Items`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases & Failure Types:** JSON parsing exceptions if the model output contains conversational text outside the requested array format.

---

#### Block 1.3: Reassembly & Archiving

##### Overview
This block aggregates all processed batches once the loop completes, sorts the dataset back into its original global index sequence, converts the data structure back into an XLSX file binary stream, updates the original Google Drive file content, and moves the completed spreadsheet to a designated destination folder.

##### Nodes Involved
- `Code: Reassemble`
- `Convert to XLSX`
- `Update Sheet`
- `Move To New Folder`

##### Node Details

- **Code: Reassemble**
  - **Type & Technical Role:** Code (`n8n-nodes-base.code`). Combines all processed loop items into a single flat array and sorts them by `_globalIndex`.
  - **Configuration Choices:** JavaScript execution mode.
  - **Key Expressions:** `all.sort((a,b)=>a._globalIndex - b._globalIndex)`
  - **Input / Output Connections:** Input: `Loop Over Items`. Output: Connects to `Convert to XLSX`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases & Failure Types:** Memory constraints if processing excessively large row counts simultaneously.

- **Convert to XLSX**
  - **Type & Technical Role:** Convert to File (`n8n-nodes-base.convertToFile`). Serializes the flat JSON row array back into binary XLSX format.
  - **Configuration Choices:** Operation set to `xlsx`.
  - **Key Expressions:** None.
  - **Input / Output Connections:** Input: `Code: Reassemble`. Output: Connects to `Update Sheet`.
  - **Version Requirements:** Type version 1.1.
  - **Edge Cases & Failure Types:** Schema discrepancies or invalid data types causing serialization errors.

- **Update Sheet**
  - **Type & Technical Role:** Google Drive (`n8n-nodes-base.googleDrive`). Overwrites the content of the original file in Google Drive.
  - **Configuration Choices:** Operation set to `update`, with `changeFileContent` enabled.
  - **Key Expressions:** File ID: `={{$node["Trigger_XLSX_Uploaded_to_Drive"].json.id}}`
  - **Input / Output Connections:** Input: `Convert to XLSX`. Output: Connects to `Move To New Folder`.
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Insufficient write permissions on the source file.

- **Move To New Folder**
  - **Type & Technical Role:** Google Drive (`n8n-nodes-base.googleDrive`). Relocates the updated file from the watched directory to an output folder.
  - **Configuration Choices:** Operation set to `move`. Drive ID set to `My Drive`. Folder ID set to `1u_4lwSfysQJ0_1KG6wP1nHi64rRv6BCy` (named "Acarlar n8n Upload").
  - **Key Expressions:** File ID: `={{ $json.id }}`
  - **Input / Output Connections:** Input: `Update Sheet`. Output: None (Terminal node).
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Invalid destination folder ID or lack of permissions on the target directory.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Trigger_XLSX_Uploaded_to_Drive` | `n8n-nodes-base.googleDriveTrigger` | Watches Drive folder for new XLSX files | None | `Download XLSX` | ## 1. Get the spreadsheet<br>Watches a Drive folder, downloads the new XLSX, parses it and splits the rows into batches. |
| `Download XLSX` | `n8n-nodes-base.googleDrive` | Downloads the binary file from Drive | `Trigger_XLSX_Uploaded_to_Drive` | `Extract from XLSX` | ## 1. Get the spreadsheet<br>Watches a Drive folder, downloads the new XLSX, parses it and splits the rows into batches. |
| `Extract from XLSX` | `n8n-nodes-base.extractFromFile` | Parses XLSX binary data into JSON rows | `Download XLSX` | `Code: ChunkToBatches` | ## 1. Get the spreadsheet<br>Watches a Drive folder, downloads the new XLSX, parses it and splits the rows into batches. |
| `Code: ChunkToBatches` | `n8n-nodes-base.code` | Splits rows into batches and assigns global index | `Extract from XLSX` | `Loop Over Items` | ## 1. Get the spreadsheet<br>Watches a Drive folder, downloads the new XLSX, parses it and splits the rows into batches. |
| `Loop Over Items` | `n8n-nodes-base.splitInBatches` | Iterates over row batches | `Code: ChunkToBatches`, `Code: MapBack` | `Code: Reassemble`, `Code: BuildPrompt` | ## 2. AI stock analysis<br>Builds a compact product list per batch, asks GPT-4.1 mini for reorder quantity and reason, then maps answers back to rows. |
| `Code: BuildPrompt` | `n8n-nodes-base.code` | Slims down row data into a compact JSON prompt payload | `Loop Over Items` | `Message a model` | ## 2. AI stock analysis<br>Builds a compact product list per batch, asks GPT-4.1 mini for reorder quantity and reason, then maps answers back to rows. |
| `Message a model` | `@n8n/n8n-nodes-langchain.openAi` | Queries GPT-4.1 mini for reorder quantities and reasons | `Code: BuildPrompt` | `Code: MapBack` | ## 2. AI stock analysis<br>Builds a compact product list per batch, asks GPT-4.1 mini for reorder quantity and reason, then maps answers back to rows. |
| `Code: MapBack` | `n8n-nodes-base.code` | Merges AI results into row objects as AB and AC columns | `Message a model` | `Loop Over Items` | ## 2. AI stock analysis<br>Builds a compact product list per batch, asks GPT-4.1 mini for reorder quantity and reason, then maps answers back to rows. |
| `Code: Reassemble` | `n8n-nodes-base.code` | Flattens and sorts processed batches by global index | `Loop Over Items` | `Convert to XLSX` | ## 3. Save the result<br>Reassembles rows in order, rebuilds the XLSX, updates the original file and moves it to the output folder. |
| `Convert to XLSX` | `n8n-nodes-base.convertToFile` | Converts JSON row array back into binary XLSX format | `Code: Reassemble` | `Update Sheet` | ## 3. Save the result<br>Reassembles rows in order, rebuilds the XLSX, updates the original file and moves it to the output folder. |
| `Update Sheet` | `n8n-nodes-base.googleDrive` | Overwrites original file content on Google Drive | `Convert to XLSX` | `Move To New Folder` | ## 3. Save the result<br>Reassembles rows in order, rebuilds the XLSX, updates the original file and moves it to the output folder. |
| `Move To New Folder` | `n8n-nodes-base.googleDrive` | Moves updated spreadsheet to output folder | `Update Sheet` | None | ## 3. Save the result<br>Reassembles rows in order, rebuilds the XLSX, updates the original file and moves it to the output folder. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a **Google Drive Trigger** node (`Trigger_XLSX_Uploaded_to_Drive`). Set the event to `fileCreated`, trigger on `specificFolder`, select your watch folder (e.g., ID `1zgN_2uUYjzYtJedS7okl9VqJzUyCtAvq`), and set polling to every minute. Configure your Google Drive OAuth2 credential.
2. **Add File Download:** Create a **Google Drive** node (`Download XLSX`), set operation to `download`, and map the file ID using expression `={{ $json.id }}`. Connect it to the trigger.
3. **Parse Spreadsheet:** Add an **Extract from File** node (`Extract from XLSX`), set operation to `xlsx`, and connect it after `Download XLSX`.
4. **Chunk Dataset:** Add a **Code** node (`Code: ChunkToBatches`). Paste the chunking script establishing a batch size of 15 and mapping `_globalIndex` values. Connect it after `Extract from XLSX`.
5. **Set Up Loop:** Add a **Split In Batches** node (`Loop Over Items`). Connect its input to `Code: ChunkToBatches`.
6. **Build Prompt Payload:** Add a **Code** node (`Code: BuildPrompt`) set to run once for each item. Paste the `slim(r)` script extracting SKU codes, sales/purchase metrics, warehouse stocks, and store inventories. Connect output 1 of `Loop Over Items` to this node.
7. **Configure AI Model:** Add an **OpenAI** node (`Message a model`). Set the model to `gpt-4.1-mini`. Configure the System Message with the inventory analyst prompt and replace `MARKA ADI` with your brand name. Set the User Message to `={{ $json.prompt }}`. Enable retry on fail. Connect it after `Code: BuildPrompt`. Add your OpenAI API credential.
8. **Map AI Responses:** Add a **Code** node (`Code: MapBack`). Paste the mapping script that parses the JSON response from OpenAI and writes `need` into column `AB` and `reason` into column `AC`. Connect it after `Message a model` and route its output back into `Loop Over Items`.
9. **Reassemble Rows:** Add a **Code** node (`Code: Reassemble`). Paste the sorting script using `_globalIndex`. Connect output 2 (loop completion branch) of `Loop Over Items` to this node.
10. **Convert to Spreadsheet:** Add a **Convert to File** node (`Convert to XLSX`), set operation to `xlsx`, and connect it after `Code: Reassemble`.
11. **Update Drive File:** Add a **Google Drive** node (`Update Sheet`), set operation to `update`, enable `changeFileContent`, and set the file ID expression to `={{$node["Trigger_XLSX_Uploaded_to_Drive"].json.id}}`. Connect it after `Convert to XLSX` using your Google Drive credential.
12. **Archive File:** Add a **Google Drive** node (`Move To New Folder`), set operation to `move`, configure the destination folder ID (e.g., `1u_4lwSfysQJ0_1KG6wP1nHi64rRv6BCy`), and map file ID to `={{ $json.id }}`. Connect it as the terminal node after `Update Sheet`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Cloud Console Setup | [Google Cloud Console](https://cloud.google.com) — Create an OAuth Client ID and Client Secret for Google Drive integration. |
| Column Name Dependency | The JavaScript code in `Code: BuildPrompt` and the AI system prompt rely on specific inventory spreadsheet column headers (e.g., `KOD`, `Son 45 Satis Miktar`, `DEPO`, store codes). Adapt these strings if your source spreadsheets use different naming conventions. |