Setup

### Step 1 — Create virtual environment

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

macOS/Linux:

```bash
source venv/bin/activate
```

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3 — Configure environment

Create a `.env` file in the project root directory with the following variables:

```env
# LLM API Keys
GROQ_API_KEY=your_groq_api_key_here
OPENAI_API_KEY=your_openai_api_key_here

# External Search
TAVILY_API_KEY=your_tavily_api_key_here

# Vector Database (Pinecone)
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX_NAME=fde-it-support-rag
PINECONE_NAMESPACE=company-it-kb

# LLM Models
GROQ_MODEL=openai/gpt-oss-20b
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small

# Security
ADMIN_API_KEY=change-me-in-production

# Application Settings
APP_ENV=development
```

#### Environment Variable Reference

| Variable | Description | Example | Required |
|---|---|---|---|
| `GROQ_API_KEY` | API key for Groq LLM (routing, grading, generation) | `gsk_...` | ✅ Yes |
| `OPENAI_API_KEY` | API key for OpenAI (embeddings and fallback LLM) | `sk-...` | ✅ Yes |
| `TAVILY_API_KEY` | API key for Tavily web search | `tvly-...` | ✅ Yes |
| `PINECONE_API_KEY` | API key for Pinecone vector database | `pckey-...` | ✅ Yes |
| `PINECONE_INDEX_NAME` | Pinecone index name | `fde-it-support-rag` | ⚠️ Optional (default: `fde-it-support-rag`) |
| `PINECONE_NAMESPACE` | Pinecone namespace for document isolation | `company-it-kb` | ⚠️ Optional (default: `company-it-kb`) |
| `GROQ_MODEL` | Groq model identifier | `openai/gpt-oss-20b` | ⚠️ Optional (default: `openai/gpt-oss-20b`) |
| `OPENAI_MODEL` | OpenAI model identifier | `gpt-4o-mini` | ⚠️ Optional (default: `gpt-4o-mini`) |
| `EMBEDDING_MODEL` | Embedding model for vectorization | `text-embedding-3-small` | ⚠️ Optional (default: `text-embedding-3-small`) |
| `ADMIN_API_KEY` | Secret key for admin endpoints (document ingestion) | `your-secure-key` | ⚠️ Optional (default: `change-me`) |
| `APP_ENV` | Application environment | `development` or `production` | ⚠️ Optional (default: `development`) |

**⚠️ Important Security Notes:**

- Never commit `.env` to version control. Add it to `.gitignore`.
- Change `ADMIN_API_KEY` to a secure random value in production.
- Use environment variables instead of hardcoding secrets.
- Restrict API key access to only the services that need them.

### Step 4 — Load sample company knowledge

```bash
python ingest_sample_kb.py
```

This demonstrates the RAG ingestion pipeline:

```text
Company Documents
   ↓
Load Documents
   ↓
Chunk Text
   ↓
HuggingFace Embeddings
   ↓
Pinecone Vector Database
```

### Step 5 — Run the application

```bash
python run.py
```

Open:

```text
http://127.0.0.1:8000
```

FastAPI API docs:

```text
http://127.0.0.1:8000/docs
```

---

## 8. Classroom Demo Scenarios

### Demo A — Private KB Success

Ask:

> **How do I connect to the company VPN from home?**

Expected path:

```text
Router → KB
Private KB Retrieval
KB Grade → GOOD
Generate from Private KB
```

Teaching point:

> Trusted internal company knowledge is preferred. No public web call is necessary.

---

### Demo B — Company Policy Question

Ask:

> **Can IT support ask me to share my MFA code?**

The internal handbook says employees must never share passwords or MFA codes with support personnel.

Expected path:

```text
Router → KB
Private KB Retrieval
KB Grade → GOOD
Private KB Answer
```

Teaching point:

> RAG allows the model to answer using company-specific information it was never trained on.

---

### Demo C — External / Current Information

Ask:

> **What is the latest Microsoft Teams outage guidance?**

Expected path when internal documents do not answer it:

```text
Router → KB
Private KB Retrieval
KB Grade → WEAK
Tavily Search
Web Grade → GOOD
Web Answer
```

Teaching point:

> Agentic RAG can choose another information source rather than blindly answering from irrelevant vectors.

---

### Demo D — Query Rewrite

Ask a deliberately vague question such as:

> **My work communication app is acting strange after the new update. What should I do?**

If both KB and first external retrieval are weak, the workflow can rewrite the query and retry.

Teaching point:

> Retrieval failure does not immediately mean failure. An agent can improve the search query and try again while using a retry guard to avoid loops.

---

### Demo E — Direct Conversation

Ask:

> **Hello!**

Expected path:

```text
Router → DIRECT
Direct Answer
```

Teaching point:

> Not every message should trigger expensive vector search or web search.

---

## 9. Document Upload Demo

The sidebar contains **Add Company Document**.

Use the `ADMIN_API_KEY` from `.env`.

Upload one of:

- `.pdf`
- `.txt`
- `.md`
- `.docx`

The backend performs:

```text
Upload
  ↓
Validate Type
  ↓
Load Text
  ↓
Recursive Chunking
  ↓
Embeddings
  ↓
Pinecone Indexing
  ↓
Immediately Available for Retrieval
```

This is useful in an FDE demonstration because the customer does not want to edit Python every time a new policy is published.

---

## 10. API Endpoints

### `GET /api/health`

Health check.

### `POST /api/chat`

Request:

```json
{
  "question": "How do I reset my company password?"
}
```

Response contains:

```json
{
  "answer": "...",
  "source_used": "private_kb",
  "trace": [
    "Router → KB",
    "Private KB retrieval → 4 chunks",
    "KB evidence grade → GOOD",
    "Answer generation → PRIVATE KB"
  ],
  "citations": []
}
```

### `POST /api/ingest`

Protected by the `X-Admin-Key` request header.

Uploads a document and adds its chunks to Pinecone.

---