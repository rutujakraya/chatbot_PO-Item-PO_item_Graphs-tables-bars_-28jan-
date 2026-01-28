# Architecture & Data Flow Diagram

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER CHATBOT UI                          │
│  "List the 5 most recently created purchase orders"             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                     (HTTP POST /query)
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      FASTAPI BACKEND                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. AgentOrchestrator.run(query)                          │  │
│  │    ├─ PlannerAgent: Intent detection                     │  │
│  │    ├─ RetrievalAgent: Fetch metadata                     │  │
│  │    ├─ SQLGeneratorAgent: Generate query                  │  │
│  │    ├─ SQLValidatorAgent: Validate SQL                    │  │
│  │    └─ SQLExecutorAgent: Execute SQL                      │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  Result: rows = [                                               │
│    ('PO-001', 'DRAFT', Decimal('5000.00'), datetime(...)),     │
│    ('PO-002', 'APPROVED', Decimal('6000.00'), datetime(...)),  │
│    ...                                                          │
│  ]                                                              │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 2. AgentOrchestrator._normalize_rows()  ✅ NEW            │  │
│  │    ├─ Decimal('5000.00') → 5000.0                         │  │
│  │    ├─ datetime(...) → '2026-01-22T00:00:00'              │  │
│  │    └─ Returns: rows_normalized = [[...], [...], ...]     │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 3. ResponseFormatterAgent.format()                       │  │
│  │    └─ text_answer = "Here are 5 purchase orders..."     │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 4. VisualizationMapper.map()  ✅ (Existing)              │  │
│  │    ├─ Detects chart_type = "table"                       │  │
│  │    ├─ Structures visualization JSON                      │  │
│  │    └─ Returns: {chart_type, columns, data}              │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 5. Build Response Payload                                │  │
│  │    {                                                     │  │
│  │      "text": "Here are 5 purchase orders...",           │  │
│  │      "columns": ["po_no", "status", "total", ...],      │  │
│  │      "rows": [[...], [...], ...],                       │  │
│  │      "visualization": {chart_type: "table", ...}        │  │
│  │    }                                                     │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 6. json.dumps(response_payload)  ✅ WORKS NOW!           │  │
│  │    └─ All values are JSON-serializable                   │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  answer = JSON_STRING                                           │
│  return {answer, sql, result, ...}                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                  (HTTP 200 with JSON)
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       FRONTEND (Browser)                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. app.js: Fetch and parse response                      │  │
│  │    const parsed = JSON.parse(response.answer)  ✅ Works  │  │
│  │    {                                                     │  │
│  │      text: "...",                                        │  │
│  │      columns: [...],                                     │  │
│  │      rows: [[...], [...], ...],                         │  │
│  │      visualization: {chart_type: "table", ...}          │  │
│  │    }                                                     │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 2. app.js: renderVisualization()  ✅ NEW                 │  │
│  │    ├─ Route by chart_type                                │  │
│  │    ├─ Call renderTable() for "table" type               │  │
│  │    └─ Or renderBarChart(), renderLineChart(), etc.      │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 3. app.js: renderTable()  ✅ NEW                         │  │
│  │    ├─ Create HTML <table>                                │  │
│  │    ├─ Add headers: <thead>                               │  │
│  │    ├─ Add rows: <tbody>                                  │  │
│  │    └─ Render with CSS styles                            │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │ 4. style.css: Apply styling  ✅ NEW                      │  │
│  │    ├─ Table: .data-table with hover effects             │  │
│  │    ├─ Headers: Purple background                         │  │
│  │    ├─ Rows: Alternating shades                          │  │
│  │    └─ Mobile responsive                                  │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                    │
│  RENDERED TABLE:                                                │
│  ┌─────────┬──────────┬───────┬────────────────────────┐       │
│  │ PO No   │ Status   │ Total │ Created At             │       │
│  ├─────────┼──────────┼───────┼────────────────────────┤       │
│  │ PO-001  │ DRAFT    │ 5000  │ 2026-01-22T17:47:21   │       │
│  │ PO-002  │ APPROVED │ 6000  │ 2026-01-21T17:47:21   │       │
│  │ PO-003  │ REJECTED │ 7000  │ 2026-01-20T17:47:21   │       │
│  │ PO-004  │ DRAFT    │ 8000  │ 2026-01-19T17:47:21   │       │
│  │ PO-005  │ APPROVED │ 9000  │ 2026-01-18T17:47:21   │       │
│  └─────────┴──────────┴───────┴────────────────────────┘       │
│                             │                                    │
└─────────────────────────────▼────────────────────────────────────┘
                              │
                        USER SEES TABLE
                    ✅ No error message
                    ✅ Professional display
                    ✅ All 5 rows visible
```

## Critical Points

```
DATABASE LAYER
  Decimal('5000.00') ← Native database type
  datetime(2026, 1, 22) ← Native database type

SERIALIZATION BARRIER ⚠️ 
  ❌ BEFORE: Try to JSON serialize raw objects
  ✅ AFTER: Normalize first (Decimal → float, datetime → ISO)

JSON LAYER
  5000.0 ✅ JSON serializable
  "2026-01-22T17:47:21" ✅ JSON serializable

FRONTEND LAYER  
  Chart.js ✅ Understands float values
  Table.render() ✅ Works with both floats and strings
  UI Display ✅ Professional table shown
```

## File Dependencies

```
┌─────────────────────────────────────────────────────┐
│         agent_orchestrator.py (Backend)             │
│  ┌────────────────────────────────────────────────┐ │
│  │ _normalize_rows() ← NEW (lines 105-133)         │ │
│  │ run() method calls normalization (line 438)     │ │
│  │ Builds response_payload with visualization     │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────┘
                       │
                    (JSON)
                       │
┌──────────────────────▼──────────────────────────────┐
│      static/index.html (Frontend HTML)              │
│  ┌────────────────────────────────────────────────┐ │
│  │ Chart.js CDN (line 10) ← NEW                   │ │
│  │ Loads app.js script                            │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────┘
                       │
                   (JavaScript)
                       │
┌──────────────────────▼──────────────────────────────┐
│      static/app.js (Frontend Logic)                 │
│  ┌────────────────────────────────────────────────┐ │
│  │ addMessage() ← Parses JSON response             │ │
│  │ renderVisualization() ← Routes to renderers    │ │
│  │ renderTable() ← Draws HTML table               │ │
│  │ renderBarChart() ← Uses Chart.js               │ │
│  │ renderLineChart() ← Uses Chart.js              │ │
│  │ renderKPI() ← Custom styling                   │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────┘
                       │
                    (CSS Classes)
                       │
┌──────────────────────▼──────────────────────────────┐
│      static/style.css (Styling)                     │
│  ┌────────────────────────────────────────────────┐ │
│  │ .data-table ← Table styles                     │ │
│  │ .chart-container ← Chart container             │ │
│  │ .kpi-container ← KPI styles                    │ │
│  │ @media (max-width: 600px) ← Mobile responsive  │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘

RESULT: Beautiful, responsive table in user's browser ✅
```

## Summary

The fix bridges the gap between database types (Decimal/datetime) and JSON serialization by normalizing data at the boundary. The frontend then uses Chart.js for professional visualization rendering.

**Before**: Data → JSON serialization FAILS ❌
**After**: Data → Normalize → JSON serialization SUCCEEDS ✅ → Render visualization → User sees table
