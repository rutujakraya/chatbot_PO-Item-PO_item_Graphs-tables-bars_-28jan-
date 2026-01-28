# Setup & Installation Guide

## Complete Tech Stack & Environment Setup

This guide walks through setting up the **Agentic RAG NL2SQL** project from scratch on any system. All components are listed with versions and installation instructions.

---

## 📋 Prerequisites Checklist

Before starting, ensure you have administrative access to install software on your system.

---

## 🛠️ Part 1: Core Development Tools

### 1.1 Visual Studio Code
**Required Version:** Latest (2024.12+)

**Download:**
- Go to https://code.visualstudio.com/
- Download and install for your OS (Windows/Mac/Linux)

**VS Code Extensions Needed:**
```
1. Python (ms-python.python)
2. Pylance (ms-python.vscode-pylance)
3. PostgreSQL (ckolkman.vscode-postgres)
4. Postman (postmanruntime.postman-for-vscode)
5. Thunder Client (rangav.vscode-thunder-client) [optional API testing]
```

**Install Extensions:**
```bash
# Open VS Code and install each extension via Extensions Marketplace (Cmd+Shift+X)
# Or via command line:
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension ckolkman.vscode-postgres
```

---

### 1.2 Python
**Required Version:** Python 3.9 or 3.10 (3.11+ recommended)

**Download:**
- Go to https://www.python.org/downloads/
- Download Python 3.10 or 3.11
- During installation, **CHECK: "Add Python to PATH"**

**Verify Installation:**
```bash
python3 --version
# Expected output: Python 3.10.x or 3.11.x

pip --version
# Expected output: pip 23.x or higher
```

---

## 🗄️ Part 2: Database Setup

### 2.1 PostgreSQL
**Required Version:** PostgreSQL 14 or higher (15+ recommended)

**Download & Install:**
- Go to https://www.postgresql.org/download/
- Download PostgreSQL 15
- During installation:
  - Set password for `postgres` user (remember this!)
  - Keep port as default: **5432**
  - Check "pgAdmin 4" during installation

**Verify Installation:**
```bash
psql --version
# Expected output: psql (PostgreSQL) 15.x

# Test connection
psql -U postgres -c "SELECT version();"
# Should return PostgreSQL version info
```

**Create Database:**
```bash
# Connect to PostgreSQL
psql -U postgres

# In psql prompt, run:
CREATE DATABASE procure_phase1_first3tables;
\q

# Verify database exists
psql -U postgres -c "\l"
```

---

### 2.2 pgAdmin
**Version:** Comes with PostgreSQL installation

**Access pgAdmin:**
```
URL: http://localhost:5050
Default Username: postgres@pgadmin.org
Default Password: admin (change this!)
```

**Setup Database Connection in pgAdmin:**
1. Right-click "Servers" → "Register" → "Server"
2. Name: `Procurement DB`
3. Connection tab:
   - Host: `localhost`
   - Port: `5432`
   - Username: `postgres`
   - Password: (your postgres password)
4. Click "Save"

---

## 🤖 Part 3: AI/ML Infrastructure

### 3.1 Ollama (LLM Runtime)
**Required Version:** Latest (0.1.x)
**Model:** Llama 2 7B or Llama 3 8B

**Download & Install:**
- Go to https://ollama.ai
- Download for your OS (Windows/Mac/Linux)
- Install and run

**Download LLM Model:**
```bash
# This downloads ~4GB, be patient
ollama pull llama2
# or for Llama 3 (smaller, better):
ollama pull llama2

# Verify model loaded
ollama list
# Should show: llama2  latest  ...
```

**Start Ollama Service:**
```bash
# Mac/Linux: automatically starts
# Windows: Run "ollama" application

# Verify it's running
curl http://localhost:11434/api/tags
# Should return model info in JSON
```

**Important:**
- Ollama runs on port **11434** (do not change)
- Keep it running in background while using the project
- First request may take 10-30 seconds (model loading)

---

### 3.2 Qdrant Vector Database
**Required Version:** Latest Docker image (0.11.x+)

**Installation Options:**

#### Option A: Docker (Recommended)
```bash
# Install Docker Desktop from https://www.docker.com/products/docker-desktop

# Run Qdrant
docker run -p 6333:6333 -p 6334:6334 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant:latest

# Qdrant will be accessible at http://localhost:6333
```

#### Option B: Local Installation (Mac)
```bash
# Install via Homebrew
brew tap qdrant/qdrant
brew install qdrant

# Start Qdrant
qdrant
```

#### Option C: Local Installation (Linux/Windows)
- Download from https://github.com/qdrant/qdrant/releases
- Extract and run executable
- Qdrant starts on port **6333**

**Verify Qdrant:**
```bash
curl http://localhost:6333/health
# Expected: {"status":"ok"}
```

---

## 📦 Part 4: Python Virtual Environment & Dependencies

### 4.1 Create Virtual Environment
```bash
# Navigate to project directory
cd /path/to/agentic-rag-nl2sql

# Create virtual environment
python3 -m venv .venv

# Activate it
# On Mac/Linux:
source .venv/bin/activate

# On Windows:
.venv\Scripts\activate

# You should see (.venv) in your terminal prompt
```

### 4.2 Install Dependencies
```bash
# Ensure you're in the activated virtual environment

# Upgrade pip
pip install --upgrade pip

# Install all project dependencies
pip install -r requirements.txt
```

**requirements.txt Contents:**
```
# Backend & API
Flask==2.3.2
requests==2.31.0
python-dotenv==1.0.0

# Database
psycopg2-binary==2.9.7
SQLAlchemy==2.0.20

# Vector Database
qdrant-client==1.7.0

# ML/AI
sentence-transformers==2.2.2
numpy==1.24.3
scikit-learn==1.3.0

# Data Processing
pandas==2.0.3
opencv-python==4.8.0.76

# Web & API
Werkzeug==2.3.7
itsdangerous==2.1.2

# Utilities
python-dateutil==2.8.2
chardet==5.1.0
urllib3==2.0.4
```

### 4.3 Verify Dependencies
```bash
# Check all installed packages
pip list

# Test key imports
python3 -c "import psycopg2; import requests; import qdrant_client; print('All dependencies OK!')"
```

---

## 🔐 Part 5: Environment Variables

### 5.1 Create .env File
```bash
# In project root directory, create .env file
touch .env
```

### 5.2 Configure .env
```bash
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=procure_phase1_first3tables
DB_USER=postgres
DB_PASSWORD=your_postgres_password

# Ollama Configuration
OLLAMA_MODEL=llama2
OLLAMA_HOST=http://localhost:11434

# Qdrant Configuration
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_API_KEY=

# Application Configuration
FLASK_ENV=development
FLASK_DEBUG=True
SECRET_KEY=your_secret_key_here
```

---

## 🎯 Part 6: Project Structure

```
agentic-rag-nl2sql/
├── .venv/                          # Virtual environment (NOT in GitHub)
├── static/
│   ├── index.html                 # Frontend UI
│   ├── app.js                     # Frontend JS
│   └── style.css                  # Frontend styles
├── agent_orchestrator.py           # Main orchestration agent
├── agent_planner.py                # Query planning agent
├── agent_retriever.py              # RAG retrieval agent
├── agent_semantic_analyzer.py      # Semantic intent analysis
├── agent_sql_generator.py          # SQL generation agent
├── agent_sql_validator.py          # SQL validation agent
├── agent_sql_executor.py           # SQL execution agent
├── agent_response_formatter.py     # Response formatting agent
├── agent_visualization_mapper.py   # Data to Nivo JSON mapper
├── agent_visualization.py          # Chart rendering agent
├── app.py                          # Flask application entry point
├── db_init.py                      # Database initialization script
├── db_metadata.txt                 # Database schema metadata
├── qdrant_setup.py                 # Qdrant vector DB setup
├── requirements.txt                # Python dependencies
├── .env                            # Environment variables (NOT in GitHub)
├── .gitignore                      # Git ignore rules
├── README.md                        # Project overview
├── ARCHITECTURE_DIAGRAM.md         # System architecture
├── SETUP_INSTALLATION_GUIDE.md     # This file
└── [OTHER_DOCUMENTATION_FILES]     # Guides and references
```

---

## 🚀 Part 7: Database Initialization

### 7.1 Load Sample Data
```bash
# Activate virtual environment first
source .venv/bin/activate  # Mac/Linux
# or
.venv\Scripts\activate     # Windows

# Run database initialization
python3 db_init.py

# This will:
# - Create tables (po, po_items, items)
# - Load sample data
# - Create soft-delete triggers
# - Setup indexes
```

### 7.2 Verify Database
```bash
# Connect to database
psql -U postgres -d procure_phase1_first3tables

# In psql, check tables
\dt

# Should show: po, po_items, items tables

# Check row counts
SELECT COUNT(*) FROM po;
SELECT COUNT(*) FROM po_items;
SELECT COUNT(*) FROM items;

# Exit
\q
```

---

## 🔄 Part 8: Vector Database Setup

### 8.1 Initialize Qdrant Collections
```bash
# Ensure Qdrant is running
# Run the Qdrant setup script

python3 qdrant_setup.py

# This will:
# - Create embeddings collection
# - Create metadata collection
# - Initialize with sample vectors
# - Test connection
```

### 8.2 Verify Qdrant
```bash
# Check Qdrant health
curl http://localhost:6333/health

# List collections
curl http://localhost:6333/collections

# Should show your collections in JSON
```

---

## ▶️ Part 9: Running the Application

### 9.1 Start All Services

**Terminal 1: Start Ollama (if not already running)**
```bash
# Mac
ollama serve

# Linux/Windows
# Run Ollama application
```

**Terminal 2: Start Qdrant (if using Docker)**
```bash
docker run -p 6333:6333 -p 6334:6334 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant:latest
```

**Terminal 3: Start Flask Application**
```bash
# Activate virtual environment
source .venv/bin/activate  # Mac/Linux
# or
.venv\Scripts\activate     # Windows

# Set Flask app
export FLASK_APP=app.py    # Mac/Linux
# or
set FLASK_APP=app.py       # Windows

# Run Flask
flask run

# Output should show:
# * Running on http://localhost:5000
```

### 9.2 Access Application
```
Open browser and go to: http://localhost:5000
```

---

## 🧪 Part 10: Testing Setup

### 10.1 Run Tests
```bash
# From project root (with .venv activated)

# Test database connection
python3 -c "from agent_sql_executor import SQLExecutorAgent; print('DB OK')"

# Test Qdrant connection
python3 -c "from agent_retriever import RetrievalAgent; print('Qdrant OK')"

# Test semantic analyzer
python3 test_semantic_quick.py

# Run full test suite
python3 test_semantic_correctness.py
```

### 10.2 Sample Queries
Once the app is running, try these in the UI:

1. "List the 5 most recently created purchase orders"
2. "Which item has the highest total ordered quantity?"
3. "How many purchase orders are in draft status?"
4. "Show me all items with their base prices"

---

## 📝 Part 11: Troubleshooting

### Issue: PostgreSQL Connection Error
```
Error: could not translate host name "localhost" to address

Solution:
1. Verify PostgreSQL is running
2. Check DB_HOST is "localhost" or "127.0.0.1"
3. Verify password in .env matches
4. Run: psql -U postgres -c "SELECT 1"
```

### Issue: Ollama Not Responding
```
Error: Connection refused (http://localhost:11434)

Solution:
1. Verify Ollama is running: ollama serve
2. Test: curl http://localhost:11434/api/tags
3. Check port 11434 is not blocked
4. Restart Ollama service
```

### Issue: Qdrant Connection Failed
```
Error: Failed to connect to Qdrant at localhost:6333

Solution:
1. Verify Docker is running (if using Docker)
2. Check: curl http://localhost:6333/health
3. Ensure port 6333 is available
4. Restart Qdrant container: docker restart [container_id]
```

### Issue: Vector Embedding Fails
```
Error: Failed to load sentence-transformers model

Solution:
1. First embedding download takes time (~500MB)
2. Ensure internet connection is stable
3. Model cached in ~/.cache/huggingface/
4. Clear cache if corrupted: rm -rf ~/.cache/huggingface/
```

### Issue: Virtual Environment Not Activating
```
Solution:
1. Delete .venv folder
2. Recreate: python3 -m venv .venv
3. Activate again: source .venv/bin/activate
4. Reinstall: pip install -r requirements.txt
```

---

## 🔄 Part 12: Updating & Maintenance

### 12.1 Update Dependencies
```bash
# Activate virtual environment
source .venv/bin/activate

# Update all packages
pip install --upgrade -r requirements.txt

# Generate new requirements file
pip freeze > requirements.txt
```

### 12.2 Reset Qdrant Data
```bash
# Stop Qdrant container
docker stop [container_id]

# Remove volume (deletes data)
docker volume rm qdrant_storage

# Restart with fresh data
docker run -p 6333:6333 qdrant/qdrant:latest
```

### 12.3 Reset Database
```bash
# WARNING: This deletes all data!

psql -U postgres

DROP DATABASE procure_phase1_first3tables;
CREATE DATABASE procure_phase1_first3tables;
\q

# Reinitialize
python3 db_init.py
```

---

## ✅ Part 13: Verification Checklist

Before declaring setup complete, verify all of these:

- [ ] Python 3.10+ installed and in PATH
- [ ] PostgreSQL running on port 5432
- [ ] Database `procure_phase1_first3tables` created
- [ ] pgAdmin accessible at http://localhost:5050
- [ ] Ollama running with llama2 model loaded
- [ ] Qdrant running on port 6333
- [ ] Virtual environment created and activated
- [ ] All requirements installed (pip list shows ~15+ packages)
- [ ] .env file configured with correct credentials
- [ ] Database initialized with sample data
- [ ] Qdrant collections created
- [ ] Flask app runs without errors
- [ ] Frontend accessible at http://localhost:5000
- [ ] Can submit a test query and get results

---

## 🎓 Quick Reference Commands

```bash
# Activate environment
source .venv/bin/activate

# Deactivate environment
deactivate

# Check Python version
python3 --version

# List installed packages
pip list

# Install specific package
pip install package_name==version

# Connect to PostgreSQL
psql -U postgres -d procure_phase1_first3tables

# Test Ollama
curl http://localhost:11434/api/tags

# Test Qdrant
curl http://localhost:6333/health

# Run Flask development server
flask run --debug

# Run specific test
python3 test_semantic_quick.py
```

---

## 📞 Support & Resources

- **Python Docs:** https://docs.python.org/3.10/
- **PostgreSQL Docs:** https://www.postgresql.org/docs/
- **Ollama Docs:** https://github.com/ollama/ollama
- **Qdrant Docs:** https://qdrant.tech/documentation/
- **Flask Docs:** https://flask.palletsprojects.com/
- **Sentence Transformers:** https://www.sbert.net/

---

**Setup Complete! You're ready to start developing.**

For architecture details and system workflow, see [ARCHITECTURE_AND_WORKFLOW.md](ARCHITECTURE_AND_WORKFLOW.md)
