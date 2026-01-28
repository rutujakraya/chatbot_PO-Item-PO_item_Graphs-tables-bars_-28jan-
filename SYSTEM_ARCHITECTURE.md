# AGENTIC RAG NL2SQL SYSTEM ARCHITECTURE
**Industry-Grade Implementation** | January 2026

---

## 🎯 SYSTEM OVERVIEW

This is a **SMART DATA ASSISTANT** that intelligently routes questions through two distinct pipelines:

1. **GENERAL QUESTIONS** → Direct LLM response (LLaMA-3 8B)
2. **DATABASE QUESTIONS** → RAG + SQL pipeline (LLaMA-3 8B + Qdrant + PostgreSQL)

The system uses **metadata-driven SQL generation**, meaning users do NOT need to know table or column names.

---

## 🏗️ ARCHITECTURE DIAGRAM

```
User Question
    ↓
┌─────────────────────────────────────────┐
│     PlannerAgent                        │
│  (classify_intent)                      │
│                                         │
│  Is this a database question?           │
│  ├─ NO  → "direct" (general Q&A)       │
│  └─ YES → "rag_sql" (database query)   │
└─────────────────────────────────────────┘
    ↓
    ├─── "direct" ────────────────────────────────────┐
    │                                                  │
    │    ResponseFormatterAgent                       │
    │    .generate_general_response()                 │
    │    ↓                                            │
    │    LLaMA-3 8B (2-line answer)                  │
    │    ↓                                            │
    │    [No database access]                        │
    │                                                │
    └────────────→ User-facing Response             │
                                                     │
    ├─── "rag_sql" ────────────────────────────────┐ │
    │                                              │ │
    │   RetrievalAgent (RAG)                       │ │
    │   └─→ Query Qdrant for metadata chunks       │ │
    │       (from db_metadata.txt)                 │ │
    │           ↓                                  │ │
    │   Chunks: table info, columns, examples      │ │
    │           ↓                                  │ │
    │   SQLGeneratorAgent                          │ │
    │   └─→ Inject metadata into LLM prompt        │ │
    │       LLaMA-3 8B generates SQL               │ │
    │           ↓                                  │ │
    │   SQLValidatorAgent                          │ │
    │   └─→ Validate SQL syntax & rules            │ │
    │           ↓                                  │ │
    │   SQLExecutorAgent                           │ │
    │   └─→ Execute on PostgreSQL                  │ │
    │           ↓                                  │ │
    │   ResponseFormatterAgent                     │ │
    │   └─→ Format results (human-readable)        │ │
    │                                              │ │
    └───────────→ User-facing Response ←───────────┘ │
                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 🔄 DETAILED AGENT WORKFLOWS

### 1️⃣ INTENT CLASSIFICATION (PlannerAgent)

**File:** `agent_planner.py`

```python
classify_intent(user_query: str) → "direct" | "rag_sql"
```

**Decision Logic:**

| Category | Keywords | Intent |
|----------|----------|--------|
| **General Knowledge** | "what is", "define", "explain", "biography" | `direct` |
| **Opinion/Creative** | "joke", "poem", "review", "opinion" | `direct` |
| **Real-World Data** | "weather", "news", "movie", "temperature" | `direct` |
| **Predictions** | "will win", "predict", "probability" | `direct` |
| **Business Data** | "count", "sum", "total", "users", "orders" | `rag_sql` |
| **Data Operations** | "list", "show", "find", "filter", "group by" | `rag_sql` |
| **Default** | No clear indicators | `direct` (safe default) |

**Example:**
```
Input:  "What is machine learning?"
Output: "direct" → Use LLaMA-3 directly

Input:  "How many users signed up last month?"
Output: "rag_sql" → Go through RAG + SQL pipeline
```

---

### 2️⃣ DIRECT LLM PATH (ResponseFormatterAgent)

**File:** `agent_response_formatter.py`

```python
generate_general_response(user_question: str) → str
```

**Flow:**
1. Create a concise prompt (2-line requirement)
2. Call LLaMA-3 8B via Ollama
3. Trim output to 2 lines max
4. Return polite, business-friendly response

**Prompt Template:**
```
You are a helpful assistant. Answer the user's question in 2 sentences only. 
Be polite and concise. Do NOT mention databases, SQL, or data lookups.

Question: {user_question}

Answer (2 sentences only):
```

**Example Response:**
```
Question: "What is machine learning?"
Answer: "Machine learning is a way for computers to learn patterns from data 
without being explicitly programmed. It enables systems to make predictions or 
decisions automatically based on examples."
```

---

### 3️⃣ RAG METADATA RETRIEVAL (RetrievalAgent)

**File:** `agent_retriever.py`

```python
retrieve(query: str, limit: int = 5) → List[Dict]
```

**Data Source:** `db_metadata.txt` (embedded and indexed in Qdrant)

**Flow:**
1. Encode user question using BGE embeddings
2. Query Qdrant for top-5 most similar chunks
3. Return chunks with structure:
   ```python
   {
       "text": "Table definition...",
       "table": "users",
       "type": "table_info",
       "score": 0.85
   }
   ```

**Chunks Include:**
- Table definitions (users, orders, products)
- Column meanings and examples
- Relationships between tables
- Example business questions
- Natural-language → SQL mappings

**Example:**
```
Query:  "How many users signed up last month?"
Retrieved Chunks:
├─ [users - table_info] "Table: users with id, name, email, created_at"
├─ [users - relationship] "Orders linked via user_id"
├─ [example] "How many users signed up last month? → COUNT + WHERE created_at"
└─ [mapping] "created_at for recency filters"
```

---

### 4️⃣ SQL GENERATION (SQLGeneratorAgent)

**File:** `agent_sql_generator.py`

```python
generate_sql(user_question: str, schema_context: list) -> str
```

**Key Features:**
1. **Metadata Injection**: Formats retrieved chunks into context
2. **Explicit Prompting**: Includes NL→SQL examples
3. **Validation**: Rejects system tables, non-SELECT queries
4. **Fallback**: Rule-based generation if LLM fails

**Prompt Structure:**
```
You are an EXPERT PostgreSQL SQL generator.

RULES:
- SELECT only
- Use ONLY provided tables/columns
- No system tables
- Include JOINs, GROUP BY, ORDER BY when appropriate

NATURAL LANGUAGE → SQL EXAMPLES:
- "How many users?" → SELECT COUNT(*) FROM users;
- "Which user has most orders?" → SELECT u.id, u.name, COUNT(o.id) 
  FROM users u JOIN orders o ON u.id=o.user_id 
  GROUP BY u.id, u.name ORDER BY COUNT(o.id) DESC LIMIT 1;

SCHEMA (from db_metadata.txt):
{formatted_chunks}

USER QUESTION: {user_question}

SQL OUTPUT (only query, no explanation):
```

**Temperature:** 0.1 (low for deterministic SQL)

**Validation Rules:**
- ✓ Starts with SELECT
- ✓ Ends with semicolon
- ✓ No system table references
- ✓ No INSERT/UPDATE/DELETE/DROP

**Fallback Patterns:**
If LLM fails, use rule-based generation:
```
Pattern: "show N <table>"    → SELECT * FROM table LIMIT N;
Pattern: "how many <table>"  → SELECT COUNT(*) FROM table;
Pattern: "most orders"       → SELECT u.*, COUNT(o.id) FROM users u 
                               JOIN orders o GROUP BY u.id ORDER BY COUNT(o.id) DESC;
```

---

### 5️⃣ SQL VALIDATION (SQLValidatorAgent)

**File:** `agent_sql_validator.py`

**Checks:**
```python
validate(sql: str) → bool
├─ Non-empty string
├─ Starts with SELECT or WITH
├─ No forbidden keywords (DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, EXEC, COPY)
└─ Proper word boundaries for forbidden keywords
```

---

### 6️⃣ SQL EXECUTION (SQLExecutorAgent)

**File:** `agent_sql_executor.py`

```python
execute(sql: str) → Dict[columns: List[str], rows: List[Tuple]]
```

**Database:** PostgreSQL on localhost:5432

**Returns:**
```python
{
    "columns": ["id", "name", "email", "count"],
    "rows": [(1, "Alice", "alice@example.com", 5), ...]
}
```

---

### 7️⃣ RESULT FORMATTING (ResponseFormatterAgent)

**File:** `agent_response_formatter.py`

```python
format(user_query: str, sql: str, result: dict) -> str
```

**Behaviors:**

| Scenario | Response |
|----------|----------|
| **No rows** | "I couldn't find any matching data." |
| **>20 rows** | "There are {N} records. Would you like: 1) Pagination 2) Filters 3) Summary?" |
| **Names + Emails** | Bullet list: "- Alice — alice@example.com" |
| **Two columns** | Key-value pairs: "- id: 1, name: Alice" |
| **Generic** | Formatted table-like output |

**Human-Readable Focus:**
- No SQL shown by default
- Business terminology
- Polite, conversational tone
- Clear summary of results

---

## 🎯 ORCHESTRATOR FLOW (AgentOrchestrator)

**File:** `agent_orchestrator.py`

```python
run(user_query: str) → Dict
```

**Main Loop (with Retry Logic):**

```
1. Classify Intent
   ├─ IF "direct" → Call generate_general_response() → Return answer
   └─ IF "rag_sql" → Continue to step 2

2. Retrieve Metadata (RAG)
   ├─ Query Qdrant for 5 relevant chunks
   └─ IF no chunks → Return polite error

3. Retry Loop (up to 2 attempts)
   ├─ 3a. Generate SQL (with metadata injection)
   ├─ 3b. Validate SQL
   ├─ 3c. Execute SQL
   ├─ 3d. Check result size
   │   ├─ 0 rows → "No matching data"
   │   ├─ >20 rows → Ask for clarification
   │   └─ 1-20 rows → Format and return
   └─ IF any step fails → Retry from 3a

4. Fallback (if all retries exhausted)
   └─ Return friendly error message
```

**Return Structure:**
```python
{
    "answer": "User-facing response string",
    "sql": "SELECT ... FROM ...; or None",
    "result": {"columns": [...], "rows": [...]} or None,
    "context": [{"text": "...", "table": "...", ...}, ...],
    "agent_flow": [
        "Intent classification: rag_sql",
        "Route: RAG + SQL",
        "Retrieved 5 metadata chunks",
        "--- Attempt 1 ---",
        "  ✓ Generated SQL: SELECT ...",
        "  ✓ SQL validation passed",
        "  ✓ SQL execution successful",
        "  → 3 rows returned and formatted"
    ]
}
```

---

## 🔑 KEY DESIGN DECISIONS

### 1. Metadata-Driven (Not Schema-Driven)
- **Why:** Users shouldn't need to know table/column names
- **How:** db_metadata.txt includes natural language descriptions, example questions, and mappings
- **Benefit:** More intuitive queries; better user experience

### 2. Default to "direct" for Ambiguous Questions
- **Why:** Safer to give a general answer than force database lookup
- **How:** Explicit database keywords trigger "rag_sql"; others → "direct"
- **Benefit:** Graceful degradation; no unnecessary DB calls

### 3. Retry Logic with Fallback
- **Why:** LLM may fail; humans need deterministic answers
- **How:** Attempt LLM generation 2x, then use rule-based fallback
- **Benefit:** High reliability; answers most questions successfully

### 4. Large Result Handling
- **Why:** Dumping 1000 rows doesn't help users
- **How:** Ask for clarification instead of showing data
- **Benefit:** Better UX; prevents information overload

### 5. Explicit Prompting
- **Why:** LLMs respond better to clear instructions
- **How:** Include examples, rules, and metadata in every prompt
- **Benefit:** Higher quality SQL; fewer validation errors

---

## 📊 EXAMPLE WORKFLOWS

### Example 1: General Question

```
User: "What is machine learning?"
↓
PlannerAgent: "direct" (keyword: "what is")
↓
ResponseFormatterAgent.generate_general_response()
↓
LLaMA-3 8B generates 2-line answer
↓
No database access, no SQL
↓
Response: "Machine learning is a way for computers to learn patterns from data. 
It enables systems to make predictions or decisions automatically."
```

---

### Example 2: Database Question

```
User: "How many users signed up last month?"
↓
PlannerAgent: "rag_sql" (keywords: "how many", "users", "signup")
↓
RetrievalAgent retrieves 5 chunks from db_metadata.txt
├─ [users - table_info]
├─ [users - created_at column]
├─ [example] "How many users signed up last month?"
├─ [mapping] "created_at for recency filters"
└─ [SQL construct] "COUNT(*) + WHERE created_at"
↓
SQLGeneratorAgent (with metadata context)
↓
LLaMA-3 8B generates:
"SELECT COUNT(*) AS new_users 
FROM users1 
WHERE created_at >= NOW() - INTERVAL '1 month';"
↓
SQLValidatorAgent: ✓ Valid
↓
SQLExecutorAgent: ✓ Executed on PostgreSQL
Result: {"columns": ["new_users"], "rows": [(245,)]}
↓
ResponseFormatterAgent.format()
↓
Response: "Found 245 users who signed up last month."
```

---

### Example 3: Complex Query

```
User: "Which products are most frequently ordered?"
↓
PlannerAgent: "rag_sql" (keywords: "products", "ordered")
↓
RetrievalAgent retrieves 5 chunks:
├─ [products - table_info]
├─ [orders - table_info]
├─ [relationship] "orders.product_id → products.id"
├─ [example] "Top products by order frequency"
└─ [mapping] "COUNT + GROUP BY + JOIN"
↓
SQLGeneratorAgent (metadata-driven)
↓
LLaMA-3 8B generates:
"SELECT p.id, p.name, COUNT(o.id) AS order_count
FROM products p
JOIN orders o ON o.product_id = p.id
GROUP BY p.id, p.name
ORDER BY order_count DESC
LIMIT 10;"
↓
Validation: ✓ Valid
Execution: ✓ Returns 10 rows
↓
ResponseFormatterAgent.format()
↓
Response:
"Found 10 products. Top results:
- Laptop Pro: 243 orders
- USB Cable: 189 orders
- Wireless Mouse: 156 orders
..."
```

---

## 🚀 DEPLOYMENT CHECKLIST

### Prerequisites
- [ ] Ollama running with llama3:8b model
- [ ] PostgreSQL running with agentic_rag_db database
- [ ] Qdrant running on localhost:6333
- [ ] db_metadata.txt chunked and indexed in Qdrant
- [ ] Python dependencies installed (requirements.txt)

### Environment Setup
```bash
# Install dependencies
pip install -r requirements.txt

# Start services
# Terminal 1: Ollama
ollama serve

# Terminal 2: PostgreSQL
# (already running, or start with: brew services start postgresql)

# Terminal 3: Qdrant
docker run -p 6333:6333 qdrant/qdrant

# Terminal 4: FastAPI Server
python -m uvicorn app:app --reload

# Terminal 5: (Optional) Seed Qdrant with metadata
python qdrant_setup.py
```

### Test Endpoints

**General Question:**
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is machine learning?"}'
```

**Database Question:**
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "How many users signed up last month?"}'
```

---

## 📝 SUMMARY: INDUSTRY-GRADE FEATURES

✅ **Intelligent Intent Classification** - Routes general vs. database questions  
✅ **Metadata-Driven SQL** - Users don't need to know table/column names  
✅ **RAG with db_metadata.txt** - Context from natural language documentation  
✅ **Dual LLM Paths** - Direct LLM for general Q&A; RAG+SQL for database  
✅ **Retry Logic with Fallback** - Handles LLM failures gracefully  
✅ **Result Pagination** - Handles large datasets without overwhelming users  
✅ **Human-Readable Output** - No SQL shown; business-friendly formatting  
✅ **Comprehensive Error Handling** - Polite failures; never crash with "Sorry"  
✅ **Validation Pipeline** - SQL validated before execution  
✅ **Observability** - Detailed agent_flow for debugging  

---

**Status:** ✅ Production-Ready  
**Last Updated:** January 2026
