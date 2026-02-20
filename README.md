# AIS SQL Agent (LangGraph + Postgres/PostGIS)

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

This notebook prepares a reduced, higher-quality AIS dataset from NOAA’s AIS archives. Because the raw AIS files are very large, the workflow focuses on a limited slice (first five days of January 2024), applies quality filters, and then reduces the dataset further by keeping only “high-frequency” vessels (vessels that appear many times in the data).

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

## Script: `sql_agent_v2.ipynb`

The end-to-end LangGraph workflow is implemented in `sql_agent_v2.ipynb` and is organized into phases.

### 1) Database Connectivity

The notebook loads environment variables and creates a PostgreSQL connection using `langchain_community.utilities.SQLDatabase`.

Key points:
- Connection details are read from environment variables (`DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME`)
- The database is scoped to the `ais_data` table via `include_tables=['ais_data']`
- Table info includes a small sample of rows to help the agent reason about the schema

---

### 2) Tool Definition

A single LangChain tool is defined:

- `execute_sql(query: str) -> str`

This tool executes the generated SQL against the database and returns the result (or an error string if execution fails). The agent uses this tool as the only mechanism for database interaction.

---

### 3) State Management

The notebook defines an `AgentState` (`TypedDict`) that carries data through the workflow, including:

- `question`: user input
- `schema_context`: schema/rules provided to the LLM
- `sql_query`: generated SQL (if required)
- `validation_status`: `VALID` / `INVALID`
- `critique`: validator feedback or DB error details
- `query_result`: raw query output
- `final_response`: user-facing response
- `retry_count`: number of fix attempts

---

### 4) Schema Context & Translation Rules

A dedicated schema prompt block (`AIS_SCHEMA_INFO`) provides:
- Column definitions for `ais_data`
- Mappings for AIS `status` codes
- Mappings for `vesseltype` codes (including ranges for categories like cargo, tanker, passenger)
- Critical translation rules, including:
  - Converting natural language vessel categories/statuses into numeric filters
  - Excluding `heading = 511` when calculating heading statistics or filtering for valid headings
- A few-shot set of example questions and expected SQL patterns

This schema context is injected into the agent’s system prompt to reduce hallucinations and enforce consistent query generation.

---

### 5) Node Implementations

The workflow is composed of five nodes:

#### A) Agent (`agent_node`)
- Determines whether to respond conversationally or generate a SQL query
- Uses the schema/rules context to generate SQL
- Binds the `execute_sql` tool but does not force tool usage (the model chooses)

Outputs:
- If conversational: sets `final_response`
- If DB needed: sets `sql_query` and clears `final_response`

#### B) Validator (`validator_node`)
- Reviews the generated SQL for unsafe operations
- Rejects queries containing dangerous keywords such as:
  - `DROP`, `DELETE`, `INSERT`, `UPDATE`, `ALTER`

Outputs:
- `validation_status` and optional `critique`

#### C) Executor (`executor_node`)
- Runs the SQL using the `execute_sql` tool
- Converts database errors into a failure state for retry logic

Outputs:
- `query_result` and optional error `critique`

#### D) Fixer (`fixer_node`)
- Activated when validation fails or execution errors occur
- Provides the question, broken query, error, and schema to the LLM
- Forces tool usage (`tool_choice="required"`) to ensure a corrected SQL query is produced

Outputs:
- Updated `sql_query`
- Incremented `retry_count`

#### E) Synthesizer (`synthesizer_node`)
- Converts the raw SQL output into a clean, user-facing explanation
- If results are empty, states that no vessels matched
- If errors persist after retries, returns a generic failure response

Outputs:
- `final_response`

---

### 6) Graph Construction & Routing

The notebook uses `langgraph.graph.StateGraph` to define execution flow and conditional routing.

Routing behavior:
- If the agent decides no SQL is needed → end immediately
- Otherwise:
  - `validator` → `executor` → `synthesizer`
- On invalid SQL or execution failure:
  - route to `fixer` and retry validation/execution
- Retries are capped at 3 attempts (`retry_count <= 3`)

The final compiled app is produced via:

```python
app = workflow.compile()
```

---

### 7) Sample Execution Phase

The notebook includes a small driver function to run example questions through the compiled graph, print the agent’s final response, and catch unexpected runtime errors.

```python
# --- PHASE 6: EXECUTION ---

def run_query(query: str):
    print(f"\nUser: '{query}'")
    initial_state = {"question": query, "retry_count": 0}
    
    try:
        final_state = app.invoke(initial_state)
        print("\n" + "="*50)
        print("🤖 Final Answer:")
        print(final_state.get("final_response", ""))
        print("="*50)
    except Exception as e:
        print(f"\nAn error occurred: {e}")

if __name__ == "__main__":
    # Test 1: Database Query (Triggers SQL pipeline)
    run_query("Get 5 unique vessel names and their mmsi numbers that have length greater than 120 meter")
```

---

## Data Model Assumptions

Across the project, the expected data includes:
- AIS telemetry fields such as position, speed, course, heading, vessel identifiers, and vessel attributes
- Sentinel/error values (e.g., `LAT=91`, `LON=181`, `Heading=511`) that must be filtered for higher-quality analysis
- Vessel identifiers (`MMSI`) used both for database querying and frequency-based dataset reduction