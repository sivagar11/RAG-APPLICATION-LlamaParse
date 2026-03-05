# RAG-MAG

Multimodal RAG system that processes PDF manuals and answers questions using both text and images.

## Quick Start

```bash
# 1. Setup
cp env.example .env  # Configure with your API keys
pip install -r requirements.txt

# 2. Parse documents (stores in Qdrant Cloud)
make parse  # or: python src/parse.py

# 3. Choose your interface:

# Option A: Streamlit UI (interactive web app)
make run    # or: streamlit run src/app.py

# Option B: FastAPI REST API (for programmatic access)
make api    # or: cd src && uvicorn api.main:app --reload

# Option C: Docker Compose (containerized API)
docker compose up -d api
# API docs at http://localhost:8000/docs
```

**Architecture:**
- **Vector Database**: Qdrant Cloud (managed, scalable)
- **Image Storage**: Compressed base64 in Qdrant (800px, 60% JPEG quality)
- **LLM**: OpenAI or Gemini (configurable)
- **Embeddings**: OpenAI text-embedding-3-small
- **Parser**: LlamaParse for multimodal PDF extraction
- **Batch Upload**: Smart batching for reliable uploads

## Table of Contents

- [Configuration](#configuration)
- [Docker Deployment](#docker-deployment)
- [Document Ingestion Workflow](#document-ingestion-workflow)
- [Ingestion Architecture](#ingestion-architecture)
- [FastAPI Backend](#fastapi-backend)
- [Project Structure](#project-structure)
- [Development](#development)
- [Testing](#testing)
- [Tech Stack](#tech-stack)
- [Next Steps](#next-steps)

---

## Configuration

All configuration is centralized in `.env` file:

```bash
cp env.example .env  # Then edit with your values
```

### Required Settings

```bash
# API Keys
OPENAI_API_KEY=your_key_here
GEMINI_API_KEY=your_key_here
LLAMAPARSE_API_KEY=your_key_here

# LLM Provider
LLM_PROVIDER=gemini  # or openai

# Vector Database (Qdrant Cloud - pre-configured)
VECTOR_DB_TYPE=qdrant
QDRANT_URL=https://your-cluster.gcp.cloud.qdrant.io:6333
QDRANT_API_KEY=your_qdrant_key

# Image Storage
IMAGE_STORAGE_FORMAT=base64  # Images stored in Qdrant
```

### Optional Settings

See `env.example` for all available configuration options including:
- LlamaParse settings (model, region, OCR options)
- LLM model selection (OpenAI/Gemini)
- RAG parameters (chunk size, similarity threshold)
- Directory paths

---

## Docker Deployment

Run the API in a container using the included `Dockerfile` and `docker-compose.yml`.

### Prerequisites

- Docker + Docker Compose installed
- `.env` configured with required API keys

### Start Services

```bash
# API only (uses your .env settings, e.g., Qdrant Cloud)
docker compose up -d api

# API + local Qdrant for development
docker compose up -d api qdrant

# Full optional local stack (api + qdrant + minio)
docker compose up -d
```

### Build / Rebuild API Image

```bash
docker compose build api
```

### Stop Services

```bash
docker compose down
```

---

## Document Ingestion Workflow

### Initial Setup

```bash
# Put your PDFs in data/
cp /path/to/manuals/*.pdf data/

# Parse and index to Qdrant Cloud
make parse
```

**What this does:**
- ✅ Parses ALL PDFs in `data/` with LlamaParse
- ✅ Extracts text and page images
- ✅ Compresses & resizes images (800px max, 60% JPEG quality)
- ✅ Converts images to base64 with data URI format
- ✅ Uploads in batches of 2 pages (avoids timeouts)
- ✅ Stores vectors and images in Qdrant Cloud
- ✅ Assigns UUID to each document for tracking
- ✅ Excludes images from embeddings (prevents token overflow)

**Output:**
```
📄 Found 25 PDF file(s)
📝 Parsing documents...
   [1/25] Parsing manual1.pdf...
       ✅ Extracted 42 pages
   ...
📊 Summary:
   Documents indexed: 25
   Total pages: 1,247
   Stored in Qdrant Cloud
```

### Add More Documents

**Via API:**
```bash
# Upload via REST API
curl -X POST "http://localhost:8000/documents" \
  -F "file=@new-manual.pdf"
```

**Via Batch Script:**
```bash
# Add new PDFs to data/
cp /path/to/more/*.pdf data/

# Add to existing index (preserves existing documents)
make add-batch
```

---

## Ingestion Architecture

### Overview

The system includes a dedicated document management module (`src/ingestion/`) that provides CRUD operations with Qdrant Cloud storage.

### Key Features

✅ **Qdrant Cloud Storage**: Vectors and images stored in managed cloud database  
✅ **Base64 Images**: Images encoded and stored directly in Qdrant  
✅ **Document Tracking**: Proper SOURCE relationships enable document-level operations  
✅ **CRUD Operations**: Add, delete, update, and list documents programmatically  
✅ **UUID System**: Each document gets a unique identifier for tracking  
✅ **Multi-Worker Ready**: Qdrant supports concurrent access from multiple API workers  

### Core Components

#### 1. **parser.py** - LlamaParse Wrapper
- Wraps LlamaParse with SOURCE relationship configuration
- Extracts text and page screenshots from PDFs
- Encodes images to base64 for Qdrant storage
- Attaches metadata: document_id, filename, page_number, image_b64

#### 2. **index_manager.py** - Index Lifecycle
- Manages connection to Qdrant Cloud
- Singleton pattern for efficient index access
- Configures LLM and embedding models
- Supports both local and cloud vector stores

#### 3. **document_manager.py** - CRUD Operations
- `add_document(pdf_path, doc_id=None)` - Add PDF to Qdrant
- `delete_document(doc_id)` - Remove document from Qdrant
- `update_document(doc_id, new_pdf_path)` - Replace document
- `list_documents()` - Get all indexed documents
- `get_document_info(doc_id)` - Detailed document information

#### 4. **vector_store.py** - Vector Database Management
- Handles Qdrant Cloud connection
- Manages vector store configuration
- Supports multiple backend options (Qdrant, local, etc.)

### Usage Examples

**CLI (Batch Processing):**
```bash
make parse        # Parse and upload to Qdrant Cloud
make add-batch    # Add more documents (preserves existing)
```

**Python (Programmatic):**
```python
from ingestion import add_document, delete_document, list_documents

# Add a document (uploads to Qdrant Cloud)
result = await add_document("manual.pdf")
doc_id = result['document_id']

# List all documents
docs = list_documents()
for doc_id, info in docs.items():
    print(f"{doc_id}: {info['metadata']['filename']}")

# Delete document (removes from Qdrant)
delete_document(doc_id)
```

**Testing:**
```bash
python scripts/test_ingestion.py  # Test CRUD operations
```

### Technical Implementation

**SOURCE Relationships:**
```python
from llama_index.core.schema import NodeRelationship, RelatedNodeInfo

# Enable document tracking and deletion
node.relationships[NodeRelationship.SOURCE] = RelatedNodeInfo(node_id=doc_id)
```

This enables proper document tracking in Qdrant and allows deletion by document ID.

**Base64 Image Storage:**
```python
# Images encoded and stored in node metadata
node.metadata["image_b64"] = base64.b64encode(image_data).decode('utf-8')
```

Images are stored directly in Qdrant alongside text embeddings for atomic operations.

### Scalability

With Qdrant Cloud:
- ✅ **Multi-Process**: Multiple API workers can access the same index
- ✅ **Automatic Persistence**: Changes are immediately synced to cloud
- ✅ **Concurrent Access**: Thread-safe operations
- ✅ **Horizontal Scaling**: Add more API workers as needed

### API Integration

The ingestion module works seamlessly with FastAPI:

```python
from fastapi import FastAPI, UploadFile, BackgroundTasks
from ingestion import add_document, delete_document

app = FastAPI()

@app.post("/documents")
async def upload(file: UploadFile, bg: BackgroundTasks):
    temp_path = f"/tmp/{file.filename}"
    with open(temp_path, "wb") as f:
        f.write(await file.read())
    bg.add_task(add_document, temp_path)
    return {"status": "processing"}

@app.delete("/documents/{doc_id}")
def delete(doc_id: str):
    return delete_document(doc_id)
```

---

## FastAPI Backend

### Overview

The FastAPI backend provides REST API endpoints for programmatic access to the RAG system.

**Features:**
- ✅ Document upload, list, retrieve, update, delete (CRUD)
- ✅ Multimodal query endpoint (text + images)
- ✅ Image serving for page screenshots
- ✅ Background task processing for uploads
- ✅ Auto-generated API documentation (Swagger UI)
- ✅ Async/await support for efficiency

### Quick Start

```bash
# Start API server
make api

# Access documentation
# - Swagger UI: http://localhost:8000/docs
# - ReDoc: http://localhost:8000/redoc
# - Health: http://localhost:8000/health
```

### API Endpoints

#### Document Management

**Upload Document**
```bash
curl -X POST "http://localhost:8000/documents" \
  -F "file=@manual.pdf"
```

**List Documents**
```bash
curl "http://localhost:8000/documents"
```

**Get Document Details**
```bash
curl "http://localhost:8000/documents/{doc_id}"
```

**Delete Document**
```bash
curl -X DELETE "http://localhost:8000/documents/{doc_id}"
```

**Update Document**
```bash
curl -X PUT "http://localhost:8000/documents/{doc_id}" \
  -F "file=@new_manual.pdf"
```

#### Query

**Query Documents**
```bash
curl -X POST "http://localhost:8000/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How to install?",
    "similarity_top_k": 3,
    "include_images": true
  }'
```

#### Images

**Get Page Image**
```bash
curl "http://localhost:8000/images/{doc_id}/{page_number}" \
  --output page.jpg
```

### Python Client Example

```python
import requests

API_URL = "http://localhost:8000"

# Upload document
with open("manual.pdf", "rb") as f:
    response = requests.post(
        f"{API_URL}/documents",
        files={"file": f}
    )
    doc_id = response.json()["document_id"]

# Query
response = requests.post(
    f"{API_URL}/query",
    json={
        "query": "How to install?",
        "similarity_top_k": 3
    }
)
print(response.json()["answer"])

# List documents
response = requests.get(f"{API_URL}/documents")
print(response.json())

# Delete document
requests.delete(f"{API_URL}/documents/{doc_id}")
```

### Testing the API

```bash
# Run example Python script
python examples/api_usage.py

# Or use the shell script
./examples/test_api.sh
```

### Production Deployment

```bash
# Multi-worker mode
make api-prod

# Or manually with 4 workers
cd src && uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 4
```

For detailed API documentation, see [`src/api/README.md`](src/api/README.md).

---

## Project Structure

```
Mag-/
├── Dockerfile              # Container image for FastAPI service
├── .dockerignore           # Docker build context exclusions
├── docker-compose.yml      # Local service orchestration (api/qdrant/minio)
├── src/                     # Application code
│   ├── config.py           # Centralized configuration
│   ├── app.py              # Streamlit web app (query interface)
│   ├── parse.py            # CLI batch document parser
│   ├── add_pdfs_batch.py   # Add PDFs to existing index
│   ├── api/                # FastAPI backend (NEW!)
│   │   ├── __init__.py     # API module exports
│   │   ├── main.py         # FastAPI app and configuration
│   │   ├── models.py       # Pydantic request/response models
│   │   ├── dependencies.py # Shared dependencies
│   │   ├── routes/         # API route modules
│   │   │   ├── documents.py # Document CRUD endpoints
│   │   │   ├── query.py    # Query endpoint
│   │   │   └── images.py   # Image serving endpoints
│   │   └── README.md       # API documentation
│   └── ingestion/          # Document management module
│       ├── __init__.py     # Module exports
│       ├── parser.py       # LlamaParse wrapper
│       ├── index_manager.py # Index lifecycle
│       ├── document_manager.py # CRUD operations
│       └── README.md       # Module documentation
├── examples/               # Usage examples (NEW!)
│   ├── api_usage.py        # Python API client example
│   └── test_api.sh         # Shell script for testing API
├── scripts/
│   ├── test_ingestion.py   # Ingestion test suite
│   ├── install-hooks.sh    # Git hook installer
│   └── clean.sh            # Cleanup utility
├── tests/                  # Test suite (43+ tests)
│   ├── test_config.py
│   ├── test_parse.py
│   ├── test_app.py
│   └── ...
├── data/                   # Input PDFs (your documents)
├── storage/                # Vector index (generated)
├── data_images/            # Page images (generated, by doc_id)
├── docs.md                 # LlamaIndex documentation reference
├── Makefile                # Common commands
├── requirements.txt        # Python dependencies
└── pytest.ini              # Test configuration
```

---

## Development

### Common Commands

```bash
make help               # Show all commands
make run                # Run Streamlit app
make api                # Run FastAPI server (development mode)
make api-prod           # Run FastAPI server (production mode with workers)
docker compose up -d api  # Run API in Docker
docker compose up -d api qdrant  # Run API + local Qdrant
make parse              # Parse all PDFs (fresh index)
make add-batch          # Add PDFs to existing index
make test               # Run full test suite
make test-quick         # Fast tests (no coverage)
make clean              # Clean Python cache
make clean-storage      # Clear vector index
make format             # Format code (black + isort)
make lint               # Lint code (flake8 + mypy)
docker compose down     # Stop Docker services
```

### Git Hook

Pre-commit hook runs tests automatically:

```bash
git commit -m "changes"  # Runs tests first
git commit --no-verify   # Skip tests (not recommended)

# Reinstall hook
./scripts/install-hooks.sh
```

---

## Testing

### Full Test Suite

```bash
make test               # Run all tests with coverage
make test-verbose       # Verbose output
make test-cov           # Coverage report (HTML + terminal)
make test-quick         # Fast (no coverage)
```

### Ingestion Module Tests

```bash
python scripts/test_ingestion.py
```

Tests all CRUD operations:
- ✅ Adding documents
- ✅ Listing documents
- ✅ Getting document info
- ✅ SOURCE relationship tracking
- ✅ Updating documents
- ✅ Deleting documents (index + images)

### Manual Testing (Python REPL)

```python
import asyncio
from src.ingestion import add_document, list_documents, delete_document

# Add a document
result = asyncio.run(add_document("data/manual.pdf"))
print(result)  # See document ID, page count

# List all
docs = list_documents()
print(f"Total documents: {len(docs)}")

# Delete
delete_document(result['document_id'])
```

### Validation Checklist

After running `make parse`, verify:

```bash
# 1. Check ref_doc_info is populated (SOURCE relationships work)
python3 -c "from src.ingestion import get_index; \
    print(f'Documents tracked: {len(get_index().ref_doc_info)}')"

# 2. Images exist in doc-specific directories
ls data_images/  # Should show UUID directories

# 3. Streamlit queries work
make run  # Test querying documents

# 4. Deletion works
python scripts/test_ingestion.py  # Run full test suite
```

**Success Criteria:**
- ✅ `ref_doc_info` shows tracked documents
- ✅ Images in `data_images/{doc_id}/` directories
- ✅ Streamlit finds and displays documents
- ✅ Test suite passes all tests
- ✅ Deletion removes index entries + images

---

## Tech Stack

- **Qdrant Cloud** - Managed vector database for embeddings and images
- **LlamaIndex** - RAG framework with document tracking
- **LlamaParse** - PDF parsing with multimodal extraction
- **OpenAI/Gemini** - LLM and embeddings
- **Streamlit** - Web UI for query interface
- **FastAPI** - REST API for document management
- **Uvicorn** - ASGI server for FastAPI
- **Pydantic** - Data validation and serialization
- **pytest** - Testing framework

---

## Performance Optimizations

### Image Compression
To handle Qdrant Cloud's 32MB payload limit and ensure fast uploads:

- **Resizing**: Images resized to 800px max (maintains aspect ratio)
- **JPEG Compression**: 60% quality (balances size vs quality)
- **Data URI Format**: Proper `data:image/jpeg;base64,` prefix for browser display
- **Result**: ~200-300KB per page (vs 2-3MB uncompressed)

### Batch Upload Strategy
- **Batch Size**: 2 pages per upload
- **Timeout**: 300 seconds (5 minutes) for reliable uploads
- **Retry Logic**: Built into Qdrant client
- **Progress Tracking**: Shows batch progress during indexing

### Embedding Optimization
- **Excluded Metadata**: Images excluded from embedding API calls
- **Token Limit**: Prevents "Requested X tokens, max 300K" errors
- **Cost Savings**: Only text is embedded, images stored separately

### Configuration
```python
# image_storage.py
max_size = 800   # pixels
quality = 60     # JPEG quality %

# parse.py
batch_size = 2   # pages per upload

# vector_store.py
timeout = 300    # seconds
```

---

## Next Steps

### Completed ✅
- [x] Qdrant Cloud integration
- [x] Compressed base64 image storage
- [x] Smart batch uploads (2 pages at a time)
- [x] Image compression (800px, 60% quality)
- [x] Embedding optimization (excludes images)
- [x] Document tracking with SOURCE relationships
- [x] FastAPI REST API
- [x] Streamlit UI with image display
- [x] Multi-worker support
- [x] CRUD operations

### Production Enhancements
- [ ] Monitoring & logging (Prometheus, Grafana)
- [ ] Authentication & authorization
- [ ] Rate limiting
- [ ] Caching layer (Redis)
- [ ] CI/CD pipeline
- [ ] Automated testing in pipeline

### Optional Improvements
- [ ] Migrate to S3 for images (if base64 becomes limiting)
- [ ] Add web scraping for non-PDF sources
- [ ] Implement semantic caching
- [ ] Add support for more document types (DOCX, HTML, etc.)

---

## Additional Documentation

- [`src/ingestion/README.md`](src/ingestion/README.md) - Detailed module API documentation
- [`VECTOR_DB_MIGRATION.md`](VECTOR_DB_MIGRATION.md) - Vector database migration guide

---

## License

All rights reserved.
