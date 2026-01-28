# Architecture & Workflow - Complete System Guide

## System Overview

The **Agentic RAG NL2SQL** system is an intelligent question-answering platform that converts natural language queries into SQL and formats results as interactive visualizations. It uses a multi-agent orchestration architecture with semantic understanding, retrieval-augmented generation (RAG), and agentic retry logic.

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER INTERFACE (Web)                       │
│  HTML/CSS/JS - Interactive Chat + Chart/Table Display          │
└───────────────────┬─────────────────────────────────────────────┘
                    │ HTTP POST (natural language question)
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FLASK APPLICATION                            │
│  app.py - REST API endpoint, request routing, response handling │
└───────────────────┬─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│              AGENT ORCHESTRATOR (Main Controller)               │
│  Coordinates all agents, manages workflow, error handling       │
└───────────────────┬─────────────────────────────────────────────┘
                    │
     ┌──────┬──────┬┴────┬──────┬──────────────────┐
     │      │      │     │      │                  │
     ▼      ▼      ▼     ▼      ▼                  ▼
  AGENT1 AGENT2 AGENT3 AGENT4 AGENT5           AGENT6+
  (See detailed agent flow below)
```

---

## 🔄 Complete End-to-End Workflow

### Phase 1: User Input → Question Understanding

```
┌─────────────────────────────────────────────────────────────────┐
│ USER SUBMITS QUESTION                                           │
│ Example: "Which item has the highest total ordered quantity?"  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ AGENT ORCHESTRATOR                                              │
│ - Receives question from Flask API                              │
│ - Initializes all agents                                        │
│ - Logs request                                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ SEMANTIC ANALYZER AGENT                                         │
│                                                                 │
│ INPUT: "Which item has highest total ordered quantity?"        │
│                                                                 │
│ PROCESS: Analyze intent across 5 semantic dimensions:          │
│   1. TABLE_INTENT: master | transactional | mixed              │
│      → "quantity ordered" = transactional (po_items table)    │
│                                                                 │
│   2. RESULT_CARDINALITY: singular | plural | unknown           │
│      → "which" keyword = singular (return 1 result)           │
│                                                                 │
│   3. AGGREGATION_TYPE: stored | derived | none                 │
│      → "total ordered quantity" = derived (SUM aggregate)     │
│                                                                 │
│   4. NULL_HANDLING: preserve | default | aggregate             │
│      → Aggregation context = aggregate NULL handling          │
│                                                                 │
│   5. ENTITY_SCOPE: all | referenced | unknown                  │
│      → "ordered" implies only items in transactions           │
│                                                                 │
│ OUTPUT:                                                         │
│   {                                                             │
│     "table_intent": "transactional",                           │
│     "result_cardinality": "singular",                          │
│     "aggregation_type": "derived",                             │
│     "null_handling": "aggregate",                              │
│     "entity_scope": "referenced"                               │
│   }                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ PLANNER AGENT                                                   │
│                                                                 │
│ INPUT: Original question + semantic context                    │
│                                                                 │
│ PROCESS: Create execution plan                                 │
│   1. Determine if RAG is needed (yes - domain knowledge)       │
│   2. Identify key entities: "item", "quantity", "ordered"      │
│   3. Estimated complexity: HIGH (requires JOIN + GROUP BY)     │
│   4. Plan: Retrieve schema → Generate SQL → Execute            │
│                                                                 │
│ OUTPUT: { "use_rag": true, "entities": [...], "complexity": 3 }│
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        Phase 2: RAG Retrieval (below)
```

---

### Phase 2: Data Retrieval with RAG (Embeddings + Vector Search)

```
┌─────────────────────────────────────────────────────────────────┐
│ RETRIEVAL AGENT (RAG Component)                                 │
│                                                                 │
│ STEP 1: EMBEDDING GENERATION                                   │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ Question: "Which item has highest total ordered quantity?" │
│ │                                                             │
│ │ Embedding Model: BAAI/bge-small-en-v1.5                    │
│ │ (384-dimensional vector)                                   │
│ │                                                             │
│ │ Process: Text → Tokenize → BERT-like encoder → Vector     │
│ │ Output: [0.234, -0.456, 0.789, ... (384 values)]          │
│ └─────────────────────────────────────────────────────────────┘
│
│ STEP 2: VECTOR DATABASE SEARCH (Qdrant)                       │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ Qdrant Storage Structure:                                  │
│ │                                                             │
│ │ Collection: "db_schema_metadata"                           │
│ │ ├── Vector Field: embedding (384-dim)                     │
│ │ ├── Payload Fields:                                        │
│ │ │   ├── table_name (po, po_items, items)                 │
│ │ │   ├── column_name (id, name, quantity, etc)            │
│ │ │   ├── column_type (INT, VARCHAR, DECIMAL, etc)         │
│ │ │   ├── table_description                                │
│ │ │   ├── column_description                               │
│ │ │   └── business_context                                 │
│ │                                                             │
│ │ Search Process:                                            │
│ │ 1. Calculate semantic similarity (cosine distance)         │
│ │ 2. Find K nearest neighbors (top-5 matches)               │
│ │ 3. Rank by relevance score                                │
│ └─────────────────────────────────────────────────────────────┘
│
│ STEP 3: RETRIEVE RELEVANT SCHEMA CHUNKS                       │
│ Qdrant returns (scored by relevance):                         │
│   1. po_items.requested_quantity (score: 0.95)              │
│   2. items.name (score: 0.93)                               │
│   3. po_items.parent_po_id (score: 0.87)                    │
│   4. items.id (score: 0.85)                                 │
│   5. po.status (score: 0.72)                                │
│                                                               │
│ Full Context Retrieved:                                       │
│ ├── Table: po_items                                           │
│ │   ├── id: INT (primary key)                               │
│ │   ├── parent_po_id: INT (FK to po)                        │
│ │   ├── item_id: INT (FK to items)                          │
│ │   ├── requested_quantity: INT (quantity ordered)          │
│ │   └── per_unit_rate: DECIMAL                              │
│ ├── Table: items                                              │
│ │   ├── id: INT (primary key)                               │
│ │   ├── name: VARCHAR (item name)                           │
│ │   ├── code_sku: VARCHAR                                   │
│ │   └── base_price: DECIMAL                                 │
│ └── Table: po                                                 │
│     ├── id: INT                                              │
│     ├── po_no: VARCHAR                                       │
│     └── status: VARCHAR                                       │
│                                                               │
│ OUTPUT: Ranked schema chunks + business context              │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        Phase 3: SQL Generation (below)
```

---

### Phase 3: SQL Generation with Semantic Rules

```
┌─────────────────────────────────────────────────────────────────┐
│ SQL GENERATOR AGENT                                             │
│                                                                 │
│ INPUTS:                                                         │
│ • Question: "Which item has highest total ordered quantity?"   │
│ • Semantic Context: {table_intent: transactional,             │
│                      result_cardinality: singular,             │
│                      aggregation_type: derived}                │
│ • Schema Chunks: [po_items, items, po tables]                 │
│                                                                 │
│ PROCESS:                                                        │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ STEP 1: SEMANTIC RULE INJECTION INTO LLM PROMPT             │
│ │                                                             │
│ │ Semantic rules added to LLM:                               │
│ │                                                             │
│ │ "SEMANTIC RULES FOR THIS QUESTION:                         │
│ │ - This question is about TRANSACTIONS (orders, purchases). │
│ │   ➡️  Query PO and PO_ITEMS tables.                        │
│ │   ➡️  Join ITEMS only for labels/names.                    │
│ │                                                             │
│ │ - User expects ONE result (top item, highest).             │
│ │   ➡️  Use ORDER BY ... DESC LIMIT 1                       │
│ │   ➡️  Return only the single best match.                   │
│ │                                                             │
│ │ - User is asking for COMPUTED/DERIVED values.              │
│ │   ➡️  Use aggregation functions (SUM, AVG, COUNT).        │
│ │   ➡️  Base calculations on transactions."                  │
│ │                                                             │
│ │ Full prompt includes:                                       │
│ │ • Query examples from schema                                │
│ │ • Procurement domain specifics                              │
│ │ • JOIN instructions                                         │
│ │ • GROUP BY + ORDER BY patterns                              │
│ │ • Soft-delete filter requirements                           │
│ └─────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ STEP 2: LLM CALLS (Ollama - Llama2/Llama3)                  │
│ │                                                             │
│ │ LLM Provider: Ollama (localhost:11434)                     │
│ │ Model: llama2 (7B) or Llama 3 (8B)                         │
│ │ Temperature: 0.3 (lower for consistency)                   │
│ │ Timeout: 120 seconds                                        │
│ │                                                             │
│ │ LLM Response:                                               │
│ │ "SELECT i.name AS item_name,                              │
│ │         SUM(poi.requested_quantity) AS total_quantity     │
│ │  FROM po_items poi                                         │
│ │  JOIN items i ON poi.item_id = i.id                       │
│ │  WHERE poi.is_deleted IS NULL OR poi.is_deleted = false   │
│ │  GROUP BY i.id, i.name                                     │
│ │  ORDER BY total_quantity DESC                              │
│ │  LIMIT 1;"                                                  │
│ │                                                             │
│ │ Post-Processing:                                            │
│ │ • Extract SQL from potential preambles                      │
│ │ • Clean whitespace and formatting                          │
│ │ • Ensure semicolon termination                             │
│ └─────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ STEP 3: FALLBACK PATTERN GENERATION (If LLM Fails)          │
│ │                                                             │
│ │ Fallback System Structure:                                  │
│ │                                                             │
│ │ if table_intent == "transactional":                        │
│ │    Route to: _generate_transactional_query()              │
│ │    Checks for:                                              │
│ │    • Specific keywords (frequency, most ordered, etc)      │
│ │    • Aggregation patterns (COUNT, SUM, GROUP BY)           │
│ │    • Time-based queries (recent, latest)                   │
│ │    • Status filters (approved, draft, pending)             │
│ │                                                             │
│ │ elif table_intent == "master":                             │
│ │    Route to: _generate_master_data_query()                │
│ │    Checks for:                                              │
│ │    • Stored value queries (price, cost)                    │
│ │    • Item property queries (SKU, name)                     │
│ │    • Catalog/inventory queries                              │
│ │                                                             │
│ │ For this question (transactional):                         │
│ │ • Matches: "highest" + "quantity" + "item"                │
│ │ • Pattern: Frequency/ranking query                         │
│ │ • Fallback Generated:                                       │
│ │   SELECT i.name, COUNT(poi.id) as frequency               │
│ │   FROM po_items poi                                        │
│ │   JOIN items i ON poi.item_id = i.id                      │
│ │   WHERE poi.is_deleted IS NULL OR poi.is_deleted = false  │
│ │   GROUP BY i.id, i.name                                   │
│ │   ORDER BY frequency DESC LIMIT 1;                         │
│ └─────────────────────────────────────────────────────────────┘
│
│ OUTPUT: Final SQL Query
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        Phase 4: SQL Validation & Execution (below)
```

---

### Phase 4: SQL Validation & Execution with Retry Logic

```
┌─────────────────────────────────────────────────────────────────┐
│ SQL VALIDATOR AGENT                                             │
│                                                                 │
│ INPUT: Generated SQL                                            │
│                                                                 │
│ VALIDATION CHECKS:                                              │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ CHECK 1: SYNTAX VALIDATION                                 │
│ │ • Must start with SELECT or WITH                           │
│ │ • Must end with semicolon                                  │
│ │ • Reject if contains non-SELECT commands                   │
│ │ • Example: Reject "DROP TABLE", "INSERT", "UPDATE"         │
│ └─────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ CHECK 2: SECURITY VALIDATION                                │
│ │ • Reject if references system tables:                       │
│ │   - information_schema.*                                    │
│ │   - pg_catalog.*                                            │
│ │   - pg_tables, pg_columns                                   │
│ │ • Reject if SQL injection patterns detected                 │
│ │ • Reject if DROP, DELETE, ALTER detected                   │
│ └─────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ CHECK 3: COLUMN EXISTENCE VALIDATION                        │
│ │ • Connect to database                                        │
│ │ • Query information_schema for actual columns               │
│ │ • Verify all referenced columns exist                       │
│ │ • Check column types match usage (numeric for SUM, etc)     │
│ │ • Validate table existence                                  │
│ └─────────────────────────────────────────────────────────────┘
│
│ VALIDATION RESULT:
│ ✓ All checks pass → proceed to execution
│ ✗ Check fails → Trigger RETRY LOGIC (see below)
│
│ RETRY LOGIC (Agentic Retry):
│ ┌─────────────────────────────────────────────────────────────┐
│ │ Max Retries: 5                                              │
│ │ Retry Strategy:                                             │
│ │                                                             │
│ │ Attempt 1: LLM with enhanced prompt                        │
│ │ Attempt 2: LLM with column names explicitly listed         │
│ │ Attempt 3: LLM with example queries                        │
│ │ Attempt 4: Semantic fallback pattern                       │
│ │ Attempt 5: Simple fallback (list all columns)              │
│ │                                                             │
│ │ Each attempt:                                               │
│ │ 1. Pass error details to agent                              │
│ │ 2. LLM generates corrected SQL                             │
│ │ 3. Re-validate                                              │
│ │ 4. If success → Execute                                     │
│ │ 5. If fail → Continue to next attempt                       │
│ │                                                             │
│ │ If all retries fail → Return error to user                 │
│ └─────────────────────────────────────────────────────────────┘
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ SQL EXECUTOR AGENT                                              │
│                                                                 │
│ INPUT: Validated SQL                                            │
│                                                                 │
│ EXECUTION:                                                      │
│ • Database: PostgreSQL (procure_phase1_first3tables)           │
│ • Connection: psycopg2 (binary driver)                         │
│ • Timeout: 30 seconds per query                                │
│ • Fetch Size: All rows (configurable)                          │
│                                                                 │
│ PROCESS:                                                        │
│ 1. Establish connection with credentials from .env             │
│ 2. Parse SQL into executable form                              │
│ 3. Execute query                                               │
│ 4. Fetch results into memory                                   │
│ 5. Convert to Python dict/list format                          │
│ 6. Handle errors (connection, timeout, etc)                    │
│                                                                 │
│ OUTPUT: Raw Results                                             │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ [                                                           │
│ │   {                                                         │
│ │     "item_name": "Laptop Computer",                        │
│ │     "total_quantity": 45                                   │
│ │   }                                                         │
│ │ ]                                                           │
│ │                                                             │
│ │ (Only 1 row due to LIMIT 1 from singular cardinality)     │
│ └─────────────────────────────────────────────────────────────┘
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        Phase 5: Response Formatting & Visualization (below)
```

---

### Phase 5: Response Formatting & Data Transformation

```
┌─────────────────────────────────────────────────────────────────┐
│ RESPONSE FORMATTER AGENT                                        │
│                                                                 │
│ INPUT: Raw SQL results                                          │
│                                                                 │
│ PROCESS:                                                        │
│                                                                 │
│ STEP 1: DATA CLEANING & VALIDATION                             │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ • Check for empty results                                  │
│ │ • Handle NULL values (convert to empty string or 0)        │
│ │ • Validate data types                                      │
│ │ • Format dates (ISO 8601)                                  │
│ │ • Format numbers (decimal precision)                       │
│ │ • Remove duplicate rows if any                             │
│ └─────────────────────────────────────────────────────────────┘
│
│ STEP 2: METADATA EXTRACTION                                    │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ From results, determine:                                    │
│ │ • Number of rows: 1                                        │
│ │ • Number of columns: 2                                     │
│ │ • Column names: ["item_name", "total_quantity"]           │
│ │ • Column types: [STRING, INTEGER]                          │
│ │ • Result summary: "1 item found"                           │
│ └─────────────────────────────────────────────────────────────┘
│
│ STEP 3: VISUALIZATION RECOMMENDATION                          │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ Analyze result structure:                                   │
│ │                                                             │
│ │ • Row count: 1 → Single value/metric                       │
│ │ • Columns: 2 (name, quantity)                              │
│ │ • Data types: String + Number                              │
│ │                                                             │
│ │ Recommendation: KPI Card / Single Metric Display           │
│ │ (Alternative: Could show as bar chart if > 1 row)         │
│ │                                                             │
│ │ Visualization Logic:                                        │
│ │ IF row_count == 1:                                          │
│ │   IF all_numeric_columns:                                  │
│ │     → Single KPI card                                       │
│ │   ELSE:                                                     │
│ │     → Key-value display                                    │
│ │ ELIF row_count <= 10:                                       │
│ │   → Table display                                           │
│ │ ELIF numeric_column_exists AND category_column_exists:     │
│ │   → Bar/Pie chart                                           │
│ │ ELSE:                                                       │
│ │   → Table with sorting/filtering                           │
│ └─────────────────────────────────────────────────────────────┘
│
│ OUTPUT: Formatted data + visualization metadata
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ VISUALIZATION MAPPER AGENT                                      │
│                                                                 │
│ INPUT: Formatted data + visualization type                      │
│                                                                 │
│ PROCESS: Convert to Nivo.js compatible JSON                    │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ NIVO CHART LIBRARY                                          │
│ │ (https://nivo.rocks/)                                       │
│ │                                                             │
│ │ Supported Charts:                                           │
│ │ • Bar Chart (ResponsiveBar)                                │
│ │ • Line Chart (ResponsiveLine)                              │
│ │ • Pie Chart (ResponsivePie)                                │
│ │ • Table (responsive grid)                                  │
│ │ • Gauge (single value display)                             │
│ │ • Sankey (flow diagram)                                    │
│ │                                                             │
│ │ JSON Format Example (Bar Chart):                            │
│ │ {                                                           │
│ │   "chart_type": "bar",                                     │
│ │   "data": [                                                │
│ │     {"name": "Laptop Computer", "quantity": 45},          │
│ │   ],                                                        │
│ │   "config": {                                               │
│ │     "margin": {"top": 20, "right": 20},                   │
│ │     "colors": "#3498db",                                   │
│ │     "animate": true,                                       │
│ │     "indexBy": "name",                                     │
│ │     "keys": ["quantity"],                                  │
│ │     "responsive": true                                     │
│ │   }                                                         │
│ │ }                                                           │
│ └─────────────────────────────────────────────────────────────┘
│
│ TRANSFORMATION FOR THIS QUERY:
│ Input: [{"item_name": "Laptop Computer", "total_quantity": 45}]
│
│ Nivo JSON (Gauge Chart for KPI):
│ {
│   "chart_type": "gauge",
│   "data": {
│     "value": 45,
│     "label": "Laptop Computer"
│   },
│   "config": {
│     "valueFormat": " >-.0f",
│     "colors": ["#3498db", "#e74c3c"],
│     "radius": 0.8,
│     "arcPadding": 0.1,
│     "startAngle": -Math.PI * 1.2,
│     "endAngle": Math.PI * 0.2
│   }
│ }
│
│ Alternative Nivo JSON (Key-Value Display):
│ {
│   "chart_type": "table",
│   "data": [
│     {"key": "Item", "value": "Laptop Computer"},
│     {"key": "Total Quantity", "value": "45"}
│   ],
│   "config": {
│     "columns": [
│       {"id": "key", "label": "Metric", "width": "40%"},
│       {"id": "value", "label": "Value", "width": "60%"}
│     ]
│   }
│ }
│
│ OUTPUT: Nivo-compatible JSON + chart metadata
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        Phase 6: Frontend Rendering (below)
```

---

### Phase 6: Frontend Rendering & User Display

```
┌─────────────────────────────────────────────────────────────────┐
│ FRONTEND (HTML/CSS/JavaScript)                                  │
│                                                                 │
│ COMPONENTS:                                                     │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐
│ │ 1. CHAT INTERFACE (Left Panel)                              │
│ │    • Message history display                                │
│ │    • Input text field                                       │
│ │    • Send button                                            │
│ │    • Loading spinner during processing                      │
│ └─────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ 2. VISUALIZATION CONTAINER (Right Panel)                    │
│ │    • Dynamic chart rendering                                │
│ │    • Loading state                                          │
│ │    • Error message display                                  │
│ │    • Data table fallback                                    │
│ └─────────────────────────────────────────────────────────────┘
│
│ RENDERING FLOW:
│
│ 1. Receive Nivo JSON from backend
│ 2. Parse chart_type from JSON
│ 3. Import corresponding Nivo component:
│    import { ResponsiveBar } from '@nivo/bar'
│    import { ResponsiveGauge } from '@nivo/gauge'
│    etc.
│ 4. Render component with data and config
│ 5. Apply interactive features (hover, click, zoom)
│ 6. Display with animations
│
│ EXAMPLE RENDERING:
│ ┌─────────────────────────────────────────────────────────────┐
│ │                    GAUGE CHART OUTPUT                       │
│ │                                                             │
│ │                        45                                   │
│ │                      ╱────╲                                 │
│ │                    ╱        ╲                               │
│ │                  ╱            ╲                             │
│ │                 │  Laptop    │                             │
│ │                  ╲            ╱                             │
│ │                    ╲        ╱                               │
│ │                      ╲────╱                                 │
│ │                                                             │
│ │              Laptop Computer: 45                            │
│ │                                                             │
│ │ (Interactive: Hovering shows details)                      │
│ │ (Responsive: Resizes with window)                          │
│ │                                                             │
│ └─────────────────────────────────────────────────────────────┘
│
│ OUTPUT: Interactive visualization displayed to user
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
          USER SEES RESULT ON SCREEN
```

---

## 🤖 Agent Architecture Details

### Agent Types & Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT ORCHESTRATOR                           │
│  Coordinates workflow, manages state, routes between agents     │
├─────────────────────────────────────────────────────────────────┤
│ Methods:                                                         │
│ • orchestrate() - Main entry point                              │
│ • _execute_agents() - Sequential agent execution                │
│ • _handle_errors() - Error recovery                             │
│ • _manage_state() - Track execution state                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SEMANTIC ANALYZER AGENT                            │
│  Analyzes question intent across 5 semantic dimensions          │
├─────────────────────────────────────────────────────────────────┤
│ 5 Dimensions:                                                    │
│ 1. Table Intent (master vs transactional)                       │
│ 2. Result Cardinality (singular vs plural)                      │
│ 3. Aggregation Type (stored vs derived)                         │
│ 4. NULL Handling (preserve vs aggregate)                        │
│ 5. Entity Scope (all vs referenced)                             │
│                                                                 │
│ Input: Natural language question                                │
│ Output: Semantic context dict                                   │
│ Pattern Matching: 30+ linguistic regex patterns                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              PLANNER AGENT                                      │
│  Creates execution strategy for the question                    │
├─────────────────────────────────────────────────────────────────┤
│ Responsibilities:                                               │
│ • Analyze question complexity                                   │
│ • Determine if RAG needed                                       │
│ • Identify key entities and relationships                       │
│ • Create execution plan                                         │
│ • Estimate expected result format                               │
│                                                                 │
│ Output: Plan dict with strategy and metadata                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              RETRIEVAL AGENT (RAG)                              │
│  Fetches relevant schema from vector database                   │
├─────────────────────────────────────────────────────────────────┤
│ Process:                                                         │
│ 1. Embed question (BAAI/bge-small-en-v1.5)                     │
│ 2. Search Qdrant vector DB                                      │
│ 3. Retrieve top-K schema chunks (K=5)                           │
│ 4. Rank by relevance score                                      │
│ 5. Return full schema context                                   │
│                                                                 │
│ Database: Qdrant (port 6333)                                    │
│ Collection: db_schema_metadata                                  │
│ Vector Dimension: 384 (BAAI model)                              │
│                                                                 │
│ Output: Schema context string                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SQL GENERATOR AGENT                                │
│  Generates SQL from question + semantic context                 │
├─────────────────────────────────────────────────────────────────┤
│ Two Paths:                                                       │
│                                                                 │
│ Path 1: LLM-based (Primary)                                     │
│ • Call Ollama LLM with semantic rules                           │
│ • LLM generates SQL aware of intent                             │
│ • Success rate: ~85% for well-formed questions                  │
│                                                                 │
│ Path 2: Fallback (Backup)                                       │
│ • Apply rule-based query generation                             │
│ • Check keywords against fallback patterns                      │
│ • Generate SQL deterministically                                │
│ • Success rate: ~95% for known patterns                         │
│                                                                 │
│ LLM Details:                                                     │
│ • Provider: Ollama (localhost:11434)                            │
│ • Model: llama2 or llama3                                       │
│ • Temperature: 0.3 (for consistency)                            │
│ • Max Tokens: 512                                               │
│ • Timeout: 120 seconds                                          │
│                                                                 │
│ Output: SQL query string                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SQL VALIDATOR AGENT                                │
│  Validates SQL safety and correctness                           │
├─────────────────────────────────────────────────────────────────┤
│ 3 Validation Layers:                                             │
│ 1. Syntax: Must start with SELECT/WITH, end with ;             │
│ 2. Security: Reject system tables, DROP/DELETE/ALTER           │
│ 3. Semantics: Verify columns exist, types match usage           │
│                                                                 │
│ Error Detection:                                                │
│ • Missing column errors                                         │
│ • Invalid table references                                      │
│ • Type mismatch in operations                                   │
│ • GROUP BY clause issues                                        │
│                                                                 │
│ Output: Validation result (true/false) + error details          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SQL EXECUTOR AGENT                                 │
│  Executes validated SQL against PostgreSQL                      │
├─────────────────────────────────────────────────────────────────┤
│ Process:                                                         │
│ 1. Connect to database (psycopg2)                               │
│ 2. Parse and prepare SQL                                        │
│ 3. Execute with timeout (30 seconds)                            │
│ 4. Fetch all results into memory                                │
│ 5. Convert to Python dict/list                                  │
│ 6. Handle errors gracefully                                     │
│                                                                 │
│ Database: PostgreSQL                                            │
│ Connection: procure_phase1_first3tables (localhost:5432)        │
│ Driver: psycopg2 binary                                         │
│ Row Limit: Configurable (default 10000)                         │
│                                                                 │
│ Output: Result list of dictionaries                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              RESPONSE FORMATTER AGENT                           │
│  Formats raw SQL results for display                            │
├─────────────────────────────────────────────────────────────────┤
│ Process:                                                         │
│ 1. Clean data (handle NULL, format dates)                       │
│ 2. Extract metadata (columns, types, row count)                 │
│ 3. Determine best visualization                                 │
│ 4. Add summary/metadata                                         │
│ 5. Prepare for next agent                                       │
│                                                                 │
│ Output: Formatted data + visualization recommendation           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              VISUALIZATION MAPPER AGENT                         │
│  Converts data to Nivo.js compatible JSON                       │
├─────────────────────────────────────────────────────────────────┤
│ Process:                                                         │
│ 1. Receive formatted data                                       │
│ 2. Determine chart type based on data shape                     │
│ 3. Map columns to Nivo data format                              │
│ 4. Generate Nivo config (colors, margins, etc)                  │
│ 5. Create Nivo-compatible JSON                                  │
│                                                                 │
│ Chart Types:                                                     │
│ • Bar Chart (categorical data)                                  │
│ • Line Chart (time series)                                      │
│ • Pie Chart (proportions)                                       │
│ • Gauge Chart (single value KPI)                                │
│ • Table (raw data with sorting)                                 │
│                                                                 │
│ Output: Nivo JSON + metadata                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              VISUALIZATION AGENT                                │
│  Renders interactive charts on frontend                         │
├─────────────────────────────────────────────────────────────────┤
│ Technology: Nivo.js (React-based charting)                      │
│ Features:                                                        │
│ • Responsive design                                             │
│ • Interactive tooltips                                          │
│ • Smooth animations                                             │
│ • Theme support                                                 │
│ • Export capabilities                                           │
│                                                                 │
│ Output: Rendered interactive visualization                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔁 Agentic Retry & Error Recovery

### Retry Logic Flow

```
┌──────────────────────────────────────────────────────────────┐
│ SQL VALIDATION FAILS                                         │
│ Error: "column 'xyz' does not exist"                         │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ RETRY ATTEMPT 1 OF 5                                         │
│ Strategy: Enhanced LLM Prompt                                │
├──────────────────────────────────────────────────────────────┤
│ • Include full schema in prompt                              │
│ • Add actual column names from error                         │
│ • Provide example queries                                    │
│ • Re-call LLM with corrected context                         │
│ • Re-validate                                                │
│ • If success → Execute                                       │
│ • If fail → Continue to Attempt 2                            │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ RETRY ATTEMPT 2 OF 5                                         │
│ Strategy: Explicit Column Listing                            │
├──────────────────────────────────────────────────────────────┤
│ • List ALL available columns in prompt                       │
│ • Show exact table.column format required                    │
│ • Provide working queries as examples                        │
│ • Focus on column precision                                  │
│ • Re-call LLM                                                │
│ • If success → Execute                                       │
│ • If fail → Continue to Attempt 3                            │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ RETRY ATTEMPT 3 OF 5                                         │
│ Strategy: Example-Based Learning                             │
├──────────────────────────────────────────────────────────────┤
│ • Show 3-5 similar queries that work                         │
│ • Explain each query component                               │
│ • Ask LLM to follow exact pattern                            │
│ • Rephrase question focus                                    │
│ • Re-call LLM                                                │
│ • If success → Execute                                       │
│ • If fail → Continue to Attempt 4                            │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ RETRY ATTEMPT 4 OF 5                                         │
│ Strategy: Semantic Fallback Generation                       │
├──────────────────────────────────────────────────────────────┤
│ • Skip LLM entirely                                          │
│ • Use rule-based fallback patterns                           │
│ • Apply semantic context rules                               │
│ • Generate deterministic SQL                                 │
│ • Validate against schema                                    │
│ • If success → Execute                                       │
│ • If fail → Continue to Attempt 5                            │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ RETRY ATTEMPT 5 OF 5 (Last Resort)                           │
│ Strategy: Simple Fallback                                    │
├──────────────────────────────────────────────────────────────┤
│ • Generate simplest possible query                           │
│ • SELECT * FROM main_table LIMIT 20                          │
│ • Or list all from primary table                             │
│ • Accept partial answer rather than failure                  │
│ • If success → Execute                                       │
│ • If fail → Return error to user                             │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│ ALL RETRIES EXHAUSTED                                        │
│ Return error message to user with:                           │
│ • SQL that was attempted                                     │
│ • Error message from database                                │
│ • Suggestion to rephrase question                            │
│ • Example questions that work                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔄 Complete Request-Response Cycle

```
USER INPUT
   │
   ▼
Flask API receives request
   │
   ▼
Agent Orchestrator initialized
   │
   ├─→ Semantic Analyzer (5-dim intent)
   │
   ├─→ Planner (strategy creation)
   │
   ├─→ Retriever (RAG - schema chunks)
   │
   ├─→ SQL Generator (LLM or fallback)
   │     ├─→ Validation ─→ If Fail ─→ Retry Loop (up to 5x)
   │     └─→ Executor
   │
   ├─→ Response Formatter (clean + metadata)
   │
   ├─→ Visualization Mapper (Nivo JSON)
   │
   └─→ Return Nivo JSON to Frontend
         │
         ▼
      Frontend receives JSON
         │
         ▼
      Render Nivo Chart
         │
         ▼
      Display to User
```

---

## 📊 Data Flow Summary

```
Question Text
    ↓ (embedding)
384-dim Vector
    ↓ (Qdrant search)
Schema Chunks
    ↓ (with semantic context)
LLM Prompt
    ↓ (LLM inference)
SQL Query
    ↓ (validation + retry)
Validated SQL
    ↓ (PostgreSQL execution)
Result Rows
    ↓ (formatting + mapping)
Nivo JSON
    ↓ (frontend rendering)
Interactive Chart
```

---

## 🎯 Key Design Principles

1. **Semantic-First**: Intent classification before SQL generation
2. **Agentic Retry**: Intelligent retry logic with 5 escalating strategies
3. **RAG-Enhanced**: Vector search for relevant schema context
4. **LLM + Fallback**: LLM-first with deterministic backup patterns
5. **Type-Safe**: Validation at every step
6. **Error Recovery**: Graceful degradation with helpful error messages
7. **Interactive Visualization**: Automatic chart selection and Nivo rendering

---

## 📁 Code Organization

- **app.py** - Flask API entry point
- **agent_orchestrator.py** - Main workflow coordinator
- **agent_semantic_analyzer.py** - Intent classification (5 dimensions)
- **agent_planner.py** - Strategy creation
- **agent_retriever.py** - RAG/vector search
- **agent_sql_generator.py** - SQL generation (LLM + fallback)
- **agent_sql_validator.py** - SQL validation
- **agent_sql_executor.py** - Database execution
- **agent_response_formatter.py** - Result formatting
- **agent_visualization_mapper.py** - Nivo JSON conversion
- **agent_visualization.py** - Chart rendering
- **static/** - Frontend HTML/CSS/JS
- **db_metadata.txt** - Schema reference
- **qdrant_setup.py** - Vector DB initialization

---

**This architecture enables accurate, intelligent SQL generation with robust error handling and beautiful interactive visualizations.**

For setup instructions, see [SETUP_INSTALLATION_GUIDE.md](SETUP_INSTALLATION_GUIDE.md)
