## AIS SQL Agent (LangGraph + Postgres/PostGIS)

This project provides an AI assistant that answers maritime AIS telemetry questions by translating natural-language requests into safe, read-only SQL queries against a PostgreSQL/PostGIS database. It is designed to work specifically with an `ais_data` table containing vessel position reports and related metadata.

At a high level, the assistant:
- Decides whether a question requires database access or can be answered conversationally
- Generates SQL only when needed (targeting the `ais_data` table)
- Validates SQL for unsafe operations before running it
- Executes the query and returns the result
- Retries with an automatic “fix” loop when a query fails
- Summarizes raw query output into a clear, natural-language response

---
## Script: `ais_data_filter.ipynb`

This notebook prepares a reduced, higher-quality AIS dataset from NOAA’s AIS archives (https://coast.noaa.gov/htdata/CMSP/AISDataHandler/2024/index.html). Because the raw AIS files are very large, the workflow focuses on a limited slice (first five days of January 2024), applies quality filters, and then reduces the dataset further by keeping only “high-frequency” vessels (vessels that appear many times in the data).

### Data Source

The raw AIS CSV files are downloaded from NOAA’s AIS data archive (2024). The notebook assumes daily files such as:

- `AIS_2024_01_01.csv`
- `AIS_2024_01_02.csv`
- `AIS_2024_01_03.csv`
- `AIS_2024_01_04.csv`
- `AIS_2024_01_05.csv`

### Phase A: Quality Filtering (Row-Level Cleaning)

The first stage removes unreliable, default, or low-signal telemetry points to keep only meaningful vessel movement data.

Quality filters applied:

- **Valid coordinates**
  - Keeps rows where `LAT` is within ±90 and `LON` is within ±180
  - Explicitly removes AIS error codes like `LAT = 91` and `LON = 181`

- **Moving vessels**
  - Keeps rows where `SOG > 0.5` knots to reduce docked/anchored points and GPS jitter

- **Valid draft**
  - Keeps rows where `Draft > 0` to ensure vessels have configured transponder details

- **Valid heading**
  - Removes rows where `Heading = 511` (AIS “Not Available” sentinel)

These masks are combined into a single `quality_mask`, producing a cleaned dataframe (e.g., `df_clean` / `df_quality`). The notebook prints how many rows were removed and retained after cleaning.

### Phase B: Frequency Analysis (Choosing a Cutoff)

Before committing to a reduction threshold, the notebook computes how many rows and vessels remain at multiple “minimum appearance” cutoffs (counts per `MMSI`). Example cutoffs include:

- `>= 1, 5, 10, 50, 100, 200, 500, 1000`

For each cutoff, it reports:
- Total rows kept (sum of counts for qualifying MMSIs)
- Percent of the cleaned data retained
- Number of unique vessels retained

This is used to select a cutoff that meaningfully reduces size while preserving vessels with enough track history.

### Phase C: Frequency Filtering (Dataset Reduction)

After selecting a threshold (commonly `frequency_cutoff = 1000`), the notebook removes vessels that appear fewer than the cutoff number of rows in the already-cleaned dataset:

- Group by `MMSI`
- Keep only MMSIs whose count is `>= frequency_cutoff`

The output is a smaller dataset consisting of “active” vessels with richer movement history, and it reports:
- Rows removed by frequency cutoff
- Final row count
- Unique MMSIs remaining

### Phase D: Multi-Day Processing & Final Dataset Build

The notebook is designed to be run repeatedly for each day’s raw AIS file (Jan 01 → Jan 05), producing per-day cleaned outputs such as:

- `AIS_2024_01_01_cleaned_HighFreq.csv` (pattern)
- ...
- `AIS_2024_01_05_cleaned_HighFreq.csv`

After generating the cleaned daily outputs, the cleaned subsets can be combined into one “ultimate dataset” covering all five days.

> Note: The notebook content shows the per-day cleaning/reduction steps and describes running the process multiple times by swapping `input_file` / `output_file`. The final merge step can be implemented by concatenating the cleaned daily CSVs into a single dataframe and exporting it.

---

## Script: `upload_to_postgresql.ipynb`

This notebook uploads the final combined AIS dataset into a Supabase-hosted PostgreSQL database with PostGIS enabled, preparing it for spatial and temporal queries from the agent.

What it does:
- Connects to Supabase (PostgreSQL) using a SQLAlchemy engine
- Loads the final combined CSV into a Pandas DataFrame
- Parses the `BaseDateTime` column into a proper timestamp datatype
- Creates a PostGIS-compatible `geometry` column from `LAT`/`LON` as point features (EPSG:4326)
- Renames all columns to lowercase to avoid PostgreSQL case-sensitivity issues (e.g., `MMSI` → `mmsi`)
- Uploads the dataset into the `ais_signals` table, replacing any existing table/data

Implementation notes (as reflected in the script):
- The geometry conversion uses GeoPandas (`GeoDataFrame` + `points_from_xy`)
- The upload uses `to_postgis(...)` with `if_exists="replace"` and chunked inserts (`chunksize=1000`)
- After upload, the script prints the final column list (lowercased) for a quick sanity check

---

## Script: `sql_agent_v9.ipynb` (Version 1.6)

The end-to-end LangGraph workflow is implemented in `sql_agent_v1_6.ipynb`. The agent runs as a ReAct-style loop where the **agent node owns synthesis** and iterates through **validate → execute → heal** until it has enough tool evidence to answer.

### 1) Database Connectivity

The notebook loads environment variables and creates a PostgreSQL connection using both a raw **SQLAlchemy engine** and a **LangChain `SQLDatabase` wrapper** — both sharing the same connection string.

Key points:
- Connection details are read from environment variables (`DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME`)
- `get_engine()` creates a shared `SQLAlchemy` engine used by Pandas and the executor tools
- `SQLDatabase` is constructed from the shared engine, scoped to the `ais_data` table via `include_tables=['ais_data']`
- Table info includes a small sample of rows to help the agent reason about the schema

---

### 2) Tool Definitions

Two LangChain tools are defined:

- **`execute_sql(query: str) -> str`** — Executes a PostgreSQL query using `pd.read_sql(query, engine)` and returns the result as a formatted, header-included text table (`df.to_string(index=False)`). Returns an empty string `""` for empty result sets and an `ERROR:` prefixed string on failure.

- **`generate_csv(query: str) -> str`** — Executes the same query via Pandas, but additionally saves the result as a timestamped, UUID-suffixed `.csv` file under an `exports/` directory. Returns both the saved file path and the formatted table string so the LLM retains data context for synthesis. Use this tool **only** when the user explicitly requests an export, download, report, or CSV file.

The agent selects between the two tools based on the user's intent. All other nodes use `execute_sql` as the default fallback.

---

### 3) State Management

The notebook defines an `AgentState` (`TypedDict`) focused on a message-first ReAct trace:

- `messages`: full conversation trace (Human → AI tool-call → ToolMessage → AI answer)
- `schema_context`: schema/rules provided to the LLM
- `sql_query`: generated SQL (if required)
- `current_tool_call_id`: tool call ID used to bind SQL outputs back into history
- `current_tool_name`: name of the tool selected by the agent (`"execute_sql"` or `"generate_csv"`)
- `validation_status`: `VALID` / `INVALID`
- `critique`: validator feedback or DB error details
- `retry_count`: number of fix attempts

> The agent relies on `messages` as the source of truth for the full ReAct trace. `current_tool_name` is a new field that enables the executor to dynamically dispatch to the correct tool rather than always invoking `execute_sql`.

---

### 4) Schema Context & Translation Rules

A dedicated schema prompt block (`AIS_SCHEMA_INFO`) is structured as **XML** (`<database>`, `<schema>`, `<critical_rules>`, `<examples>`) and provides:
- Column definitions for `ais_data`
- Mappings for AIS `status` codes
- Mappings for `vesseltype` codes (including ranges for categories like cargo, tanker, passenger)
- Critical translation rules, including:
  - Converting natural language vessel categories/statuses into numeric filters
  - Excluding `heading = 511` when calculating heading statistics or filtering for valid headings
  - "Current state" deduplication using `DISTINCT ON (mmsi)` + `ORDER BY basedatetime DESC`
  - Exact uppercase equality for vessel name lookups (`WHERE vesselname = 'NAME'`); no `ILIKE` unless explicitly requested
  - Geometry column safety: the raw `geometry` column and `ST_AsText(geometry)` are banned from the final `SELECT` output; `lat` and `lon` must be used for location output instead
  - Predictive projection using `ST_Project`, with correct knots-to-meters-per-second conversion
  - Result size guard: always append `LIMIT 10` unless the user requests an aggregation or uses "all"
- A few-shot set of example questions and expected SQL patterns (including distance queries and CTE-based deduplication)

This schema context is injected into the agent's system prompt to reduce hallucinations and enforce consistent query generation.

---

### 5) Node Implementations (ReAct Loop)

The workflow is composed of four nodes:

#### A) Agent (`agent_node`)
- Prunes message history using a **sliding-window trim** before calling the LLM
  - Uses `trim_messages(...)` with:
    - `allow_partial=False` to avoid splitting tool-calls from ToolMessages
    - `start_on="human"` to ensure the window begins with a user message
- **Max-retry short-circuit:** If `retry_count > 3`, bypasses the normal reasoning prompt entirely and invokes a dedicated apology prompt (no tools bound) that generates a graceful failure message → `END`
- Decides whether to respond conversationally, call `execute_sql`, or call `generate_csv`
- Instructs the LLM to prefer a single SQL query for multi-part related questions (using window functions or multi-column SELECTs) before considering separate tool calls
- Both tools are bound via `llm.bind_tools([execute_sql, generate_csv])`

Outputs:
- If DB needed: sets `sql_query`, `current_tool_call_id`, `current_tool_name`, resets `retry_count`
- If conversational or max-retry apology: returns a normal assistant message and ends

#### B) Validator (`validator_node`)
- Reviews the generated SQL for unsafe operations and schema violations
- Rejects queries containing dangerous keywords such as:
  - `DROP`, `DELETE`, `INSERT`, `UPDATE`, `ALTER`
- Rejects hallucinated column names and invalid typing of `status` / `vesseltype`

Outputs:
- `validation_status` and optional `critique`

#### C) Executor (`executor_node`)
- Reads `current_tool_name` from state and dynamically dispatches to either `generate_csv` or `execute_sql`
- On success (non-empty result), returns a `ToolMessage` containing the formatted output **bound to the original `tool_call_id`**
- On empty result, returns a "not found" `ToolMessage` with an apology instruction — skipping the fixer to avoid unnecessary retries for valid queries that simply return no data
- On DB/tool error, converts failure into `critique` for the fixer

Outputs:
- `ToolMessage` (success or not-found) or error `critique`

#### D) Fixer (`fixer_node`)
- Activated when validation fails or execution errors occur
- Forces tool usage (`tool_choice="required"`) with both tools bound (`[execute_sql, generate_csv]`) to produce corrected SQL
- Retries are capped at 3 attempts; after that, it returns a `ToolMessage` error so the agent can respond naturally

Outputs:
- Updated `sql_query`
- Incremented `retry_count`

---

![AIS SQL Agent Graph Diagram](agent_graph_diagram_v8.png)

### 6) Graph Construction & Routing

The notebook uses `langgraph.graph.StateGraph` to define execution flow and conditional routing.

Routing behavior:
- Agent produces a tool call (`execute_sql` or `generate_csv`) → `validator` → (`executor` or `fixer`)
- Successful execution loops back to the **agent** (ReAct: reason → tool → observe → reason)
- Invalid SQL or execution failure routes to `fixer` and retries validation/execution
- Retries are capped at 3 attempts; persistent failures are surfaced back to the agent via a ToolMessage error
- If `retry_count > 3` when the agent node is re-entered, the agent short-circuits directly to an apology response without further tool calls

The final compiled app is produced via:

```python
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

---

---

---

## Script: `gradio_sql_agent_v9_refined.ipynb` (Gradio UI based on file sql_agent_v9.ipynb)

This script provides a **web chat interface** for interacting with the AIS SQL Agent (v1.6) using **Gradio**.

What it does:
- Builds the full LangGraph **ReAct** agent (validate → execute → heal with `MemorySaver` checkpointing), applying all v1.6 logic including dual-tool support (`execute_sql` and `generate_csv`), Pandas-based query execution, and XML-structured schema prompting
- Launches a Gradio **Blocks** chat UI (MARIA v2.0) for natural-language AIS questions
- Uses **per-user session isolation** by binding Gradio's `request.session_hash` to LangGraph's `thread_id`, so each browser session has its own conversation memory
- Appends messages in Gradio's `{"role": "...", "content": "..."}` format and displays the agent's final synthesized response in the chat
- Exposes a **CSV Download button** that automatically appears when the user requests an export — the file path is read directly from the LangGraph state and served via `gr.DownloadButton`

Notes:
- Requires the same environment variables as the agent (`DB_*`, `API_KEY`)
- Uses `share=True` in `demo.launch(...)` to create a public Gradio link (remove `share=True` for local-only deployment)
- CSV exports are saved to the `exports/` directory (auto-created on startup) with timestamped, UUID-suffixed filenames

![MARIA Web App Interface](web_app_interface.png)

---


## Data Model Assumptions

Across the project, the expected data includes:
- AIS telemetry fields such as position, speed, course, heading, vessel identifiers, and vessel attributes
- Sentinel/error values (e.g., `LAT=91`, `LON=181`, `Heading=511`) that must be filtered for higher-quality analysis
- Vessel identifiers (`MMSI`) used both for database querying and frequency-based dataset reduction