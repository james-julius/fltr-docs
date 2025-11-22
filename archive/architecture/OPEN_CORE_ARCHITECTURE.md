# Open-Core Document Processing Architecture

**Version:** 1.0
**Status:** Design Proposal
**Last Updated:** 2025-11-09

---

## 🎯 Vision

Create a **pluggable, driver-based document processing system** that allows users to:

- **Input from any source**: Local files, S3, R2, GCS, Dropbox, Google Drive, URLs
- **Parse any format**: PDF, DOCX, HTML, JSON via custom parsers (Unstructured, LlamaParse, etc.)
- **Output to any driver**: Milvus, Pinecone, Weaviate, Qdrant, PostgreSQL, Elasticsearch
- **Use any embedding provider**: OpenAI, Voyage, Cohere, local models
- **Configure via simple YAML/JSON**: No code changes needed

---

## 🏆 Goals

### For Users
- ✅ **No vendor lock-in** - Bring your own cloud, database, AI provider
- ✅ **Plug-and-play** - Add new drivers without forking codebase
- ✅ **Config-driven** - Switch providers via config file, not code
- ✅ **Test locally** - Use LocalFileDriver + Milvus Lite for development

### For Developers
- ✅ **Clear abstractions** - Well-defined interfaces for all components
- ✅ **Easy testing** - Mock any driver for unit tests
- ✅ **Type-safe** - Full type hints and validation
- ✅ **Extensible** - Add custom parsers/drivers without modifying core

### For Business
- ✅ **Open-core model** - Community edition + enterprise features
- ✅ **Partner ecosystem** - Integrations with Pinecone, Weaviate, etc.
- ✅ **Wider adoption** - Not tied to specific tech stack
- ✅ **Premium upsell** - Enterprise drivers (Dropbox, Google Drive, LlamaParse)

---

## 📐 Architecture Overview

### Current State Analysis

**✅ Good Examples:**
- **Vision Service** ([vision_service.py](modal/services/vision_service.py:16-228)) - Already plugin-based with multiple providers (GPT-4V, Claude, Gemini)
- **Service Layer** - Clean separation of concerns
- **Configuration** - Pydantic-based settings

**❌ Tightly Coupled:**
- **R2 Storage** - Hardcoded in 10+ files with direct boto3 calls
- **Milvus** - No abstraction for other vector databases
- **OpenAI Embeddings** - Single provider, no fallback
- **Document Parsers** - Simple if/else, not extensible

### Proposed Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     FLTR Document Processing                 │
│                         Pipeline                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Core Abstraction Layer                    │
│  (Abstract interfaces for all pluggable components)          │
└─────────────────────────────────────────────────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Storage    │ │   Parsers    │ │ Vector Store │ │  Embeddings  │
│   Drivers    │ │              │ │   Drivers    │ │   Providers  │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
│                │                │                │
├─ R2           ├─ PyMuPDF4LLM   ├─ Milvus        ├─ OpenAI
├─ S3           ├─ Docling       ├─ Pinecone      ├─ Voyage
├─ GCS          ├─ Unstructured  ├─ Weaviate      ├─ Cohere
├─ Local        ├─ LlamaParse    ├─ Qdrant        ├─ HuggingFace
├─ Dropbox*     ├─ Markdown      ├─ PostgreSQL    └─ Ollama
└─ G Drive*     └─ Custom        └─ Elasticsearch

* = Enterprise only
```

---

## 🔌 Core Abstractions

### 1. Storage Drivers

**Purpose:** Abstract file storage/retrieval from any source

**Interface:**
```python
from abc import ABC, abstractmethod
from typing import List, Optional

class StorageDriver(ABC):
    """Abstract interface for file storage providers"""

    @abstractmethod
    async def download(self, path: str) -> bytes:
        """Download file as bytes"""
        pass

    @abstractmethod
    async def upload(self, path: str, data: bytes, metadata: Optional[dict] = None) -> str:
        """Upload file, return URL/key"""
        pass

    @abstractmethod
    async def exists(self, path: str) -> bool:
        """Check if file exists"""
        pass

    @abstractmethod
    async def list(self, prefix: str, limit: int = 1000) -> List[str]:
        """List files with prefix"""
        pass

    @abstractmethod
    async def delete(self, path: str) -> None:
        """Delete file"""
        pass

    @abstractmethod
    async def get_presigned_url(self, path: str, expires_in: int = 3600) -> str:
        """Generate presigned URL for direct upload/download"""
        pass
```

**Implementations:**
- `R2StorageDriver` - Cloudflare R2 (current default)
- `S3StorageDriver` - AWS S3
- `GCSStorageDriver` - Google Cloud Storage
- `AzureBlobDriver` - Azure Blob Storage
- `LocalFileDriver` - Local filesystem (for testing)
- `DropboxDriver` - Dropbox (enterprise)
- `GoogleDriveDriver` - Google Drive (enterprise)

**Configuration Example:**
```yaml
storage:
  input:
    driver: s3
    config:
      bucket: my-input-bucket
      region: us-east-1
      credentials:
        access_key_id: ${AWS_ACCESS_KEY_ID}
        secret_access_key: ${AWS_SECRET_ACCESS_KEY}

  output:
    driver: r2
    config:
      account_id: ${R2_ACCOUNT_ID}
      bucket: my-processed-files
      credentials:
        access_key_id: ${R2_ACCESS_KEY_ID}
        secret_access_key: ${R2_SECRET_ACCESS_KEY}
```

---

### 2. Document Parsers

**Purpose:** Parse any document format into structured data

**Interface:**
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Any, Optional

@dataclass
class ParsedDocument:
    """Standardized parsed document output"""
    title: str
    content: str              # Markdown text
    format: str               # Parser used
    metadata: Dict[str, Any]
    images: List[Dict[str, Any]] = None
    tables: List[Dict[str, Any]] = None

class DocumentParser(ABC):
    """Abstract interface for document parsers"""

    @property
    @abstractmethod
    def supported_formats(self) -> List[str]:
        """File extensions this parser supports (e.g., ['.pdf', '.docx'])"""
        pass

    @property
    @abstractmethod
    def parser_name(self) -> str:
        """Unique identifier for this parser"""
        pass

    @abstractmethod
    async def parse(
        self,
        file_bytes: bytes,
        filename: str,
        **kwargs
    ) -> ParsedDocument:
        """Parse document into structured format"""
        pass

    async def supports(self, filename: str) -> bool:
        """Check if parser supports this file"""
        ext = os.path.splitext(filename)[1].lower()
        return ext in self.supported_formats
```

**Parser Registry:**
```python
class ParserRegistry:
    """Manages document parser selection and priority"""

    def __init__(self):
        self._parsers: Dict[str, DocumentParser] = {}
        self._priority: List[str] = []

    def register(self, parser: DocumentParser, priority: int = 100):
        """Register a parser with priority (lower = higher priority)"""
        pass

    def get_parser(self, filename: str) -> Optional[DocumentParser]:
        """Get best parser for filename based on priority"""
        pass

    def list_parsers(self) -> List[str]:
        """List all registered parsers"""
        pass
```

**Implementations:**
- `PyMuPDF4LLMParser` - PDF with semantic structure (current default)
- `DoclingParser` - Office formats (DOCX, PPTX, XLSX)
- `UnstructuredParser` - Multi-format parser (Unstructured.io)
- `LlamaParseParser` - Premium PDF parser (enterprise)
- `MarkdownParser` - Native Markdown
- `HTMLParser` - Web pages
- `JSONParser` - JSON documents
- `CustomParser` - User-provided parser

**Configuration Example:**
```yaml
parsers:
  priority:
    - pymupdf4llm      # Try first for PDFs
    - llamaparse       # Premium fallback (enterprise)
    - unstructured     # Generic fallback

  config:
    pymupdf4llm:
      use_vision: true
      vision_provider: gemini
      extract_images: true
      dpi: 150

    unstructured:
      strategy: hi_res
      extract_images: true

    llamaparse:
      api_key: ${LLAMAPARSE_API_KEY}
      premium_mode: true
```

---

### 3. Vector Store Drivers

**Purpose:** Store and search document embeddings in any vector database

**Interface:**
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Any, Optional

@dataclass
class VectorData:
    """Standardized vector data format"""
    id: str
    vector: List[float]
    metadata: Dict[str, Any]
    text: Optional[str] = None

@dataclass
class SearchResult:
    """Vector search result"""
    id: str
    score: float
    metadata: Dict[str, Any]
    text: Optional[str] = None

class VectorStoreDriver(ABC):
    """Abstract interface for vector databases"""

    @abstractmethod
    async def create_collection(
        self,
        name: str,
        dimension: int,
        metric: str = "cosine",
        schema: Optional[Dict] = None
    ) -> None:
        """Create a new collection/index"""
        pass

    @abstractmethod
    async def insert(
        self,
        collection: str,
        vectors: List[VectorData]
    ) -> int:
        """Insert vectors, return count inserted"""
        pass

    @abstractmethod
    async def search(
        self,
        collection: str,
        query_vector: List[float],
        top_k: int = 10,
        filter: Optional[Dict] = None
    ) -> List[SearchResult]:
        """Search for similar vectors"""
        pass

    @abstractmethod
    async def delete(
        self,
        collection: str,
        filter: Dict[str, Any]
    ) -> int:
        """Delete vectors matching filter, return count deleted"""
        pass

    @abstractmethod
    async def collection_stats(self, collection: str) -> Dict[str, Any]:
        """Get collection statistics"""
        pass
```

**Implementations:**
- `MilvusDriver` - Milvus (current default)
- `PineconeDriver` - Pinecone
- `WeaviateDriver` - Weaviate
- `QdrantDriver` - Qdrant
- `PostgresVectorDriver` - PostgreSQL with pgvector
- `ElasticsearchDriver` - Elasticsearch
- `ChromaDriver` - ChromaDB

**Configuration Example:**
```yaml
vectorstore:
  driver: pinecone
  config:
    api_key: ${PINECONE_API_KEY}
    environment: us-east-1-aws
    index: documents
    dimension: 1536
    metric: cosine
    namespace: production

  # Alternative: Milvus
  # driver: milvus
  # config:
  #   uri: ${MILVUS_URI}
  #   token: ${MILVUS_TOKEN}
  #   collection: fltr_documents
```

---

### 4. Embedding Providers

**Purpose:** Generate vector embeddings from text using any provider

**Interface:**
```python
from abc import ABC, abstractmethod
from typing import List

class EmbeddingProvider(ABC):
    """Abstract interface for embedding providers"""

    @property
    @abstractmethod
    def dimension(self) -> int:
        """Embedding dimension size"""
        pass

    @property
    @abstractmethod
    def model_name(self) -> str:
        """Model identifier"""
        pass

    @abstractmethod
    async def embed(self, texts: List[str]) -> List[List[float]]:
        """Embed multiple texts (batched)"""
        pass

    async def embed_single(self, text: str) -> List[float]:
        """Embed single text"""
        result = await self.embed([text])
        return result[0]

    @property
    def max_batch_size(self) -> int:
        """Maximum texts per batch"""
        return 100
```

**Implementations:**
- `OpenAIEmbeddingProvider` - OpenAI (current default)
- `VoyageAIProvider` - Voyage AI
- `CohereProvider` - Cohere
- `HuggingFaceProvider` - HuggingFace models (local or API)
- `OllamaProvider` - Ollama (local)
- `AzureOpenAIProvider` - Azure OpenAI

**Configuration Example:**
```yaml
embeddings:
  provider: voyage
  config:
    api_key: ${VOYAGE_API_KEY}
    model: voyage-2
    batch_size: 100

  # Alternative: OpenAI
  # provider: openai
  # config:
  #   api_key: ${OPENAI_API_KEY}
  #   model: text-embedding-3-small
  #   dimensions: 1536
```

---

## 📦 Package Structure

### Open-Core Organization

```
fltr/                              # Main repository
├── packages/
│   ├── fltr-core/                 # Open-source core (MIT License)
│   │   ├── fltr/
│   │   │   ├── __init__.py
│   │   │   ├── drivers/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── storage/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── base.py         # StorageDriver ABC
│   │   │   │   │   ├── r2.py
│   │   │   │   │   ├── s3.py
│   │   │   │   │   ├── gcs.py
│   │   │   │   │   ├── azure.py
│   │   │   │   │   └── local.py
│   │   │   │   ├── vectorstore/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── base.py         # VectorStoreDriver ABC
│   │   │   │   │   ├── milvus.py
│   │   │   │   │   ├── pinecone.py
│   │   │   │   │   ├── qdrant.py
│   │   │   │   │   ├── weaviate.py
│   │   │   │   │   └── postgres.py
│   │   │   │   └── embeddings/
│   │   │   │       ├── __init__.py
│   │   │   │       ├── base.py         # EmbeddingProvider ABC
│   │   │   │       ├── openai.py
│   │   │   │       ├── voyage.py
│   │   │   │       ├── cohere.py
│   │   │   │       └── huggingface.py
│   │   │   ├── parsers/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py             # DocumentParser ABC
│   │   │   │   ├── registry.py         # ParserRegistry
│   │   │   │   ├── pymupdf_parser.py
│   │   │   │   ├── docling_parser.py
│   │   │   │   ├── markdown_parser.py
│   │   │   │   ├── html_parser.py
│   │   │   │   └── json_parser.py
│   │   │   ├── pipeline/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── processor.py        # DocumentProcessor
│   │   │   │   ├── chunker.py
│   │   │   │   └── orchestrator.py
│   │   │   ├── config/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── schema.py           # Pydantic models
│   │   │   │   └── loader.py           # YAML/JSON loader
│   │   │   └── utils/
│   │   │       ├── __init__.py
│   │   │       └── registry.py         # Plugin discovery
│   │   ├── tests/
│   │   ├── docs/
│   │   ├── examples/
│   │   ├── pyproject.toml
│   │   └── README.md
│   │
│   └── fltr-enterprise/               # Proprietary features (Commercial License)
│       ├── fltr_enterprise/
│       │   ├── __init__.py
│       │   ├── drivers/
│       │   │   ├── storage/
│       │   │   │   ├── dropbox.py
│       │   │   │   ├── google_drive.py
│       │   │   │   ├── onedrive.py
│       │   │   │   └── box.py
│       │   │   └── vectorstore/
│       │   │       └── elasticsearch.py  # Enterprise features
│       │   ├── parsers/
│       │   │   ├── llamaparse.py        # Premium parser
│       │   │   └── azure_di.py          # Azure Document Intelligence
│       │   └── features/
│       │       ├── sso.py
│       │       ├── audit.py
│       │       └── compliance.py
│       ├── tests/
│       ├── pyproject.toml
│       └── README.md
│
├── modal/                             # Current Modal serverless code
│   └── services/                      # (Will migrate to use fltr-core)
│
├── fastapi/                           # Current FastAPI backend
│   └── services/                      # (Will migrate to use fltr-core)
│
└── nextjs/                            # Frontend (unchanged)
```

---

## ⚙️ Configuration System

### Unified Config Schema

**File:** `fltr/config/schema.py`

```python
from pydantic import BaseModel, Field
from typing import Dict, Any, List, Optional, Literal

class StorageConfig(BaseModel):
    """Storage driver configuration"""
    driver: str = Field(..., description="Storage driver name")
    config: Dict[str, Any] = Field(default_factory=dict)

class ParserConfig(BaseModel):
    """Parser configuration"""
    priority: List[str] = Field(default=["pymupdf4llm", "docling"])
    config: Dict[str, Dict[str, Any]] = Field(default_factory=dict)

class VectorStoreConfig(BaseModel):
    """Vector store configuration"""
    driver: str = Field(..., description="Vector store driver name")
    config: Dict[str, Any] = Field(default_factory=dict)

class EmbeddingConfig(BaseModel):
    """Embedding provider configuration"""
    provider: str = Field(..., description="Embedding provider name")
    config: Dict[str, Any] = Field(default_factory=dict)

class ProcessingConfig(BaseModel):
    """Document processing configuration"""
    chunk_size: int = Field(default=1000)
    chunk_overlap: int = Field(default=200)
    max_workers: int = Field(default=10)
    batch_size: int = Field(default=100)

class FLTRConfig(BaseModel):
    """Complete FLTR configuration"""
    storage_input: StorageConfig
    storage_output: StorageConfig
    parsers: ParserConfig
    vectorstore: VectorStoreConfig
    embeddings: EmbeddingConfig
    processing: ProcessingConfig = Field(default_factory=ProcessingConfig)

    class Config:
        extra = "allow"  # Allow custom fields for plugins
```

### YAML Configuration

**File:** `config.yaml`

```yaml
# FLTR Document Processing Configuration

# Input storage (where documents are uploaded)
storage_input:
  driver: s3
  config:
    bucket: my-input-bucket
    region: us-east-1
    credentials:
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}

# Output storage (for processed images, cached data)
storage_output:
  driver: r2
  config:
    account_id: ${R2_ACCOUNT_ID}
    bucket: fltr-processed
    credentials:
      access_key_id: ${R2_ACCESS_KEY_ID}
      secret_access_key: ${R2_SECRET_ACCESS_KEY}

# Document parsers (in priority order)
parsers:
  priority:
    - pymupdf4llm      # Try PyMuPDF4LLM first for PDFs
    - unstructured     # Fallback to Unstructured
    - docling          # Last resort

  config:
    pymupdf4llm:
      use_vision: true
      vision_provider: gemini
      extract_images: true
      dpi: 150
      margins: [0, 50, 0, 50]

    unstructured:
      strategy: hi_res
      extract_images: true
      include_page_breaks: true

    docling:
      extract_tables: true
      extract_figures: true

# Vector database
vectorstore:
  driver: pinecone
  config:
    api_key: ${PINECONE_API_KEY}
    environment: us-east-1-aws
    index: documents
    dimension: 1024
    metric: cosine
    namespace: production

# Embedding provider
embeddings:
  provider: voyage
  config:
    api_key: ${VOYAGE_API_KEY}
    model: voyage-2
    batch_size: 100
    input_type: document

# Processing settings
processing:
  chunk_size: 1000
  chunk_overlap: 200
  max_workers: 10
  batch_size: 100

# Vision processing (for multimodal)
vision:
  provider: gemini
  fallback_provider: gpt4v
  config:
    gemini:
      api_key: ${GOOGLE_API_KEY}
      model: gemini-2.0-flash-exp
    gpt4v:
      api_key: ${OPENAI_API_KEY}
      model: gpt-4o
```

### Environment Variables (Alternative)

```bash
# .env

# Storage
STORAGE_INPUT_DRIVER=s3
STORAGE_INPUT_BUCKET=my-input-bucket
STORAGE_INPUT_REGION=us-east-1

STORAGE_OUTPUT_DRIVER=r2
STORAGE_OUTPUT_BUCKET=fltr-processed

# Parsers
PARSER_PRIORITY=pymupdf4llm,unstructured,docling
PYMUPDF_USE_VISION=true
PYMUPDF_VISION_PROVIDER=gemini

# Vector Store
VECTORSTORE_DRIVER=pinecone
PINECONE_API_KEY=xxx
PINECONE_INDEX=documents

# Embeddings
EMBEDDING_PROVIDER=voyage
VOYAGE_API_KEY=xxx
VOYAGE_MODEL=voyage-2

# Processing
CHUNK_SIZE=1000
CHUNK_OVERLAP=200
```

---

## 🔌 Plugin System

### Entry Point Discovery

**File:** `pyproject.toml`

```toml
[project.entry-points."fltr.storage_drivers"]
s3 = "fltr.drivers.storage.s3:S3StorageDriver"
r2 = "fltr.drivers.storage.r2:R2StorageDriver"
gcs = "fltr.drivers.storage.gcs:GCSStorageDriver"
local = "fltr.drivers.storage.local:LocalFileDriver"

# Enterprise drivers
dropbox = "fltr_enterprise.drivers.storage.dropbox:DropboxDriver"
google_drive = "fltr_enterprise.drivers.storage.google_drive:GoogleDriveDriver"

[project.entry-points."fltr.parsers"]
pymupdf4llm = "fltr.parsers.pymupdf_parser:PyMuPDF4LLMParser"
docling = "fltr.parsers.docling_parser:DoclingParser"
unstructured = "fltr.parsers.unstructured_parser:UnstructuredParser"

# Enterprise parsers
llamaparse = "fltr_enterprise.parsers.llamaparse:LlamaParseParser"

[project.entry-points."fltr.vectorstore_drivers"]
milvus = "fltr.drivers.vectorstore.milvus:MilvusDriver"
pinecone = "fltr.drivers.vectorstore.pinecone:PineconeDriver"
qdrant = "fltr.drivers.vectorstore.qdrant:QdrantDriver"
weaviate = "fltr.drivers.vectorstore.weaviate:WeaviateDriver"

[project.entry-points."fltr.embedding_providers"]
openai = "fltr.drivers.embeddings.openai:OpenAIEmbeddingProvider"
voyage = "fltr.drivers.embeddings.voyage:VoyageAIProvider"
cohere = "fltr.drivers.embeddings.cohere:CohereProvider"
```

### Custom User Plugins

**Example:** Custom PDF parser using a proprietary tool

```python
# my_company_parser.py
from fltr.parsers.base import DocumentParser, ParsedDocument
from typing import List

class MyCompanyParser(DocumentParser):
    """Custom parser using proprietary PDF tool"""

    @property
    def supported_formats(self) -> List[str]:
        return [".pdf"]

    @property
    def parser_name(self) -> str:
        return "mycompany"

    async def parse(self, file_bytes: bytes, filename: str, **kwargs) -> ParsedDocument:
        # Use proprietary parsing logic
        from mycompany.pdf import parse_pdf

        result = parse_pdf(file_bytes)

        return ParsedDocument(
            title=result.title,
            content=result.markdown,
            format="mycompany",
            metadata=result.metadata,
            images=result.images
        )

# Register plugin programmatically
from fltr.parsers import get_registry

registry = get_registry()
registry.register(MyCompanyParser(), priority=50)  # High priority
```

**Or via entry points:**

```toml
# pyproject.toml
[project.entry-points."fltr.parsers"]
mycompany = "mycompany.parsers:MyCompanyParser"
```

---

## 🚀 Usage Examples

### Example 1: Simple Processing (Python API)

```python
from fltr import DocumentProcessor
from fltr.config import load_config

# Load configuration from YAML
config = load_config("config.yaml")

# Initialize processor
processor = DocumentProcessor(config)

# Process a document
result = await processor.process(
    file_bytes=pdf_bytes,
    filename="annual_report.pdf",
    metadata={"year": 2024, "department": "finance"}
)

print(f"Processed {result.chunk_count} chunks")
print(f"Extracted {len(result.images)} images")
print(f"Stored in {result.vectorstore}")
```

### Example 2: Custom Pipeline

```python
from fltr.drivers.storage import S3StorageDriver, R2StorageDriver
from fltr.drivers.vectorstore import PineconeDriver
from fltr.drivers.embeddings import VoyageAIProvider
from fltr.parsers import PyMuPDF4LLMParser
from fltr.pipeline import Pipeline

# Compose custom pipeline
pipeline = Pipeline(
    storage_input=S3StorageDriver(
        bucket="uploads",
        region="us-east-1"
    ),
    storage_output=R2StorageDriver(
        account_id="xxx",
        bucket="processed"
    ),
    parser=PyMuPDF4LLMParser(
        use_vision=True,
        vision_provider="gemini"
    ),
    vectorstore=PineconeDriver(
        api_key="xxx",
        index="documents"
    ),
    embeddings=VoyageAIProvider(
        api_key="xxx",
        model="voyage-2"
    )
)

# Run pipeline
await pipeline.run("s3://uploads/document.pdf")
```

### Example 3: CLI Usage

```bash
# Process single file
fltr process document.pdf --config config.yaml

# Process entire directory
fltr process ./documents/ --recursive --config config.yaml

# Switch drivers on the fly (overrides config)
fltr process document.pdf \
  --storage-driver s3 \
  --vectorstore-driver pinecone \
  --embedding-provider voyage

# Batch process from S3
fltr batch s3://my-bucket/documents/ \
  --output-vectorstore pinecone \
  --workers 10

# List available drivers
fltr drivers list
fltr drivers list --type storage
fltr drivers list --type parsers

# Validate configuration
fltr config validate config.yaml

# Show current configuration
fltr config show
```

### Example 4: FastAPI Integration

```python
from fastapi import FastAPI, UploadFile
from fltr import DocumentProcessor
from fltr.config import load_config

app = FastAPI()
config = load_config("config.yaml")
processor = DocumentProcessor(config)

@app.post("/documents/process")
async def process_document(file: UploadFile):
    """Process uploaded document"""

    file_bytes = await file.read()

    result = await processor.process(
        file_bytes=file_bytes,
        filename=file.filename
    )

    return {
        "chunks": result.chunk_count,
        "images": len(result.images),
        "status": "success"
    }
```

---

## 🛣️ Migration Path

### Phase 1: Create Abstractions (Week 1-2)
**Goal:** Build foundation without breaking existing code

**Tasks:**
1. Create abstract base classes:
   - `StorageDriver` ([vision_service.py:16-23](modal/services/vision_service.py) as template)
   - `VectorStoreDriver`
   - `EmbeddingProvider`
   - `DocumentParser`

2. Create `fltr-core` package structure
3. Write comprehensive tests for abstractions
4. Document interfaces with examples

**Deliverables:**
- `fltr/drivers/storage/base.py`
- `fltr/drivers/vectorstore/base.py`
- `fltr/drivers/embeddings/base.py`
- `fltr/parsers/base.py`
- `fltr/config/schema.py`
- Test suite for abstractions

### Phase 2: Implement Default Drivers (Week 3-4)
**Goal:** Implement drivers for current stack (non-breaking)

**Tasks:**
1. Implement current drivers:
   - `R2StorageDriver` (refactor `modal/services/storage.py`)
   - `MilvusDriver` (refactor `modal/services/vector_store.py`)
   - `OpenAIEmbeddingProvider` (refactor `modal/services/embeddings.py`)
   - `PyMuPDF4LLMParser` (refactor `modal/services/multimodal_processor.py`)

2. Add alternative implementations:
   - `S3StorageDriver`
   - `LocalFileDriver`
   - `PineconeDriver`
   - `VoyageAIProvider`

3. Create ParserRegistry

**Deliverables:**
- Working R2, Milvus, OpenAI drivers (current stack)
- S3, Pinecone, Voyage alternatives
- Parser registry with priority system
- Integration tests

### Phase 3: Config System (Week 5)
**Goal:** YAML/JSON configuration with validation

**Tasks:**
1. Pydantic schema for all drivers
2. YAML/JSON loader with environment variable substitution
3. Config validation
4. Hot-reload support (for development)
5. Migration tool: current hardcoded → new config

**Deliverables:**
- `fltr/config/loader.py`
- Example `config.yaml`
- Migration script: `fltr config migrate`
- Documentation

### Phase 4: Plugin System (Week 6)
**Goal:** Enable third-party plugins

**Tasks:**
1. Entry point discovery system
2. Plugin registration API
3. Plugin validation (check interface compliance)
4. Example custom plugins
5. Plugin development guide

**Deliverables:**
- `fltr/utils/registry.py`
- Example plugins (custom parser, custom storage)
- Plugin developer documentation
- `fltr plugins list` CLI command

### Phase 5: Integrate with Existing Code (Week 7)
**Goal:** Migrate modal/ and fastapi/ to use fltr-core

**Tasks:**
1. Update `modal/modal_app.py` to use drivers
2. Update `fastapi/services/` to use drivers
3. Add config file support
4. Maintain backward compatibility (env vars still work)
5. Update deployment scripts

**Deliverables:**
- `modal/` using fltr-core drivers
- `fastapi/` using fltr-core drivers
- Backward compatible with current setup
- No breaking changes to API

### Phase 6: Open-Core Split (Week 8)
**Goal:** Separate open-source vs enterprise

**Tasks:**
1. Create `fltr-core` repo (MIT license)
2. Create `fltr-enterprise` repo (commercial license)
3. CI/CD for both packages
4. PyPI publication for fltr-core
5. Documentation site
6. Partner integrations (Pinecone, Weaviate docs)

**Deliverables:**
- Public `fltr-core` on PyPI
- Private `fltr-enterprise`
- docs.fltr.com site
- Partner integration examples

---

## 🧪 Testing Strategy

### Unit Tests
- Test each driver in isolation
- Mock external dependencies (APIs, databases)
- Test configuration validation
- Test plugin discovery

### Integration Tests
- Test full pipeline with LocalFileDriver + Milvus Lite
- Test driver switching (swap Milvus for Pinecone)
- Test parser fallback (primary fails → secondary)
- Test config hot-reload

### End-to-End Tests
- Process real PDFs through pipeline
- Verify vector storage
- Verify retrieval quality
- Performance benchmarks

---

## 📊 Success Metrics

### Technical Metrics
- ✅ **Zero breaking changes** during migration
- ✅ **100% test coverage** for abstractions
- ✅ **<5% performance overhead** from abstraction layer
- ✅ **Plugin registration in <10 lines** of code

### Adoption Metrics
- ✅ **10+ community drivers** within 6 months
- ✅ **5+ enterprise customers** using custom drivers
- ✅ **100+ GitHub stars** on fltr-core
- ✅ **Partner integrations** with 3+ vector DB vendors

---

## 🎓 Key Design Principles

### 1. Follow VisionService Pattern
The existing [vision_service.py](modal/services/vision_service.py:16-228) is **excellent**:
- Enum-based provider selection
- Dataclass configuration
- Unified service interface
- Easy to add providers
- Built-in fallback

**Use this as the template for all abstractions.**

### 2. Configuration Over Code
- Users should **never need to modify code** to switch providers
- Simple YAML/JSON config changes everything
- Environment variables for secrets

### 3. Sensible Defaults
- Should work out-of-box with zero config
- Default: Current stack (R2 + Milvus + OpenAI + PyMuPDF4LLM)
- Progressive enhancement via config

### 4. Type Safety
- Full type hints everywhere
- Pydantic validation for configs
- Runtime checks for plugin compliance

### 5. Testing First
- Every driver must have tests
- Mock external services
- LocalFileDriver + Milvus Lite for local testing

---

## 📚 Documentation Plan

### For Users
- **Getting Started** - Install, basic config, first document
- **Configuration Guide** - All config options explained
- **Driver Reference** - All built-in drivers documented
- **Migration Guide** - Moving from hardcoded to config

### For Plugin Developers
- **Plugin Development Guide** - How to create custom drivers
- **API Reference** - All abstract interfaces
- **Examples** - Custom storage, parser, vector store
- **Testing Plugins** - How to test your plugin

### For Contributors
- **Architecture Overview** - This document
- **Contributing Guide** - How to add new drivers
- **Code Style** - Type hints, docstrings, tests
- **Release Process** - PyPI, versioning, changelog

---

## 💼 Business Model

### Open-Source (MIT License)
- Core abstractions
- Built-in drivers:
  - Storage: R2, S3, GCS, Azure, Local
  - Vector: Milvus, Pinecone, Qdrant, Weaviate, PostgreSQL
  - Embeddings: OpenAI, Voyage, Cohere, HuggingFace
  - Parsers: PyMuPDF4LLM, Docling, Markdown, HTML

### Enterprise (Commercial License)
- Premium drivers:
  - Storage: Dropbox, Google Drive, OneDrive, Box
  - Parsers: LlamaParse, Azure Document Intelligence
  - Vector: Elasticsearch (enterprise features)
- Enterprise features:
  - SSO integration
  - Audit logging
  - Compliance tools (GDPR, HIPAA)
  - Priority support
  - SLA guarantees

### Pricing Model
- **Open-source**: Free forever
- **Enterprise**: $500/month per instance
- **Cloud**: Usage-based pricing (future)

---

## 🔒 Licensing

```
fltr-core/
├── LICENSE              # MIT License
├── NOTICE               # Copyright notice
└── README.md            # Open-source README

fltr-enterprise/
├── LICENSE              # Commercial License
├── EULA.md              # End-user license agreement
└── README.md            # Enterprise README
```

---

## 🤝 Community & Ecosystem

### Partner Integrations
- **Pinecone**: Official Pinecone driver, co-marketing
- **Weaviate**: Official Weaviate driver, documentation
- **Qdrant**: Integration examples
- **Unstructured**: Parser integration
- **LlamaParse**: Enterprise parser partnership

### Community Drivers
- Encourage community to build drivers
- Showcase in "Community Drivers" page
- Monthly "Driver of the Month" highlight
- Contributor rewards program

---

## 📝 Open Questions

1. **Naming:** Should it be `fltr-core` or `fltr` (with `fltr-enterprise`)?
2. **Versioning:** Semantic versioning? Pin core to enterprise?
3. **Distribution:** PyPI only, or also Docker images?
4. **CLI Tool:** Separate `fltr` CLI package, or included in core?
5. **Plugin Registry:** Centralized registry (like npm) or just entry points?

---

## 📅 Timeline

| Week | Phase | Deliverables |
|------|-------|--------------|
| 1-2 | Core Abstractions | Base classes, config schema, tests |
| 3-4 | Default Drivers | R2, Milvus, OpenAI, PyMuPDF4LLM drivers |
| 5 | Config System | YAML loader, validation, migration tool |
| 6 | Plugin System | Entry points, registration, examples |
| 7 | Integration | Migrate modal/ and fastapi/ to use drivers |
| 8 | Open-Core Split | Separate repos, CI/CD, PyPI, docs site |

**Total:** 8 weeks to full open-core release

---

## 🎯 Next Steps

1. **Review & Approve** - Team review of this architecture
2. **Prototype** - Build one complete driver (StorageDriver) as proof-of-concept
3. **Feedback** - Get community feedback on proposed interfaces
4. **Implement** - Follow migration path outlined above

---

**Document Status:** Ready for Review
**Last Updated:** 2025-11-09
**Author:** AI Architecture Team
**Reviewers Needed:** Engineering Team, Product Team, Community