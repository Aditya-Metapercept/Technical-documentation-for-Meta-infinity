# Data Ingestion Pipeline - Complete Guide

## Overview

The data ingestion pipeline processes documents through multiple stages: validation → storage → parsing → indexing → RDF processing. This guide explains each stage and how to use the pipeline.

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Upload (API)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              1. FILE VALIDATION                             │
│  - Format check (ALLOWED_EXTENSIONS)                        │
│  - Size check (MAX_FILE_SIZE: 500MB)                        │
│  - Type check (FILE_TYPE_CONFIG)                            │
│  - Domain validation                                        │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              2. STORAGE UPLOAD                              │
│  - Upload to S3/Connector                                   │
│  - Path: uploads/{domain}/{doc_id}/{filename}               │
│  - Store metadata                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              3. DATABASE RECORD                             │
│  - Create Document record                                   │
│  - Store metadata                                           │
│  - Set status: "pending"                                    │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              4. XML PROCESSING                              │
│  - Convert to XML (if needed)                               │
│  - Parse XML structure                                      │
│  - Extract metadata (XPath)                                 │
│  - Chunk content                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              5. OPENSEARCH INDEXING                         │
│  - Create document chunks                                   │
│  - Generate embeddings                                      │
│  - Index in OpenSearch                                      │
│  - Store metadata                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              6. RDF PROCESSING (Parallel)                   │
│  - Queue SQS task                                           │
│  - Process in background                                    │
│  - Update knowledge graph                                   │
│  - Set status: "completed"                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    COMPLETION                               │
│  - Document ready for search                                │
│  - Metadata indexed                                         │
│  - RDF graph updated                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Stage 1: File Validation

### Purpose
Validate file format, size, type, and domain before processing.

### Location
`app/ingestion/processors/document_processor.py`

### Key Methods

#### `validate_file()`
```python
def validate_file(self, file: UploadFile, format: str = None, domain: str = None) -> Dict[str, Any]:
    """
    Validates file type, format, domain and size
    
    Returns:
        Dict with file handling information
    
    Raises:
        UnsupportedDocumentTypeError: If file type not supported
    """
```

**Validation Checks**:
1. File extension against ALLOWED_EXTENSIONS
2. MIME type against VALID_MIME_TYPES
3. Format parameter against FILE_TYPE_CONFIG
4. Domain exists in database

**Supported Extensions**:
```python
ALLOWED_EXTENSIONS = {
    ".html", ".zip", ".xml", ".docx", 
    ".doc", ".md", ".mdx", ".markdown"
}
```

**Example**:
```python
processor = DocumentProcessor()
file = UploadFile(filename="document.xml")
result = processor.validate_file(file)

# Returns:
{
    "file_type": ".xml",
    "handling_type": "both",
    "is_container": False,
    "container_type": None,
    "format": ".xml",
    "domain": None
}
```

#### `validate_file_size()`
```python
async def validate_file_size(self, file: UploadFile, content_length: Optional[str] = None) -> bytes:
    """
    Validates file size and returns content if valid
    
    Raises:
        DocumentTooLargeError: If file exceeds MAX_FILE_SIZE (500MB)
    """
```

**Example**:
```python
content = await processor.validate_file_size(file)
# Returns file content as bytes
```

### Error Handling

```python
# Unsupported format
UnsupportedDocumentTypeError("Unsupported file format: .exe")

# File too large
DocumentTooLargeError("File exceeds maximum size of 500MB")

# Invalid domain
HTTPException(status_code=400, detail="Domain 'INVALID' not found")
```

---

## Stage 2: Storage Upload

### Purpose
Upload validated file to S3/Connector storage.

### Location
`app/ingestion/processors/document_processor.py`

### Key Methods

#### `classify_file()`
```python
def classify_file(self, file: UploadFile, content: bytes, domain: str = None, format: str = None) -> Dict[str, Any]:
    """
    Classifies file and prepares for storage
    
    Returns:
        Dict with file classification information
    """
```

**Returns**:
```python
{
    "document_id": "uuid-string",
    "filename": "document.xml",
    "content": b"file_content",
    "size": 1024,
    "content_type": "application/xml",
    "s3_prefix": "uploads/SPACE/uuid/",
    "is_container": False,
    "format": ".xml",
    "domain": "SPACE",
    "file_extension": ".xml"
}
```

#### `save_to_storage()`
```python
def save_to_storage(self, file_data, prefix: str) -> str:
    """
    Saves file to S3/Connector and returns storage key
    
    Storage Path Pattern:
        uploads/{domain}/{document_id}/{original_filename}
    """
```

**Example**:
```python
file_data = {
    "filename": "document.xml",
    "content": b"<xml>...</xml>",
    "content_type": "application/xml"
}

storage_key = processor.save_to_storage(file_data, "uploads/SPACE/doc-id/")
# Returns: "uploads/SPACE/doc-id/document.xml"
```

### Storage Structure

```
s3://metr-infinity-bucket/
├── uploads/
│   ├── SPACE/
│   │   ├── doc-id-1/
│   │   │   └── document.xml
│   │   └── doc-id-2/
│   │       └── document.docx
│   ├── TIME/
│   │   └── doc-id-3/
│   │       └── equipment.xml
│   └── POWER/
│       └── doc-id-4/
│           └── contract.xml
├── converted/
│   ├── SPACE/
│   │   └── doc-id-1/
│   │       └── converted.xml
│   └── POWER/
│       └── doc-id-4/
│           └── converted.xml
```

---

## Stage 3: Database Record

### Purpose
Create document record in PostgreSQL with metadata.

### Location
`app/ingestion/processors/document_processor.py`

### Key Methods

#### `create_or_update_document_record()`
```python
def create_or_update_document_record(self, document_data: Dict[str, Any], domain_obj=None) -> Dict[str, Any]:
    """
    Creates new document or updates existing one
    
    Returns:
        Dict with document_id and action (created/updated)
    """
```

**Document Data**:
```python
{
    "document_id": "uuid",
    "filename": "document.xml",
    "s3_path": "uploads/SPACE/uuid/document.xml",
    "status": "pending",
    "is_container": 0,
    "file_size": 1024,
    "content_type": "application/xml",
    "file_metadata": {...},
    "parsed_metadata": {...},
    "ontology_id": 1,
    "start_time": datetime.utcnow()
}
```

**Database Model**:
```python
class Document(Base):
    __tablename__ = "documents"
    
    id = Column(Integer, primary_key=True)
    document_id = Column(String, unique=True)
    filename = Column(String)
    s3_key = Column(String)
    s3_bucket = Column(String)
    status = Column(String, default="pending")  # pending, processing, completed, failed
    title = Column(String, nullable=True)
    abstract = Column(String, nullable=True)
    authors = Column(String, nullable=True)
    file_size = Column(Integer)
    content_type = Column(String)
    file_metadata = Column(JSON)
    dynamic_metadata = Column(JSON)
    is_container = Column(Integer, default=0)
    parent_document_id = Column(String, nullable=True)
    ontology_id = Column(Integer, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    processed_at = Column(DateTime, nullable=True)
```

**Example**:
```python
result = processor.create_or_update_document_record(
    {
        "document_id": "doc-123",
        "filename": "test.xml",
        "s3_path": "uploads/SPACE/doc-123/test.xml",
        "status": "pending",
        "file_size": 1024,
        "content_type": "application/xml"
    },
    domain_obj=domain
)

# Returns:
{
    "document_id": "doc-123",
    "is_update": False,
    "action": "created"
}
```

### Status Lifecycle

```
pending → processing → completed
              ↓
            failed
```

---

## Stage 4: XML Processing

### Purpose
Convert documents to XML, parse structure, extract metadata, and chunk content.

### Location
`app/ingestion/processors/xml_processor.py` and `app/ingestion/chunker.py`

### XML Conversion

#### `call_conversion_api()`
```python
def call_conversion_api(self, file_content: bytes, filename: str, format: str, complete_file_name: str) -> Dict[str, Any]:
    """
    Converts document to XML using external API
    
    Supported formats: .docx, .doc, .html, .md, .mdx
    """
```

**Conversion Flow**:
```
Input File (DOCX/HTML/MD)
    ↓
XML_CONVERSION_API_URL
    ↓
Download Converted XML
    ↓
Store in S3 (converted/{domain}/{doc_id}/)
    ↓
Index in OpenSearch
```

**Example**:
```python
processor = XMLProcessor()
result = processor.call_conversion_api(
    file_content=b"PK\x03\x04...",  # DOCX content
    filename="document.docx",
    format=".docx",
    complete_file_name="document.docx"
)

# Returns:
{
    "success": True,
    "download_url": "https://api.example.com/download/uuid",
    "format": ".docx"
}
```

### XML Parsing

#### `ConfigurableXMLChunker`
```python
class ConfigurableXMLChunker:
    def __init__(self, domain: str):
        """Initialize chunker with domain-specific config"""
    
    def process_and_chunk(self, xml_content: bytes) -> Dict[str, Any]:
        """
        Parses XML and extracts metadata using XPath
        
        Returns:
            Dict with title, authors, keywords, body, metadata
        """
```

**XPath Extraction**:
```python
# config/config.yaml defines XPath mappings
space:
  FES-Space:
    title:
      - "/topic/title"
    authors:
      - "/topic/prolog/author"
    keywords:
      - "/topic/prolog/metadata/keywords/*"
    body:
      - "/topic/body"
```

**Example**:
```python
chunker = ConfigurableXMLChunker(domain="SPACE")
result = chunker.process_and_chunk(xml_content)

# Returns:
{
    "title": "Document Title",
    "authors": ["Author 1", "Author 2"],
    "keywords": ["keyword1", "keyword2"],
    "body": "Document content...",
    "metadata": {
        "created_date": "2024-01-15",
        "prodinfo": "...",
        "othermeta": "..."
    },
    "chunks": [
        {"text": "chunk 1", "index": 0},
        {"text": "chunk 2", "index": 1}
    ]
}
```

### Chunking Strategy

**Semantic Chunking**:
- Splits content into meaningful chunks
- Default chunk size: 1024 tokens
- Overlap: 128 tokens
- Preserves document structure

**Example Chunks**:
```python
chunks = [
    {
        "text": "Introduction paragraph...",
        "index": 0,
        "metadata": {"section": "introduction"}
    },
    {
        "text": "Methods section...",
        "index": 1,
        "metadata": {"section": "methods"}
    },
    {
        "text": "Results section...",
        "index": 2,
        "metadata": {"section": "results"}
    }
]
```

---

## Stage 5: OpenSearch Indexing

### Purpose
Index document chunks in OpenSearch for full-text search and semantic search.

### Location
`app/ingestion/processors/document_indexer.py`

### Key Methods

#### `index_document()`
```python
async def index_document(self, s3_path: str, index_name: str, document_id: str, domain: str, abstract_type: str) -> Dict[str, Any]:
    """
    Indexes document in OpenSearch
    
    Returns:
        Dict with chunks_indexed and total_chunks
    """
```

**Example**:
```python
indexer = DocumentIndexer(opensearch_config, s3_bucket)
result = await indexer.index_document(
    s3_path="uploads/SPACE/doc-id/document.xml",
    index_name="metr-documents",
    document_id="doc-id",
    domain="SPACE",
    abstract_type="abstract"
)

# Returns:
{
    "success": True,
    "chunks_indexed": 10,
    "total_chunks": 10,
    "document_id": "doc-id"
}
```

### Index Mapping

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  },
  "mappings": {
    "properties": {
      "document_id": {"type": "keyword"},
      "title": {"type": "text"},
      "abstract": {"type": "text"},
      "content": {"type": "text"},
      "keywords": {"type": "keyword"},
      "domain": {"type": "keyword"},
      "chunk_index": {"type": "integer"},
      "embedding": {"type": "dense_vector", "dimension": 1536},
      "created_at": {"type": "date"},
      "metadata": {"type": "object", "enabled": false}
    }
  }
}
```

### Chunk Structure

```python
{
    "document_id": "doc-id",
    "title": "Document Title",
    "abstract": "Document abstract...",
    "content": "Chunk text content...",
    "keywords": ["keyword1", "keyword2"],
    "domain": "SPACE",
    "chunk_index": 0,
    "embedding": [0.123, 0.456, ...],  # 1536 dimensions
    "created_at": "2024-01-15T10:30:00Z",
    "metadata": {
        "section": "introduction",
        "page": 1,
        "source": "document.xml"
    }
}
```

---

## Stage 6: RDF Processing

### Purpose
Process document in knowledge graph (Neptune/GraphDB) in parallel.

### Location
`app/ingestion/sqs_tasks.py`

### SQS Queue

#### `send_process_rdf_graph()`
```python
def send_process_rdf_graph(document_id: str):
    """
    Queues RDF processing task in SQS
    
    Processes in background after indexing completes
    """
```

**Example**:
```python
from app.ingestion.sqs_tasks import send_process_rdf_graph

# Queue RDF processing
send_process_rdf_graph("doc-id")

# Worker processes task asynchronously
```

### RDF Processing Flow

```
Document Indexed
    ↓
Queue SQS Task
    ↓
Background Worker Picks Up
    ↓
Extract RDF Triples
    ↓
Update Knowledge Graph
    ↓
Set Status: "completed"
```

---

## Complete Workflow Example

### 1. Upload Document

```bash
curl -X POST http://localhost:8000/api/upload \
  -F "file=@document.xml" \
  -F "domain=SPACE"
```

### 2. API Endpoint

```python
# app/api/routes/upload.py
@router.post("/upload")
async def upload_documents(
    files: List[UploadFile],
    domain: str,
    format: Optional[str] = None,
    ontology_id: Optional[int] = None,
    db: Session = Depends(get_db)
):
    processor = DocumentProcessor()
    result = await processor.initial_processing(
        files=files,
        domain=domain,
        format=format,
        ontology_id=ontology_id,
        db=db
    )
    return result
```

### 3. Processing Pipeline

```python
# app/ingestion/processors/document_processor.py
async def initial_processing(self, files, domain, format, ontology_id, db):
    # 1. Validate domain and ontology
    domain_obj, ontology_obj = self.validate_domain_and_ontology(db, domain, ontology_id)
    
    # 2. Process each file
    for file in files:
        # Validate file
        file_info = self.validate_file(file, format, domain)
        content = await self.validate_file_size(file)
        
        # Upload to storage
        s3_path = self.save_to_storage(file_data, s3_prefix)
        
        # Create database record
        result = self.create_or_update_document_record(document_data, domain_obj)
        
        # Parse XML
        parsed_data = chunker.process_and_chunk(content)
        
        # Index in OpenSearch
        await indexer.index_document(s3_path, index_name, document_id, domain)
        
        # Queue RDF processing
        send_process_rdf_graph(document_id)
    
    # 3. Return results
    return {
        "status": "success",
        "document_ids": document_ids,
        "file_count": len(files)
    }
```

### 4. Response

```json
{
    "message": "1 document(s) processed successfully",
    "document_ids": ["doc-uuid-1"],
    "file_count": 1,
    "uploaded_files": 1,
    "domain": "SPACE",
    "format": ".xml",
    "ontology_id": null,
    "ontology_name": null,
    "status": "pending",
    "tags": [],
    "update_enabled": true
}
```

---

## Batch Processing

### Multiple Files

```python
# Upload 5 files at once
files = [
    ("file", ("doc1.xml", content1)),
    ("file", ("doc2.xml", content2)),
    ("file", ("doc3.xml", content3)),
    ("file", ("doc4.xml", content4)),
    ("file", ("doc5.xml", content5))
]

response = client.post(
    "/api/upload",
    files=files,
    data={"domain": "SPACE"}
)

# Returns:
{
    "document_ids": ["doc-1", "doc-2", "doc-3", "doc-4", "doc-5"],
    "file_count": 5
}
```

### Batch Configuration

```python
# app/config.py
BATCH_PROCESSING_SIZE = 10  # Process 10 files per batch
```

---

## ZIP File Handling

### ZIP Upload

```bash
curl -X POST http://localhost:8000/api/upload \
  -F "file=@documents.zip" \
  -F "domain=SPACE"
```

### ZIP Processing

```python
# Extracts all XML files from ZIP
# Creates parent document + child documents
# Returns all document IDs
```

**Example Response**:
```json
{
    "document_ids": [
        "parent-doc-id",
        "child-doc-1",
        "child-doc-2",
        "child-doc-3"
    ],
    "file_count": 4
}
```

---

## Error Handling

### Validation Errors

```python
# File format not supported
{
    "detail": "Unsupported file format: .exe. Allowed formats: .html, .zip, .xml, .docx, .doc, .md, .mdx, .markdown"
}

# File too large
{
    "detail": "File exceeds maximum size of 500MB"
}

# Domain not found
{
    "detail": "Domain 'INVALID' not found"
}
```

### Processing Errors

```python
# Conversion API failure
{
    "status": "conversion_failed",
    "error": "API call failed"
}

# Storage upload failure
{
    "status": "download_failed",
    "error": "Failed to download file from storage"
}

# Indexing failure
{
    "status": "indexing_failed",
    "error": "OpenSearch connection error"
}
```

---

## Monitoring

### Check Document Status

```bash
curl http://localhost:8000/api/documents/{document_id}/status
```

**Response**:
```json
{
    "document_id": "doc-id",
    "filename": "document.xml",
    "status": "completed",
    "title": "Document Title",
    "is_container": false,
    "created_at": "2024-01-15T10:30:00Z",
    "processed_at": "2024-01-15T10:35:00Z"
}
```

### Get Document Details

```bash
curl http://localhost:8000/api/documents/{document_id}
```

**Response**:
```json
{
    "document_id": "doc-id",
    "filename": "document.xml",
    "title": "Document Title",
    "abstract": "Document abstract...",
    "authors": ["Author 1", "Author 2"],
    "keywords": ["keyword1", "keyword2"],
    "domain": "SPACE",
    "status": "completed",
    "file_size": 1024,
    "created_at": "2024-01-15T10:30:00Z",
    "processed_at": "2024-01-15T10:35:00Z"
}
```

---

## Performance Tips

1. **Batch Upload**: Upload multiple files together
2. **Async Processing**: Use async endpoints for large files
3. **Parallel RDF**: RDF processing happens in background
4. **Caching**: Enable Redis for frequently accessed data
5. **Monitoring**: Track processing metrics

---

## Troubleshooting

### Document Stuck in "pending"

**Cause**: SQS worker not running or queue issue

**Solution**:
```bash
# Check SQS queue
aws sqs get-queue-attributes --queue-url $SQS_QUEUE_URL --attribute-names All

# Check worker logs
tail -f logs/app.log | grep worker

# Restart worker
docker restart metr-worker
```

### Indexing Fails

**Cause**: OpenSearch connection or mapping issue

**Solution**:
```bash
# Check OpenSearch
curl -u admin:password https://opensearch:9200/_cluster/health

# Check index
curl -u admin:password https://opensearch:9200/metr-documents

# Recreate index
curl -X DELETE -u admin:password https://opensearch:9200/metr-documents
```

### XML Parsing Fails

**Cause**: Invalid XPath or XML structure

**Solution**:
```python
# Test XPath
from lxml import etree
tree = etree.parse('document.xml')
root = tree.getroot()
result = root.xpath('/topic/title/text()')
print(result)
```

---

## References

- [DocumentProcessor](../metR-Infinity-backend/app/ingestion/processors/document_processor.py)
- [XMLProcessor](../metR-Infinity-backend/app/ingestion/processors/xml_processor.py)
- [ConfigurableXMLChunker](../metR-Infinity-backend/app/ingestion/chunker.py)
- [DocumentIndexer](../metR-Infinity-backend/app/ingestion/processors/document_indexer.py)
- [Configuration Guide](./CONFIGURATION_GUIDE.md)
- [Testing Guide](./TESTING_GUIDE.md)

