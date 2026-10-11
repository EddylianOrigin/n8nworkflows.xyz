Plan and recheck trips with Gemini, Gmail, Open-Meteo, Tavily and Data Tables

https://n8nworkflows.xyz/workflows/plan-and-recheck-trips-with-gemini--gmail--open-meteo--tavily-and-data-tables-20589


# Plan and recheck trips with Gemini, Gmail, Open-Meteo, Tavily and Data Tables

### 1. Workflow Overview

This workflow is an advanced, multilingual AI travel planning and pre-trip monitoring system. It collects traveler preferences, validates trip feasibility through real-world web research, generates personalized day-by-day itineraries, stores records in n8n Data Tables, delivers itineraries via email and interactive web forms, and executes automated pre-departure rechecks.

The logical execution is grouped into two main functional pathways:
- **Interactive Trip-Planning Flow:** Collects user locale, travel style, interests, food/stay preferences, and special needs via multi-step forms. It gathers trip basics and specific preferences, assesses real-world feasibility using AI and external APIs (Tavily, Google Maps Routes), generates a customized itinerary, saves it to data storage, and delivers the plan.
- **Scheduled Pre-Trip Recheck Flow:** Runs daily via a schedule trigger to inspect upcoming trips, performs targeted multi-source checks (Open-Meteo weather, Tavily transport/traffic/facility rechecks, and Gemini analysis), evaluates whether changes affect the itinerary, rebuilds the itinerary when necessary, updates data records, and sends notification emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Interactive Trip-Planning Flow

##### 1.1 Input Reception
- **Overview:** Initializes the form trigger where users select their preferred language and captures the initial submission timestamp.
- **Nodes Involved:** `AI Travel Planner`, `Normalize Locale`
- **Node Details:**
  - `AI Travel Planner`: n8n Forms trigger node. Renders a dropdown allowing users to select one of nine supported languages (English, Japanese, Spanish, French, German, Italian, Portuguese, Korean, Chinese). Outputs submission metadata and selected language.
  - `Normalize Locale`: Code node (JavaScript). Maps the selected display language to standard ISO language (`en`, `ja`, etc.) and locale codes (`en-US`, `ja-JP`, etc.), appending session start attributes.
  - **Inputs:** External user trigger (Form webhooks).
  - **Outputs:** Connected to `Travel Profile Start`.
  - **Edge Cases:** Missing or unsupported language selection throws an explicit JavaScript error.

##### 1.2 Traveler Profile & Preferences Collection
- **Overview:** Collects the traveler’s pace, travel style, budget approach, interests, food/stay preferences, and special considerations across multi-step localized n8n forms with built-in validation and correction loops.
- **Nodes Involved:** `Travel Profile Start`, `Normalize Travel Style`, `Travel Interests`, `Normalize Travel Interests`, `Travel Interests Valid?`, `Travel Interests Correction`, `Normalize Corrected Travel Interests`, `Prepare Profile Preferences`, `Food & Stay Preferences`, `Normalize Food & Stay Preferences`, `Food & Stay Preferences Valid?`, `Food & Stay Preferences Correction`, `Normalize Corrected Food & Stay Preferences`, `Prepare Food & Stay Preferences`, `Special Considerations`, `Normalize Special Considerations`, `Special Considerations Valid?`, `Special Considerations Correction`, `Normalize Corrected Special Considerations`, `Prepare Travel Profile`, `Prepare Travel Profile Storage`, `Save Travel Profile`
- **Node Details:**
  - `Travel Profile Start`, `Travel Interests`, `Food & Stay Preferences`, `Special Considerations`: Dynamic n8n Form nodes. Render localized form fields (radio buttons and checkboxes) determined by expressions referencing the user's preferred language.
  - `Normalize Travel Style`, `Normalize Travel Interests`, `Normalize Food & Stay Preferences`, `Normalize Special Considerations`: Code nodes. Translate localized multi-language form options into standardized internal database keys (e.g., translating Japanese/French/Spanish terms to canonical strings like `history_culture`, `relaxed`, `budget`).
  - `Travel Interests Valid?`, `Food & Stay Preferences Valid?`, `Special Considerations Valid?`: IF condition nodes. Verify whether exclusivity rules are met (e.g., ensuring "Let AI decide" or "No special considerations" checkboxes are selected exclusively without conflicting selections).
  - `Travel Interests Correction`, `Food & Stay Preferences Correction`, `Special Considerations Correction`: Corrective dynamic form nodes presented to users when validation fails.
  - `Prepare Profile Preferences`, `Prepare Food & Stay Preferences`, `Prepare Travel Profile`, `Prepare Travel Profile Storage`: Code nodes that consolidate, shape, and JSON-stringify multi-category profile preferences before database storage.
  - `Save Travel Profile`: n8n Data Table node (`travel_profiles`). Upserts the normalized traveler profile using `profile_key` as the match column.
  - **Inputs:** Prior node outputs.
  - **Outputs:** Routes to `Trip Basics`.
  - **Edge Cases:** Multiple incompatible selections trigger validation rules leading to correction forms. Database upsert failure halts the workflow.

##### 1.3 Trip Requirements & Constraints
- **Overview:** Collects core trip logistics (origin, destination, start date, end date, purpose) and trip-specific preferences (companions, transport mode, budget, accommodation status, must-visit places, and priorities), validating the date range and completeness.
- **Nodes Involved:** `Trip Basics`, `Normalize Trip Basics`, `Trip Basics Valid?`, `Trip Preferences`, `Normalize Trip Preferences`, `Prepare Full Trip Request`
- **Node Details:**
  - `Trip Basics`: Dynamic form node capturing start point, destination text, start/end dates, and primary purpose with localization support.
  - `Normalize Trip Basics`: Code node. Splits multi-line destinations, calculates total trip duration (`trip_days`), and validates required inputs (dates, origin, destination).
  - `Trip Basics Valid?`: IF condition node ensuring `trip_basics_valid` is true.
  - `Trip Preferences`: Dynamic form node capturing companion type, transport preferences, budget details, accommodation status, preferred stay area, must-visit notes, and food/stay priority.
  - `Normalize Trip Preferences`: Code node. Normalizes international text answers for companions, transport modes, and priorities into canonical keys.
  - `Prepare Full Trip Request`: Code node. Bundles trip basics, trip preferences, and context into a standardized payload structure.
  - **Inputs:** `Save Travel Profile` output and `Trip Basics Valid?` branch.
  - **Outputs:** Routes to `Prepare Feasibility Request`.
  - **Edge Cases:** Invalid date ranges (`trip_days < 1`) or missing locations throw validation errors.

---

#### 2.2 Feasibility Research & Evidence Validation

##### 2.1 Feasibility Requirements Analysis
- **Overview:** Uses Google Gemini to analyze the trip request, traveler profile, and constraints to determine what real-world facts and transportation checks must be verified.
- **Nodes Involved:** `Prepare Feasibility Request`, `Analyze Feasibility Requirements`, `Gemini | Feasibility Analysis1`, `Normalize Feasibility Requirements`, `Prepare Feasibility Research`, `Hard Research Required?`
- **Node Details:**
  - `Prepare Feasibility Request`: Code node formatting structured JSON payloads for AI analysis.
  - `Analyze Feasibility Requirements`: LangChain Basic LLM Chain node using `Gemini | Feasibility Analysis1`. Returns a strictly structured JSON specifying route scope, complexity, and an array of specific verification checks (`road_route_and_duration`, `transit_route_and_duration`, `border_entry_requirements`, etc.).
  - `Gemini | Feasibility Analysis1`: Google Gemini Chat Model (`models/gemini-3.1-flash-lite`).
  - `Normalize Feasibility Requirements`: Code node. Cleans markdown fences, parses JSON, and validates structural schemas for route scopes and check types.
  - `Prepare Feasibility Research`: Code node. Separates checks into hard checks (transport, duration, border, timezone) and soft deferred checks (accommodation, food, general sightseeing).
  - `Hard Research Required?`: IF condition checking if external hard research is needed.
  - **Inputs:** `Prepare Full Trip Request`.
  - **Outputs:** Diverges to `Prepare Route Research`.
  - **Edge Cases:** Malformed JSON outputs from the LLM are caught and trigger recovery exceptions.

##### 2.2 Multi-Channel Research & Evidence Verification
- **Overview:** Executes parallel targeted research branches for public transportation, general destination conditions, and driving routes using Tavily web searches, Google Maps Routes API, and Google Gemini evidence evaluators.
- **Nodes Involved:** `Prepare Route Research`, `Transit Research Required?`, `Research Transit`, `Gemini | Transit Research`, `Normalize Transit Research`, `External Transit Research Required?`, `Prepare Transit Search Queries`, `Search Transit Web`, `Verify Transit Evidence`, `Gemini | Transit Verification`, `Normalize Transit Evidence`, `Additional Transit Research Required?`, `Prepare Additional Transit Research`, `Search Additional Transit Web`, `Verify Additional Transit Evidence`, `Gemini | Transit Evidence Verification`, `Aggregate Additional Transit Research`, `Normalize Final Transit Evidence`, `General Research Required?`, `Generate General Research Queries`, `Gemini | General Research Queries`, `Prepare General Research`, `Search General Research Web`, `Analyze General Research Evidence`, `Gemini | General Research Evidence`, `Aggregate General Research Evidence`, `Synthesize General Research`, `Gemini | General Research Synthesis`, `Normalize General Research Synthesis`, `Route API Research Required?`, `Prepare Route API Research`, `Get Driving Route`, `Normalize Route API Research`, `Route API Not Required`, `General Research Not Required`, `Transit Research Not Required`, `Merge Feasibility Research`, `Transit Research Result`, `General Research Result`, `Route API Research Result`, `Collect Feasibility Research`
- **Node Details:**
  - `Research Transit`, `Verify Transit Evidence`, `Verify Additional Transit Evidence`, `Generate General Research Queries`, `Analyze General Research Evidence`, `Synthesize General Research`: LangChain Chain Nodes powered by Google Gemini models (`models/gemini-3.1-flash-lite`) performing extraction, query generation, and evidence verification.
  - `Search Transit Web`, `Search Additional Transit Web`, `Search General Research Web`: Tavily tool nodes executing targeted web searches.
  - `Get Driving Route`: HTTP Request node querying the Google Maps Routes API (`https://routes.googleapis.com/directions/v2:computeRoutes`) with `X-Goog-FieldMask` header headers to retrieve driving distance and duration.
  - `Route API Not Required`, `General Research Not Required`, `Transit Research Not Required`: Set nodes providing fallback structures when categories are unneeded (`required: false`, `status: "not_required"`).
  - `Merge Feasibility Research`: Merge node combining results from transit, general, and route API research streams.
  - `Collect Feasibility Research`: Code node consolidating research statuses.
  - **Inputs:** `Hard Research Required?` and routing checks.
  - **Outputs:** Routes to `Prepare Feasibility Assessment`.
  - **Edge Cases:** API rate limits or search failures handled via fallback states and unverified flags.

##### 2.3 Final Feasibility Decision
- **Overview:** Evaluates all gathered evidence through an AI assessment engine to determine whether the trip is feasible and whether itinerary generation can proceed.
- **Nodes Involved:** `Prepare Feasibility Assessment`, `Assess Travel Feasibility`, `Gemini | Feasibility Assessment`, `Normalize Feasibility Assessment`, `Can Generate Itinerary?`
- **Node Details:**
  - `Prepare Feasibility Assessment`: Code node compiling research findings, later rechecks, and limitations.
  - `Assess Travel Feasibility`: LangChain Chain node using `Gemini | Feasibility Assessment` (`models/gemini-3.5-flash`). Returns an overall feasibility status (`feasible`, `feasible_with_conditions`, `difficult`, `infeasible`, `insufficient_information`) and boolean `can_generate_itinerary`.
  - `Gemini | Feasibility Assessment`: Google Gemini Chat Model (`models/gemini-3.5-flash`).
  - `Normalize Feasibility Assessment`: Code node parsing assessment JSON.
  - `Can Generate Itinerary?`: IF condition node ensuring `can_generate_itinerary` is true before proceeding.
  - **Inputs:** `Collect Feasibility Research`.
  - **Outputs:** Routes to `Prepare Itinerary Generation`.
  - **Edge Cases:** If `can_generate_itinerary` evaluates to false, the workflow halts to prevent generating flawed itineraries.

---

#### 2.3 Itinerary Generation, Storage & Delivery

##### 3.1 Itinerary Generation & Storage
- **Overview:** Generates a structured multi-day itinerary using Google Gemini, prepares pre-trip recheck plans, stores the trip record in an n8n Data Table, and builds an HTML display view.
- **Nodes Involved:** `Prepare Itinerary Generation`, `Generate Itinerary`, `Normalize Generated Itinerary`, `Prepare Itinerary Rechecks`, `Prepare Trip Record`, `Save Trip Record`, `Prepare Itinerary Response`, `Build Itinerary Display`, `Form Ending`, `Send a message`
- **Node Details:**
  - `Prepare Itinerary Generation`: Code node merging the full trip request, traveler profile, feasibility assessment, and verified research.
  - `Generate Itinerary`: Google Gemini Chat Model node (`models/gemini-3-flash-preview`) generating the day-by-day JSON schedule.
  - `Normalize Generated Itinerary`: Code node parsing and validating the generated itinerary.
  - `Prepare Itinerary Rechecks`: Code node extracting later recheck requirements and determining trip date boundaries.
  - `Prepare Trip Record`: Code node constructing a unique `trip_id` and serializing itinerary and recheck JSON structures.
  - `Save Trip Record`: n8n Data Table node (`travel_trip_records`). Inserts the newly created trip record.
  - `Prepare Itinerary Response`, `Build Itinerary Display`: Code nodes rendering responsive, localized HTML views of the itinerary, budget notes, and recheck items.
  - `Form Ending`: n8n Form completion node displaying the final itinerary inside the web form.
  - `Send a message`: Gmail node sending the final travel plan email using OAuth2 credentials (`n8n Creator Travel Planner Gmail`).
  - **Inputs:** `Can Generate Itinerary?` true branch.
  - **Outputs:** Final workflow termination nodes.
  - **Edge Cases:** Email delivery failure or JSON parsing errors in generated itineraries.

---

#### 2.4 Daily Pre-Trip Recheck Flow

##### 4.1 Scheduled Monitoring & Target Classification
- **Overview:** Runs daily on a schedule to retrieve planned trips from storage, identifies trips occurring the next day (`day_before`), and classifies weather locations, transport services, traffic areas, facilities, and special checks.
- **Nodes Involved:** `Daily Pre-Trip Recheck`, `Get Planned Trips`, `Find Trips Requiring Recheck`, `Prepare Recheck Research`, `Classify Recheck Targets`, `Normalize Recheck Targets`
- **Node Details:**
  - `Daily Pre-Trip Recheck`: Schedule Trigger node firing daily at 08:00.
  - `Get Planned Trips`: n8n Data Table node retrieving all records where `status` equals `planned`.
  - `Find Trips Requiring Recheck`: Code node filtering trips where `recheck_plan.required` is true, pre-trip recheck is enabled, and `days_until_trip === 1` (the day before departure), while preventing duplicate rechecks for the same itinerary version.
  - `Prepare Recheck Research`: Code node extracting itinerary schedules, locations, and transport modes into research tasks.
  - `Classify Recheck Targets`: Google Gemini model node (`models/gemini-3.1-flash-lite`) normalizing and categorizing weather locations, transport lines, traffic areas, and facilities into structured targets.
  - `Normalize Recheck Targets`: Code node parsing classification outputs.
  - **Inputs:** Scheduled trigger.
  - **Outputs:** Expands into the multi-source research branches (`Expand Weather Locations`, `Expand Transport Services`, `Expand Traffic Areas`, `Expand Facilities`, `Expand Special Checks`).
  - **Edge Cases:** Trips with missing date fields or already completed recheck versions are safely skipped.

##### 4.2 Multi-Source Pre-Trip Research
- **Overview:** Simultaneously rechecks real-time weather forecasts (Open-Meteo), public transport disruptions (Tavily + Gemini), traffic conditions, facility hours, and special checks.
- **Nodes Involved:** `Expand Weather Locations`, `Geocode Weather Locations`, `Prepare Weather Forecast Request`, `Get Weather Forecast`, `Normalize Weather Forecast`, `Aggregate Weather by Trip`, `Expand Transport Services`, `Prepare Transport Recheck Queries`, `Search Transport Recheck Web`, `Analyze Transport Recheck`, `Normalize Transport Recheck`, `Aggregate Transport by Trip`, `Expand Traffic Areas`, `Build Traffic Recheck Query`, `Search Traffic Recheck Web`, `Analyze Traffic Recheck`, `Normalize Traffic Recheck`, `Aggregate Traffic by Trip`, `Expand Facilities`, `Build Facility Recheck Query`, `Search Facility Recheck Web`, `Analyze Facility Recheck`, `Normalize Facility Recheck`, `Aggregate Facility by Trip`, `Expand Special Checks`, `Build Special Check Query`, `Search`, `Analyze Special Check`, `Normalize Special Check`, `Aggregate Special Checks by Trip`, `Wait for All Recheck Summaries`, `Combine Recheck Results by Trip`
- **Node Details:**
  - `Geocode Weather Locations`: HTTP Request node querying Open-Meteo Geocoding API (`https://geocoding-api.open-meteo.com/v1/search`).
  - `Get Weather Forecast`: HTTP Request node querying Open-Meteo Forecast API (`https://api.open-meteo.com/v1/forecast`) for daily weather codes, temperatures, precipitation, and winds.
  - `Search Transport Recheck Web`, `Search Traffic Recheck Web`, `Search Facility Recheck Web`, `Search`: Tavily nodes executing targeted web searches.
  - `Analyze Transport Recheck`, `Analyze Traffic Recheck`, `Analyze Facility Recheck`, `Analyze Special Check`: Google Gemini model nodes analyzing evidence and checking date relevance.
  - `Wait for All Recheck Summaries`: Merge node (5 inputs) synchronizing all five research branches.
  - `Combine Recheck Results by Trip`: Code node merging summaries into a unified recheck package per trip ID.
  - **Inputs:** `Normalize Recheck Targets`.
  - **Outputs:** Routes to `Synthesize Recheck Result`.
  - **Edge Cases:** Geocoding failures fallback to default coordinates or skip missing locations without crashing execution.

##### 4.3 Decision Synthesis & Itinerary Rebuilding
- **Overview:** Synthesizes all recheck results using AI to determine the final decision status (`no_change`, `action_required`, `change_required`, `uncertain`). If itinerary changes are required, it retrieves the current record, rebuilds the itinerary via Gemini, creates a new version, and updates storage.
- **Nodes Involved:** `Synthesize Recheck Result`, `Normalize Synthesized Recheck Result`, `Route Recheck Decision`, `Prepare Recheck No Change Notice`, `Prepare Recheck Action Notice`, `Prepare Recheck Uncertain Notice`, `Prepare Itinerary Rebuild Request`, `Get Current Itinerary for Rebuild`, `Build Itinerary Rebuild Context`, `Rebuild Itinerary`, `Normalize Rebuilt Itinerary`, `Prepare Rebuilt Itinerary Update`, `Update Rebuilt Trip Record`, `Build Rebuilt Itinerary Display`
- **Node Details:**
  - `Synthesize Recheck Result`: Google Gemini node (`models/gemini-3.1-flash-lite`) producing the final synthesized decision JSON and recommended next steps.
  - `Route Recheck Decision`: Switch node branching execution based on `final_status`.
  - `Rebuild Itinerary`: Google Gemini chat model (`models/gemini-3.5-flash`) revising the itinerary when changes are mandated while preserving unaffected constraints.
  - `Update Rebuilt Trip Record`: n8n Data Table update node saving the new `itinerary_json` and incremented `itinerary_version`.
  - **Inputs:** `Combine Recheck Results by Trip`.
  - **Outputs:** Routes to delivery and completion nodes.
  - **Edge Cases:** Parsing failures default to `uncertain` status to prevent unintended automated itinerary alterations.

##### 4.4 Completion & Notification Delivery
- **Overview:** Marks the scheduled recheck as completed in the data table, prepares localized notification messages, and delivers the recheck results to the traveler via Gmail.
- **Nodes Involved:** `Prepare Recheck Delivery`, `Prepare Recheck Completion Update`, `Mark Recheck Completed`, `Prepare Recheck Result`, `Build Recheck Result Display`, `Prepare Final Trip Response`, `Send Recheck Email`
- **Node Details:**
  - `Mark Recheck Completed`: n8n Data Table update node setting `recheck_completed_at` timestamp and `recheck_completed_version`.
  - `Send Recheck Email`: Gmail node delivering the final notice or rebuilt itinerary summary to the traveler using OAuth2 credentials (`n8n Creator Travel Planner Gmail`).
  - **Inputs:** Rebuild and notice preparation branches.
  - **Outputs:** Terminal email delivery.
  - **Edge Cases:** Database update conflicts are avoided by matching both `trip_id` and `itinerary_version`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| AI Travel Planner | formTrigger | Collects initial language choice and starts workflow | None | Normalize Locale | AI Travel Planner |
| Normalize Locale | code | Maps selected language to ISO code and locale | AI Travel Planner | Travel Profile Start | AI Travel Planner |
| Travel Profile Start | form | Collects travel pace, style, budget, and email | Normalize Locale | Normalize Travel Style | 1. Traveler Profile & Preferences |
| Normalize Travel Style | code | Normalizes travel style answers to database keys | Travel Profile Start | Travel Interests | 1. Traveler Profile & Preferences |
| Travel Interests | form | Collects traveler interests (max 5) | Normalize Travel Style | Normalize Travel Interests | 1. Traveler Profile & Preferences |
| Normalize Travel Interests | code | Normalizes and validates interest selections | Travel Interests | Travel Interests Valid? | 1. Traveler Profile & Preferences |
| Travel Interests Valid? | if | Checks if interests selection is valid | Normalize Travel Interests | Prepare Profile Preferences, Travel Interests Correction | 1. Traveler Profile & Preferences |
| Travel Interests Correction | form | Corrective form for invalid interest selections | Travel Interests Valid? | Normalize Corrected Travel Interests | 1. Traveler Profile & Preferences |
| Normalize Corrected Travel Interests | code | Normalizes corrected interest selections | Travel Interests Correction | Prepare Profile Preferences | 1. Traveler Profile & Preferences |
| Food & Stay Preferences | form | Collects food and accommodation priorities | Prepare Profile Preferences | Normalize Food & Stay Preferences | 1. Traveler Profile & Preferences |
| Normalize Food & Stay Preferences | code | Normalizes food and stay preference selections | Food & Stay Preferences | Food & Stay Preferences Valid? | 1. Traveler Profile & Preferences |
| Prepare Profile Preferences | code | Prepares profile preferences object | Travel Interests Valid?, Normalize Corrected Travel Interests | Food & Stay Preferences | 1. Traveler Profile & Preferences |
| Food & Stay Preferences Valid? | if | Validates food and stay preference selections | Normalize Food & Stay Preferences | Prepare Food & Stay Preferences, Food & Stay Preferences Correction | 1. Traveler Profile & Preferences |
| Food & Stay Preferences Correction | form | Corrective form for food/stay preferences | Food & Stay Preferences Valid? | Normalize Corrected Food & Stay Preferences | 1. Traveler Profile & Preferences |
| Normalize Corrected Food & Stay Preferences | code | Normalizes corrected food and stay preferences | Food & Stay Preferences Correction | Prepare Food & Stay Preferences | 1. Traveler Profile & Preferences |
| Prepare Food & Stay Preferences | code | Prepares preferences payload for considerations | Normalize Corrected Food & Stay Preferences, Normalize Food & Stay Preferences | Special Considerations | 1. Traveler Profile & Preferences |
| Special Considerations | form | Collects special travel considerations | Prepare Food & Stay Preferences | Normalize Special Considerations | 1. Traveler Profile & Preferences |
| Normalize Special Considerations | code | Normalizes special considerations selections | Special Considerations | Special Considerations Valid? | 1. Traveler Profile & Preferences |
| Special Considerations Valid? | if | Validates special considerations selections | Normalize Special Considerations | Prepare Travel Profile, Special Considerations Correction | 1. Traveler Profile & Preferences |
| Special Considerations Correction | form | Corrective form for special considerations | Special Considerations Valid? | Normalize Corrected Special Considerations | 1. Traveler Profile & Preferences |
| Normalize Corrected Special Considerations | code | Normalizes corrected special considerations | Special Considerations Correction | Prepare Travel Profile | 1. Traveler Profile & Preferences |
| Prepare Travel Profile | code | Prepares complete traveler profile data | Normalize Special Considerations Valid?, Normalize Corrected Special Considerations | Prepare Travel Profile Storage | 1. Traveler Profile & Preferences |
| Save Travel Profile | dataTable | Upserts traveler profile in data table | Prepare Travel Profile Storage | Trip Basics | 1. Traveler Profile & Preferences |
| Prepare Travel Profile Storage | code | Formats profile data for database storage | Prepare Travel Profile | Save Travel Profile | 1. Traveler Profile & Preferences |
| Trip Basics | form | Collects core trip basics (dates, origin, destination) | Save Travel Profile | Normalize Trip Basics | 2. Trip Requirements & Constraints |
| Normalize Trip Basics | code | Normalizes trip basics and calculates trip days | Trip Basics | Trip Basics Valid? | 2. Trip Requirements & Constraints |
| Trip Basics Valid? | if | Validates trip basics completeness | Normalize Trip Basics | Trip Preferences | 2. Trip Requirements & Constraints |
| Prepare Feasibility Request | code | Formats trip request for feasibility analysis | Normalize Trip Preferences | Analyze Feasibility Requirements | 3. Feasibility Research & Evidence Validation |
| Analyze Feasibility Requirements | chainLlm | AI identifies required real-world feasibility checks | Prepare Feasibility Request | Normalize Feasibility Requirements | 3. Feasibility Research & Evidence Validation |
| Normalize Feasibility Requirements | code | Parses and validates feasibility requirements JSON | Analyze Feasibility Requirements | Prepare Feasibility Research | 3. Feasibility Research & Evidence Validation |
| Prepare Feasibility Research | code | Separates hard checks from deferred planning checks | Normalize Feasibility Requirements | Hard Research Required? | 3. Feasibility Research & Evidence Validation |
| Hard Research Required? | if | Checks if hard research is required | Prepare Feasibility Research | Prepare Route Research | 3. Feasibility Research & Evidence Validation |
| Prepare Route Research | code | Categorizes checks into route API, transit, and general | Hard Research Required? | General Research Required?, Transit Research Required?, Route API Research Required? | 3. Feasibility Research & Evidence Validation |
| Transit Research Required? | if | Checks if transit research is required | Prepare Route Research | Research Transit, Transit Research Not Required | 3. Feasibility Research & Evidence Validation |
| Research Transit | chainLlm | AI analyzes public transportation checks | Transit Research Required? | Normalize Transit Research | 3. Feasibility Research & Evidence Validation |
| Gemini | Transit Research | lmChatGoogleGemini | Provides language model for transit research | Research Transit | 3. Feasibility Research & Evidence Validation |
| Gemini | Feasibility Analysis1 | lmChatGoogleGemini | Provides language model for feasibility requirements | Analyze Feasibility Requirements | 3. Feasibility Research & Evidence Validation |
| Normalize Transit Research | code | Parses transit research output | Research Transit | External Transit Research Required? | 3. Feasibility Research & Evidence Validation |
| External Transit Research Required? | if | Checks if external transit web search is needed | Normalize Transit Research | Prepare Transit Search Queries | 3. Feasibility Research & Evidence Validation |
| Search Transit Web | tavily | Searches web for transit evidence | Prepare Transit Search Queries | Verify Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Verify Transit Evidence | chainLlm | AI evaluates transit web search evidence | Search Transit Web | Normalize Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Gemini | Transit Verification | lmChatGoogleGemini | Provides language model for transit evidence verification | Verify Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Normalize Transit Evidence | code | Parses transit evidence verification output | Verify Transit Evidence | Additional Transit Research Required? | 3. Feasibility Research & Evidence Validation |
| Additional Transit Research Required? | if | Checks if second-pass transit research is needed | Normalize Transit Evidence | Prepare Additional Transit Research, Normalize Final Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Prepare Additional Transit Research | code | Prepares queries for second-pass transit research | Additional Transit Research Required? | Search Additional Transit Web | 3. Feasibility Research & Evidence Validation |
| Search Additional Transit Web | tavily | Executes additional web search for transit | Prepare Additional Transit Research | Aggregate Additional Transit Research | 3. Feasibility Research & Evidence Validation |
| Verify Additional Transit Evidence | chainLlm | AI verifies additional transit evidence | Aggregate Additional Transit Research | Normalize Final Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Gemini | Transit Evidence Verification | lmChatGoogleGemini | Provides language model for second-pass transit verification | Verify Additional Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Aggregate Additional Transit Research | code | Aggregates additional transit search results | Search Additional Transit Web | Verify Additional Transit Evidence | 3. Feasibility Research & Evidence Validation |
| Normalize Final Transit Evidence | code | Normalizes final transit evidence output | Verify Additional Transit Evidence, Additional Transit Research Required? | Transit Research Result | 3. Feasibility Research & Evidence Validation |
| General Research Required? | if | Checks if general research is required | Prepare Route Research | Generate General Research Queries, General Research Not Required | 3. Feasibility Research & Evidence Validation |
| Prepare General Research | code | Prepares general research queries from AI output | Generate General Research Queries | Search General Research Web | 3. Feasibility Research & Evidence Validation |
| Generate General Research Queries | chainLlm | AI generates focused general research queries | General Research Required? | Prepare General Research | 3. Feasibility Research & Evidence Validation |
| Gemini | General Research Queries | lmChatGoogleGemini | Provides language model for general research queries | Generate General Research Queries | 3. Feasibility Research & Evidence Validation |
| Search General Research Web | tavily | Searches web for general feasibility evidence | Prepare General Research | Analyze General Research Evidence | 3. Feasibility Research & Evidence Validation |
| Analyze General Research Evidence | chainLlm | AI analyzes general research web search results | Search General Research Web | Aggregate General Research Evidence | 3. Feasibility Research & Evidence Validation |
| Gemini | General Research Evidence | lmChatGoogleGemini | Provides language model for general evidence analysis | Analyze General Research Evidence | 3. Feasibility Research & Evidence Validation |
| Aggregate General Research Evidence | code | Aggregates general research evidence items | Analyze General Research Evidence | Synthesize General Research | 3. Feasibility Research & Evidence Validation |
| Synthesize General Research | chainLlm | AI synthesizes general research evidence | Aggregate General Research Evidence | Normalize General Research Synthesis | 3. Feasibility Research & Evidence Validation |
| Gemini | General Research Synthesis | lmChatGoogleGemini | Provides language model for general research synthesis | Synthesize General Research | 3. Feasibility Research & Evidence Validation |
| Normalize General Research Synthesis | code | Parses general research synthesis output | Synthesize General Research | General Research Result | 3. Feasibility Research & Evidence Validation |
| Collect Feasibility Research | code | Collects and aggregates all research branches | Merge Feasibility Research | Prepare Feasibility Assessment | 4. Final Feasibility Decision |
| Route API Research Required? | if | Checks if route API driving research is required | Prepare Route Research | Prepare Route API Research, Route API Not Required | 3. Feasibility Research & Evidence Validation |
| Prepare Route API Research | code | Prepares route API check payloads | Route API Research Required? | Get Driving Route | 3. Feasibility Research & Evidence Validation |
| Get Driving Route | httpRequest | Queries Google Maps Routes API for driving distance/time | Prepare Route API Research | Normalize Route API Research | 3. Feasibility Research & Evidence Validation |
| Normalize Route API Research | code | Parses Google Maps Routes API response | Get Driving Route | Route API Research Result | 3. Feasibility Research & Evidence Validation |
| Route API Not Required | set | Sets fallback state when route API is not required | Route API Research Required? | Route API Research Result | 3. Feasibility Research & Evidence Validation |
| General Research Not Required | set | Sets fallback state when general research is not required | General Research Required? | General Research Result | 3. Feasibility Research & Evidence Validation |
| Transit Research Not Required | set | Sets fallback state when transit research is not required | Transit Research Required? | Transit Research Result | 3. Feasibility Research & Evidence Validation |
| Merge Feasibility Research | merge | Merges research branch outputs | Transit Research Result, General Research Result, Route API Research Result | Collect Feasibility Research | 3. Feasibility Research & Evidence Validation |
| Transit Research Result | set | Formats transit research result | Normalize Final Transit Evidence, Transit Research Not Required | Merge Feasibility Research | 3. Feasibility Research & Evidence Validation |
| General Research Result | set | Formats general research result | Normalize General Research Synthesis, General Research Not Required | Merge Feasibility Research | 3. Feasibility Research & Evidence Validation |
| Route API Research Result | set | Formats route API research result | Normalize Route API Research, Route API Not Required | Merge Feasibility Research | 3. Feasibility Research & Evidence Validation |
| Prepare Feasibility Assessment | code | Prepares input payload for final feasibility assessment | Collect Feasibility Research | Assess Travel Feasibility | 4. Final Feasibility Decision |
| Assess Travel Feasibility | chainLlm | AI evaluates overall trip feasibility | Prepare Feasibility Assessment | Normalize Feasibility Assessment | 4. Final Feasibility Decision |
| Gemini | Feasibility Assessment | lmChatGoogleGemini | Provides language model for final feasibility assessment | Assess Travel Feasibility | 4. Final Feasibility Decision |
| Normalize Feasibility Assessment | code | Parses final feasibility assessment JSON | Assess Travel Feasibility | Can Generate Itinerary? | 4. Field Verification & Final Decision |
| Can Generate Itinerary? | if | Checks if itinerary generation can proceed | Normalize Feasibility Assessment | Prepare Itinerary Generation | 4. Final Feasibility Decision |
| Prepare Itinerary Generation | code | Prepares input payload for itinerary generation | Can Generate Itinerary? | Generate Itinerary | 5. Itinerary Generation, Storage & Delivery |
| Trip Preferences | form | Collects trip-specific preferences and details | Trip Basics Valid? | Normalize Trip Preferences | 2. Trip Requirements & Constraints |
| Normalize Trip Preferences | code | Normalizes trip preferences and answers | Trip Preferences | Prepare Full Trip Request | 2. Trip Requirements & Constraints |
| Prepare Transit Search Queries | code | Generates search queries for transit verification | External Transit Research Required? | Search Transit Web | 3. Feasibility Research & Evidence Validation |
| Generate Itinerary | googleGemini | AI generates day-by-day travel itinerary | Prepare Itinerary Generation | Normalize Generated Itinerary | 5. Itinerary Generation, Storage & Delivery |
| Normalize Generated Itinerary | code | Parses generated itinerary JSON | Generate Itinerary | Prepare Itinerary Rechecks | 5. Itinerary Generation, Storage & Delivery |
| Prepare Itinerary Rechecks | code | Prepares pre-trip recheck plan from itinerary | Normalize Generated Itinerary | Prepare Trip Record | 5. Itinerary Generation, Storage & Delivery |
| Prepare Trip Record | code | Structures trip record for database storage | Prepare Itinerary Rechecks | Save Trip Record | 5. Itinerary Generation, Storage & Delivery |
| Save Trip Record | dataTable | Inserts trip record into database | Prepare Trip Record | Prepare Itinerary Response | 5. Itinerary Generation, Storage & Delivery |
| Prepare Itinerary Response | code | Parses saved trip record and itinerary | Save Trip Record | Build Itinerary Display | 5. Itinerary Generation, Storage & Delivery |
| Build Itinerary Display | code | Renders HTML display view of travel itinerary | Prepare Itinerary Response | Form Ending, Send a message | 5. Itinerary Generation, Storage & Delivery |
| Form Ending | form | Displays completed itinerary in web form | Build Itinerary Display | None | 5. Itinerary Generation, Storage & Delivery |
| Send a message | gmail | Sends final itinerary via Gmail | Build Itinerary Display | None | 5. Itinerary Generation, Storage & Delivery |
| Sticky Note | stickyNote | General workflow description and setup guide | None | None | None |
| Sticky Note1 | stickyNote | Overview of traveler profile and preferences | None | None | None |
| Sticky Note2 | stickyNote | Overview of trip requirements and constraints | None | None | None |
| Sticky Note3 | stickyNote | Overview of feasibility research and evidence validation | None | None | None |
| Sticky Note4 | stickyNote | Overview of final feasibility decision | None | None | None |
| Sticky Note5 | stickyNote | Overview of itinerary generation, storage, and delivery | None | None | None |
| Daily Pre-Trip Recheck | scheduleTrigger | Triggers daily pre-trip monitoring at 08:00 | None | Get Planned Trips | Recheck README |
| Get Planned Trips | dataTable | Retrieves planned trips from data table | Daily Pre-Trip Recheck | Find Trips Requiring Recheck | Recheck README |
| Find Trips Requiring Recheck | code | Filters trips due for pre-trip recheck | Get Planned Trips | Prepare Recheck Research | Recheck README |
| Prepare Recheck Research | code | Prepares recheck tasks from trip record | Find Trips Requiring Recheck | Classify Recheck Targets | Recheck README |
| Classify Recheck Targets | googleGemini | AI classifies recheck targets and locations | Prepare Recheck Research | Normalize Recheck Targets | Recheck README |
| Normalize Recheck Targets | code | Parses recheck targets classification | Classify Recheck Targets | Expand Weather Locations, Expand Transport Services, Expand Traffic Areas, Expand Facilities, Expand Special Checks | Recheck README |
| Geocode Weather Locations | httpRequest | Geocodes weather locations via Open-Meteo API | Expand Weather Locations | Prepare Weather Forecast Request | Recheck README |
| Expand Weather Locations | code | Expands weather locations into individual items | Normalize Recheck Targets | Geocode Weather Locations | Recheck README |
| Prepare Weather Forecast Request | code | Prepares weather forecast request parameters | Geocode Weather Locations | Get Weather Forecast | Recheck README |
| Get Weather Forecast | httpRequest | Requests weather forecast from Open-Meteo API | Prepare Weather Forecast Request | Normalize Weather Forecast | Recheck README |
| Normalize Weather Forecast | code | Normalizes weather forecast days | Get Weather Forecast | Aggregate Weather by Trip | Recheck README |
| Expand Transport Services | code | Expands transport services into individual items | Normalize Recheck Targets | Prepare Transport Recheck Queries | Recheck README |
| Prepare Transport Recheck Queries | code | Builds search queries for transport disruptions | Expand Transport Services | Search Transport Recheck Web | Recheck README |
| Search Transport Recheck Web | tavily | Searches web for transport recheck evidence | Prepare Transport Recheck Queries | Analyze Transport Recheck | Recheck README |
| Analyze Transport Recheck | googleGemini | AI analyzes transport recheck evidence | Search Transport Recheck Web | Normalize Transport Recheck | Recheck README |
| Normalize Transport Recheck | code | Normalizes transport recheck analysis | Analyze Transport Recheck | Aggregate Transport by Trip | Recheck README |
| Expand Traffic Areas | code | Expands traffic areas into individual items | Normalize Recheck Targets | Build Traffic Recheck Query | Recheck README |
| Build Traffic Recheck Query | code | Builds search queries for traffic congestion | Expand Traffic Areas | Search Traffic Recheck Web | Recheck README |
| Search Traffic Recheck Web | tavily | Searches web for traffic recheck evidence | Build Traffic Recheck Query | Analyze Traffic Recheck | Recheck README |
| Analyze Traffic Recheck | googleGemini | AI analyzes traffic recheck evidence | Search Traffic Recheck Web | Normalize Traffic Recheck | Recheck README |
| Normalize Traffic Recheck | code | Normalizes traffic recheck analysis | Analyze Traffic Recheck | Aggregate Traffic by Trip | Recheck README |
| Expand Facilities | code | Expands facilities into individual items | Normalize Recheck Targets | Build Facility Recheck Query | Recheck README |
| Build Facility Recheck Query | code | Builds search queries for facility availability | Expand Facilities | Search Facility Recheck Web | Recheck README |
| Search Facility Recheck Web | tavily | Searches web for facility recheck evidence | Build Facility Recheck Query | Analyze Facility Recheck | Recheck README |
| Analyze Facility Recheck | googleGemini | AI analyzes facility recheck evidence | Search Facility Recheck Web | Normalize Facility Recheck | Recheck README |
| Normalize Facility Recheck | code | Normalizes facility recheck analysis | Analyze Facility Recheck | Aggregate Facility by Trip | Recheck README |
| Expand Special Checks | code | Expands special checks into individual items | Normalize Recheck Targets | Build Special Check Query | Recheck README |
| Build Special Check Query | code | Builds search queries for special checks | Expand Special Checks | Search | Recheck README |
| Search | tavily | Searches web for special check evidence | Build Special Check Query | Analyze Special Check | Recheck README |
| Analyze Special Check | googleGemini | AI analyzes special check evidence | Search | Normalize Special Check | Recheck README |
| Normalize Special Check | code | Normalizes special check analysis | Analyze Special Check | Aggregate Special Checks by Trip | Recheck README |
| Aggregate Special Checks by Trip | code | Aggregates special checks by trip ID | Normalize Special Check | Wait for All Recheck Summaries | Recheck README |
| Aggregate Transport by Trip | code | Aggregates transport checks by trip ID | Normalize Transport Recheck | Wait for All Recheck Summaries | Recheck README |
| Aggregate Traffic by Trip | code | Aggregates traffic checks by trip ID | Normalize Traffic Recheck | Wait for All Recheck Summaries | Recheck README |
| Aggregate Facility by Trip | code | Aggregates facility checks by trip ID | Normalize Facility Recheck | Wait for All Recheck Summaries | Recheck README |
| Aggregate Weather by Trip | code | Aggregates weather locations by trip ID | Normalize Weather Forecast | Wait for All Recheck Summaries | Recheck README |
| Combine Recheck Results by Trip | code | Combines all aggregated research summaries per trip | Wait for All Recheck Summaries | Synthesize Recheck Result | Recheck README |
| Wait for All Recheck Summaries | merge | Waits for all five recheck research summaries | Aggregate Weather by Trip, Aggregate Transport by Trip, Aggregate Traffic by Trip, Aggregate Facility by Trip, Aggregate Special Checks by Trip | Combine Recheck Results by Trip | Recheck README |
| Synthesize Recheck Result | googleGemini | AI synthesizes all recheck evidence into final decision | Combine Recheck Results by Trip | Normalize Synthesized Recheck Result | Recheck README |
| Normalize Synthesized Recheck Result | code | Parses synthesized recheck result JSON | Synthesize Recheck Result | Route Recheck Decision | Recheck README |
| Route Recheck Decision | switch | Routes workflow based on final recheck status | Normalize Synthesized Recheck Result | Prepare Recheck No Change Notice, Prepare Recheck Action Notice, Prepare Itinerary Rebuild Request, Prepare Recheck Uncertain Notice | Recheck README |
| Prepare Recheck Action Notice | code | Prepares action required notice for traveler | Route Recheck Decision | Prepare Recheck Delivery | Recheck README |
| Prepare Recheck Uncertain Notice | code | Prepares uncertain recheck notice | Route Recheck Decision | Prepare Recheck Delivery | Recheck README |
| Prepare Itinerary Rebuild Request | code | Prepares request to rebuild itinerary | Route Recheck Decision | Get Current Itinerary for Rebuild | Recheck README |
| Get Current Itinerary for Rebuild | dataTable | Retrieves current itinerary from data table for rebuilding | Prepare Itinerary Rebuild Request | Build Itinerary Rebuild Context | Recheck README |
| Build Itinerary Rebuild Context | code | Prepares rebuild context combining request and current itinerary | Get Current Itinerary for Rebuild | Rebuild Itinerary | Recheck README |
| Rebuild Itinerary | googleGemini | AI rebuilds affected itinerary parts | Build Itinerary Rebuild Context | Normalize Rebuilt Itinerary | Recheck README |
| Normalize Rebuilt Itinerary | code | Normalizes rebuilt itinerary JSON | Rebuild Itinerary | Prepare Rebuilt Itinerary Update | Recheck README |
| Prepare Rebuilt Itinerary Update | code | Prepares data table update payload for rebuilt itinerary | Normalize Rebuilt Itinerary | Update Rebuilt Trip Record | Recheck README |
| Update Rebuilt Trip Record | dataTable | Updates trip record in data table with new itinerary version | Prepare Rebuilt Itinerary Update | Build Rebuilt Itinerary Display | Recheck README |
| Build Rebuilt Itinerary Display | code | Renders HTML display for rebuilt itinerary | Update Rebuilt Trip Record | Prepare Recheck Completion Update | Recheck README |
| Prepare Recheck Completion Update | code | Prepares recheck completion status update | Build Rebuilt Itinerary Display, Prepare Recheck Delivery | Mark Recheck Completed | Recheck README |
| Mark Recheck Completed | dataTable | Marks recheck completed in data table | Prepare Recheck Completion Update | Prepare Recheck Result | Recheck README |
| Prepare Recheck Result | code | Prepares final recheck result payload | Mark Recheck Completed | Build Recheck Result Display | Recheck README |
| Build Recheck Result Display | code | Builds final recheck result display message | Prepare Recheck Result | Prepare Final Trip Response | Recheck README |
| Prepare Final Trip Response | code | Prepares final response message for delivery | Build Recheck Result Display | Send Recheck Email | Recheck README |
| Prepare Recheck No Change Notice | code | Prepares no-change notice text | Route Recheck Decision | Prepare Recheck Delivery | Recheck README |
| Prepare Recheck Delivery | code | Prepares delivery envelope for recheck notices | Prepare Recheck No Change Notice, Prepare Recheck Action Notice, Prepare Recheck Uncertain Notice | Prepare Recheck Completion Update | Recheck README |
| Send Recheck Email | gmail | Sends recheck result notification email via Gmail | Prepare Final Trip Response | None | Recheck README |
| Recheck README | stickyNote | Documentation note for Daily Pre-Trip Recheck workflow | None | None | None |
| Recheck Sticky Note 1 | stickyNote | Overview of trip selection and recheck preparation | None | None | None |
| Recheck Sticky Note 2 | stickyNote | Overview of multi-source pre-trip research | None | None | None |
| Recheck Sticky Note 3 | stickyNote | Overview of recheck synthesis and decision | None | None | None |
| Recheck Sticky Note 4 | stickyNote | Overview of response, itinerary rebuild, and delivery | None | None | None |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this entire n8n workflow manually without importing the JSON, follow these sequential setup and node construction steps:

#### Step 1: Environment & Credential Prerequisites
1. **Google Gemini (Google PaLM) Credential:** Create an API credential (`googlePalmApi`) for Gemini API access.
2. **Tavily API Credential:** Create a Tavily credential (`tavilyApi`) for web search tools.
3. **Google Maps Routes API:** Create a Generic Header Auth credential (`httpHeaderAuth`) with header name `X-Goog-FieldMask` and your API key value. Ensure the Routes API is enabled in your Google Cloud Console.
4. **Gmail OAuth2 Credential:** Set up a Gmail OAuth2 credential (`gmailOAuth2`) for sending traveler itineraries and notifications.
5. **n8n Data Tables:** Create two tables in your workspace:
   - `travel_profiles`: Columns for `profile_key`, `profile_name`, `profile_version`, `preferred_language`, `preferred_locale`, `travel_pace`, `travel_style`, `budget_style`, `travel_interests`, `food_preferences`, `stay_preferences`, `special_considerations`, `profile_complete`, `profile_session_started_at`.
   - `travel_trip_records`: Columns for `trip_id`, `status`, `trip_start_date`, `trip_end_date`, `language`, `itinerary_version`, `itinerary_json`, `recheck_plan_json`, `recheck_completed_at`, `recheck_completed_version`, `email`.

#### Step 2: Build the Traveler Profile & Preferences Flow
1. **AI Travel Planner:** Create a Form Trigger node (`n8n-nodes-base.formTrigger`) titled "AI Travel Planner" with a dropdown field `Language / 言語` supporting 9 languages.
2. **Normalize Locale:** Add a Code node (`n8n-nodes-base.code`) to map form language names to ISO language codes (`en`, `ja`, etc.).
3. **Travel Profile Start & Normalize Travel Style:** Create a dynamic Form node and a Code node to collect and normalize travel pace, style, and budget preferences.
4. **Travel Interests, Validation, and Correction:** Create a Form node for checkbox selection of interests (min 1, max 5), followed by an IF node (`Travel Interests Valid?`) and a correction Form node for exclusivity validation ("Let AI decide" rule).
5. **Food & Stay Preferences & Special Considerations:** Repeat similar patterns for food preferences, accommodation priorities, and special considerations using dynamic forms, validation IF nodes, and code normalization nodes.
6. **Save Profile:** Add a Code node (`Prepare Travel Profile Storage`) to JSON-stringify array fields and connect it to a Data Table node (`Save Travel Profile`) configured to upsert into `travel_profiles`.

#### Step 3: Build Trip Requirements & Feasibility Research Flow
1. **Trip Basics:** Create a Form node capturing origin, destinations, start date, end date, and primary purpose.
2. **Normalize Trip Basics & Trip Preferences:** Add normalization code nodes and a subsequent Form node (`Trip Preferences`) collecting companion type, transport mode, budget, accommodation status, and must-visit notes.
3. **Feasibility Request & LLM Analysis:** Combine requests using a Code node (`Prepare Full Trip Request`), then attach an AI Agent / LLM Chain node (`Analyze Feasibility Requirements`) connected to a Gemini Chat Model node using `models/gemini-3.1-flash-lite`.
4. **Multi-Channel Research Branches:**
   - **Transit Branch:** Connect IF nodes (`Transit Research Required?`), a LangChain Chain node (`Research Transit`) with Tavily web search (`Search Transit Web`) and evidence verification nodes.
   - **General Research Branch:** Connect IF nodes (`General Research Required?`), query generation, Tavily search, and synthesis LLM nodes.
   - **Route API Branch:** Connect IF nodes (`Route API Research Required?`), an HTTP Request node (`Get Driving Route`) targeting `https://routes.googleapis.com/directions/v2:computeRoutes` using your HTTP Header Auth credential.
5. **Final Feasibility Gate:** Merge research branches (`Merge Feasibility Research`), collect results, and attach an AI Assessment Chain node (`Assess Travel Feasibility`) powered by `models/gemini-3.5-flash`. Add an IF node (`Can Generate Itinerary?`) to block infeasible trips.

#### Step 4: Build Itinerary Generation, Storage & Delivery Flow
1. **Generate Itinerary:** Attach a Google Gemini Chat Model node (`models/gemini-3-flash-preview`) to generate structured day-by-day itineraries.
2. **Storage:** Add Code nodes to prepare recheck plans and trip records, then insert them using a Data Table node (`Save Trip Record`) targeting `travel_trip_records`.
3. **Delivery:** Add Code nodes (`Build Itinerary Display`) to generate responsive HTML, connect a Form Completion node (`Form Ending`), and a Gmail node (`Send a message`) using your Gmail OAuth2 credential.

#### Step 5: Build the Daily Pre-Trip Recheck Companion Flow
1. **Schedule Trigger:** Create a Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`) set to execute daily at 08:00.
2. **Trip Selection:** Add a Data Table node (`Get Planned Trips`) to retrieve rows where status equals `planned`, followed by a Code node (`Find Trips Requiring Recheck`) filtering trips departing tomorrow (`days_until_trip === 1`).
3. **Recheck Research Branches:** Add Gemini classification nodes, geocoding nodes, Open-Meteo weather forecast HTTP request nodes (`https://api.open-meteo.com/v1/forecast`), and Tavily web search nodes for transport, traffic, facilities, and special checks.
4. **Synthesis & Rebuild:** Merge research branches, add a Gemini decision node (`Synthesize Recheck Result`), and a Switch node (`Route Recheck Decision`). For changes required, connect data table retrieval nodes, an itinerary rebuild Gemini node (`models/gemini-3.5-flash`), data table update nodes, and Gmail notification nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Workflow Components & Sticky Notes Documentation | [n8n Sticky Notes Guide](https://docs.n8n.io/workflows/components/sticky-notes/) |
| Google Maps Routes API Documentation | [Google Maps Routes API Guide](https://developers.google.com/maps/documentation/routes) |
| Open-Meteo Weather & Geocoding API Documentation | [Open-Meteo Documentation](https://open-meteo.com/en/docs) |
| Tavily AI Search API Documentation | [Tavily API Documentation](https://docs.tavily.com/) |