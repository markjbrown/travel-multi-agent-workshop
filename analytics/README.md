# Travel Multi-Agent Analytics Guide

This guide walks you through generating realistic data in the Travel Multi-Agent application, then using Microsoft Fabric to analyze multi-agent memory patterns, trip planning behavior, and user preferences.

## What You'll Build

- **Data Generator** — A Python script that simulates 12 diverse users having realistic conversations with the travel assistant, generating memories, trips, and conversation data in Cosmos DB.
- **Spark Notebook** — A Fabric notebook that reads mirrored Cosmos DB data, flattens nested JSON structures, and writes 10 analytical Delta tables to OneLake.
- **SQL Queries** — Ready-to-run queries for the Fabric SQL Analytics Endpoint for quick ad-hoc exploration.
- **Power BI Report** — A 5-page report visualizing memory intelligence, user preferences, trip patterns, destination insights, and memory health.

## Prerequisites

- The Travel Multi-Agent application deployed and running ([see main README](../README.md))
- Azure Cosmos DB with the application's containers populated
- Microsoft Fabric workspace with:
  - Cosmos DB Mirroring configured for: **Memories**, **Users**, **Trips**, **Places**
  - A Lakehouse for analytical tables

---

## Step 1: Generate Data

The application ships with only a handful of seed users and trips. The data generator creates realistic conversation data by simulating 12 user personas interacting with the travel assistant.

### 1.1 Set Up the Analytics Virtual Environment

```powershell
cd analytics
python -m venv .venv
.venv\Scripts\Activate.ps1      # Windows
# source .venv/bin/activate     # macOS/Linux
pip install httpx
```

### 1.2 Start the Application

You need two services running before the data generator can work. Open separate terminals for each:

**Terminal 1 — MCP Tool Server** (start first):

```powershell
cd 02_completed\mcp_server
..\venv\Scripts\Activate.ps1
$env:PYTHONPATH="..\python"
python mcp_http_server.py
```

Wait for `Travel Assistant MCP server initialized`.

**Terminal 2 — API Server**:

```powershell
cd 02_completed\python
..\venv\Scripts\Activate.ps1
uvicorn src.app.travel_agents_api:app --reload --host 0.0.0.0 --port 8000
```

Wait for `✅ Agents initialized successfully!`

> **Note:** The Angular frontend (port 4200) is **not required** for data generation. The data generator talks directly to the API server.

### 1.3 Run a Dry Run

Preview what the generator will do without making any API calls:

```powershell
cd analytics
.venv\Scripts\Activate.ps1
python data_generator.py --dry-run --personas 2
```

This shows the user personas, their conversations, and all messages that will be sent.

### 1.4 Generate Data

Start with a small run to verify everything works:

```powershell
python data_generator.py --personas 2
```

Once confirmed, generate the full dataset (12 personas, ~30 conversations, ~80 messages):

```powershell
python data_generator.py
```

> **Note:** The data generator takes **2+ hours** to complete. It simulates real users having conversations one message at a time, and the multi-agent system takes 30-90 seconds to process each message (searching places, extracting memories, creating itineraries). You can safely leave it running and come back later -- progress is logged to the console in real time.

> **Azure OpenAI TPM Requirements:** Each user message triggers 7-8 LLM calls internally (orchestrator routing, preference extraction, conflict resolution, specialist agent, place search). For the full 292-message run, the system consumes approximately **10M tokens** on the completion model. Recommended TPM settings:
>
> | Model Deployment | Recommended TPM | Notes |
> |-----------------|----------------|-------|
> | **Completion model (e.g. gpt-4o)** | **150K TPM** | 100K minimum but expect occasional 429 throttling. 150K gives headroom for itinerary-heavy sequences. |
> | **Embedding model (e.g. text-embedding-3-small)** | **10K TPM** | Actual peak usage is ~2K TPM. The default deployment minimum is sufficient. |
>
> If you hit rate limit errors (429s), increase `--delay` to space out messages (e.g., `--delay 5` or `--delay 10`).

**Command-line options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--base-url` | `http://localhost:8000` | Travel API base URL |
| `--tenant` | `analytics_demo` | Tenant ID for generated data |
| `--personas` | all 12 | Limit to first N personas |
| `--delay` | `3` | Seconds between messages |
| `--timeout` | `180` | HTTP timeout per request (seconds) |
| `--dry-run` | off | Print plan without calling API |

### 1.5 What Gets Generated

The 12 personas cover diverse traveler archetypes:

| Persona | Style | Key Preferences | Destinations |
|---------|-------|-----------------|--------------|
| Maya Chen | Budget backpacker | Vegan, peanut allergy, hostel | Bangkok, Tokyo |
| James Mitchell | Luxury couple | Fine dining, 5-star, romantic | Paris, Rome |
| Sarah Johnson | Family with kids | Gluten-free, wheelchair access, kid-friendly | London, Barcelona |
| David Okafor | Business traveler | Halal, executive hotels, nightlife | Singapore, Dubai, Frankfurt |
| Elena Vasquez | Adventure solo | Vegetarian, eco-lodges, hiking | New Zealand, Iceland |
| Robert Williams | Retired couple | Low-sodium, senior-friendly, classical | Barcelona, Lisbon, Vienna |
| Jordan Taylor | College group | Budget, nightlife, cheap eats | Miami, Amsterdam |
| Priya Sharma | Food tourism | All cuisines, food tours, cooking classes | Tokyo, Istanbul, Copenhagen |
| Alex Brennan | Digital nomad | Pescatarian→vegetarian, wifi priority | Lisbon, Bali |
| Isabelle Dupont | Art & culture | Flexitarian, galleries, design hotels | Amsterdam, Berlin |
| Aisha Rahman | Wellness seeker | Halal, yoga, spa, organic | Bali, Stockholm |
| Marco Rossi | History & architecture | Traditional cuisine, historic hotels | Prague, Istanbul, Budapest |

Each persona has 2-3 conversations that generate:

- **Memories** -- dietary restrictions, budget preferences, accessibility needs, style preferences (declarative, procedural, episodic types)
- **Trips** -- day-by-day itineraries with hotel, restaurant, and activity recommendations
- **Memory conflicts** -- e.g., Maya updates from "vegan" to "pescatarian", Alex goes from "pescatarian" to "vegetarian"
- **Trip status mix** -- after all conversations, the generator automatically updates some trips to "confirmed" and "completed" status

### 1.6 Enrich with Preference Conflicts (Optional)

After running the data generator, you can optionally run the enricher to add **more preference conflicts** for richer analytics. The enricher sends short conversations where users contradict their earlier preferences, triggering memory supersession in the AI system.

```powershell
python data_enricher.py --dry-run    # Preview
python data_enricher.py              # Run
```

The enricher adds conflicts for 6 users not already covered by the generator:

| User | Conflict | Original Preference |
|------|----------|-------------------|
| Sarah Johnson | No longer gluten-free, no wheelchair needed, wants luxury | Was gluten-free, wheelchair access, budget |
| David Okafor | Plant-based diet, prefers co-working, early mornings | Was halal, executive lounges, nightlife |
| Elena Vasquez | Now omnivore, prefers luxury, wants guided tours | Was vegetarian, eco-lodges, solo hiking |
| Jordan Taylor | Wants nice hotel, cultural experiences, local cuisine | Was cheapest hotel, nightlife, fast food |
| Alex Brennan | Eating meat again, no wifi priority, chain hotels | Was vegetarian, wifi-first, boutique |
| Isabelle Dupont | Now fully vegan, street art, budget accommodation | Was flexitarian, galleries, design hotels |

These conflicts create **superseded memories** that show up in the Memory Health page of the Power BI report, demonstrating the AI's ability to handle changing preferences.

**Command-line options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--base-url` | `http://localhost:8000` | Travel API base URL |
| `--tenant` | `analytics_demo` | Tenant ID for generated data |
| `--delay` | `3` | Seconds between messages |
| `--timeout` | `300` | HTTP timeout per request (seconds) |
| `--dry-run` | off | Print plan without calling API |

---

## Step 2: Mirror Data to Fabric

If you haven't already set up Cosmos DB Mirroring:

1. In your Fabric workspace, create a **Mirrored Database** item. Name it **CosmosTravelAssistant**.
2. Connect it to your Azure Cosmos DB account.
3. Select these containers to mirror: **Memories**, **Users**, **Trips**, **Places**.
4. Start mirroring and wait for initial sync to complete.

The mirrored data appears as Delta tables accessible via both Spark and the SQL Analytics Endpoint. The schema inside the mirrored database will automatically be named **TravelAssistant**, matching the Cosmos DB database name from the Bicep deployment.

---

## Step 3: Run the Spark Notebook

The notebook (`TravelAgentAnalyticsNotebook.ipynb`) reads from the mirrored database and writes analytical tables to a Lakehouse.

### 3.1 Upload to Fabric

1. In your Fabric workspace, create a **Lakehouse** named **TravelAssistantLakehouse**. During creation, check **Enable schemas** (required for mirrored database access).
2. Click **Import notebook** and upload `analytics/TravelAgentAnalyticsNotebook.ipynb`.
3. Open the notebook in the Fabric portal.

### 3.2 Attach Data Sources

Before running the notebook, you need to attach both the Lakehouse and the mirrored database:

1. **Attach the Lakehouse**: In the notebook's left sidebar (Explorer), your Lakehouse should already be attached. If not, click **Add Lakehouse** and select **TravelAssistantLakehouse**.
2. **Attach the Mirrored Database**: Click **Add data source** > **Existing data sources** > select **CosmosTravelAssistant** (your mirrored database). You should see the **TravelAssistant** schema with the four tables (Memories, Users, Trips, Places) in the Explorer sidebar.

> **Important:** The notebook reads data via `pyodbc` (not through the Explorer attachments), but attaching the mirrored database confirms it is accessible from your notebook environment.

### 3.3 How It Reads Data

The notebook connects directly to the mirrored database's **SQL Analytics Endpoint** using `pyodbc` with AAD token authentication. This bypasses Spark's catalog entirely (which cannot resolve mirrored databases) and reads data via standard SQL queries into pandas DataFrames, then converts to Spark.

You need the SQL connection string from the Fabric portal:

1. Open your mirrored database in the Fabric portal.
2. Go to **Settings** (or the connection info panel).
3. Copy the **SQL connection string** (it looks like `xxxxx.datawarehouse.fabric.microsoft.com`).

### 3.4 Configure

Update these values in the first code cell:

```python
WORKSPACE_NAME = "YourWorkspaceName"
MIRRORED_DB = "CosmosTravelAssistant"
MIRRORED_SCHEMA = "TravelAssistant"
SQL_ENDPOINT = "xxxxx.datawarehouse.fabric.microsoft.com"  # from Fabric portal
```

> **Note:** `MIRRORED_DB` is the name you gave the mirrored database artifact in Fabric (e.g., `CosmosTravelAssistant`). `MIRRORED_SCHEMA` is the Cosmos DB database name, which is automatically set to `TravelAssistant` by the Bicep deployment. If you followed the naming above, these values do not need to change -- only `WORKSPACE_NAME` and `SQL_ENDPOINT` require updating.

### 3.5 Run All Cells

The notebook produces these tables in your Lakehouse:

| Layer | Table | Description |
|-------|-------|-------------|
| Silver | `silver_memories_flat` | Memories with facets extracted as columns |
| Silver | `silver_trips_days` | Trips exploded to one row per day |
| Silver | `silver_trip_activities` | One row per activity slot (morning/lunch/afternoon/dinner/accommodation) |
| Gold | `gold_user_memory_profile` | Per-user KPIs: memory counts, avg salience, conflict rate |
| Gold | `gold_memory_salience_analysis` | Per-memory detail: health, lifespan, recall tracking |
| Gold | `gold_destination_popularity` | Trip counts, duration, travelers per destination |
| Gold | `gold_place_inventory` | Hotels/restaurants/activities per city |
| Gold | `gold_popular_places` | Most recommended places across all trips |
| Gold | `gold_memory_trip_alignment` | Do stored preferences match actual trip recommendations? |
| Gold | `gold_memory_lifecycle` | Supersession rates, recall rates, short-term vs. long-term health |

---

## Step 4: Power BI Report

The report visualizes how the multi-agent system learns about users, plans trips, and uses memory to personalize recommendations. It has 5 pages that tell a story about the AI's memory and decision-making patterns.

### Option A: Use the Pre-built Report

A pre-built Power BI report is included in this repository:

1. Download `TravelAnalyticsReport.pbix` from the `analytics/` folder.
2. Open it in **Power BI Desktop**.
3. Update the data source connection to point to your Lakehouse SQL endpoint:
   - Go to **Home > Transform data > Data source settings**.
   - Update the server URL to your Lakehouse's SQL Analytics Endpoint.
   - You can find this URL in the Fabric portal: open your Lakehouse > click the **SQL Analytics Endpoint** dropdown > copy the connection string.
4. Click **Refresh** to load your data.
5. Publish to your Fabric workspace if desired.

> **Important:** The `.pbix` file contains a data source connection tied to the original author's Fabric workspace. You **must** update this connection to your own Lakehouse SQL endpoint before the report will work. The table and column names are identical across all environments (they are created by the notebook), so only the server URL needs to change. Your workspace name does not matter.

### Option B: Build from Scratch

Create a new report from your Lakehouse:

1. Open your Lakehouse's **SQL Analytics Endpoint** in the Fabric portal.
2. Click **New semantic model** and select the `gold_*` and `silver_*` tables.
3. From the semantic model, click **Create report**.
4. Build the 5 pages described below.

---

### Page 1: Memory Intelligence Overview

This page answers: **How much has the AI learned about its users?**

It shows the total volume of memories the system has extracted from conversations, broken down by type (declarative facts, procedural patterns, episodic trip-specific). The salience distribution reveals how confident the system is in what it has learned, while the per-user breakdown shows which users the system knows best.

**Key insights to highlight:**

- Declarative memories (permanent facts like "vegan" or "allergic to peanuts") vs. episodic memories (trip-specific, 90-day TTL)
- High-salience memories indicate strong, clearly stated preferences
- The superseded count shows how many times the AI corrected its understanding when users changed their minds

![Memory Intelligence](media/report_page1_memory_intelligence.png)

---

### Page 2: Memory Health

This page answers: **How healthy is the AI's memory system? Are stored preferences actually influencing recommendations?**

This is the most analytically interesting page. The short-term vs. long-term memory chart shows the balance between permanent preferences (dietary, accessibility) and trip-specific episodic memories (which expire after 90 days). The memory lifecycle table reveals supersession rates — how often the AI had to update its understanding when users contradicted earlier statements. The alignment table is the "closing the loop" metric: do users with hotel preferences actually get hotel recommendations in their trips?

**Key insights to highlight:**

- Memory supersession demonstrates the AI's ability to handle conflicting information (e.g., "I'm vegan" followed by "I actually eat seafood now")
- The recall rate shows what percentage of memories were re-used after extraction — memories that are never recalled may indicate over-extraction
- The alignment score directly measures whether the memory system influences trip planning outcomes

![Memory Health](media/report_page2_memory_health.png)

---

### Page 3: User Preference Profiles

This page answers: **What does the AI know about each user's preferences?**

The preference categories matrix shows the types of preferences stored per user — dietary restrictions, hotel style, cuisine, activity types, budget, and accessibility needs. This is the AI's "mental model" of each traveler, built entirely from natural language conversations.

**Key insights to highlight:**

- Maya Chen has dietary preferences that changed over time (vegan to pescatarian) — visible as superseded memories
- David Okafor's halal dietary requirement appears consistently across all his conversations
- Some users have preferences across many categories (well-known to the system) while others are sparse

![User Preferences](media/report_page3_user_preferences.png)

---

### Page 4: Trip Planning Insights

This page answers: **Where are users traveling, and what's the status of their plans?**

The top destinations chart shows which cities the AI has planned trips for most frequently. The status donut shows the pipeline from planning through confirmed to completed. Travel month distribution reveals seasonal patterns in trip planning.

**Key insights to highlight:**

- Bangkok and Barcelona are the most popular destinations across all users
- The confirmed/planning/completed split shows the trip lifecycle
- Most trips are 3-5 days, reflecting the typical city break pattern the personas requested

![Trip Planning](media/report_page4_trip_planning.png)

---

### Page 5: Destination Intelligence

This page answers: **What places does the system know about, and which ones does it recommend most?**

The places by city chart shows the inventory of hotels, restaurants, and activities available in each of the 49 supported cities. The most recommended places table reveals which specific venues the AI recommends most frequently across all trip itineraries — this is the AI's "favorites" list, influenced by user preferences and place quality.

**Key insights to highlight:**

- Each city has a balanced inventory of hotels, restaurants, and activities (~20 each across 49 cities = ~2,900 total)
- The most recommended places are the ones that best match the diverse preferences of all users
- Some places appear in multiple users' itineraries, showing the AI's confidence in those recommendations

![Destination Intelligence](media/report_page5_destination_intelligence.png)

---

## Step 5: SQL Endpoint Queries (Optional)

The file `sql_endpoint_queries.sql` contains ready-to-run queries for the Fabric SQL Analytics Endpoint. These are useful for:

- Quick exploration of the mirrored data before running the notebook
- Power BI DirectQuery connections for simple flat-field analytics
- Debugging data quality issues

Open the SQL Analytics Endpoint in your Fabric workspace and paste queries from the file.

> **Limitation:** SQL queries cannot easily flatten deeply nested JSON arrays (e.g., Trips `days[]`). Use the Spark notebook for those transformations.

---

## File Reference

| File | Purpose |
|------|---------|
| `data_generator.py` | Generates 12 users with 10+ turn conversations, creates trips, and updates trip statuses |
| `data_enricher.py` | Adds preference-conflict conversations to trigger memory supersession (run after generator) |
| `TravelAgentAnalyticsNotebook.ipynb` | Fabric Spark notebook -- reads mirrored data, flattens JSON, writes analytical Delta tables |
| `sql_endpoint_queries.sql` | SQL queries for the Fabric SQL Analytics Endpoint (optional) |
| `TravelAnalyticsReport.pbix` | Pre-built Power BI report (export from Fabric and add to repo) |
| `.venv/` | Python virtual environment for the scripts (httpx) |
| `README.md` | This guide |
