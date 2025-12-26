# Data Ingestion Pipeline - Testing Guide

## Overview

This guide provides comprehensive testing strategies for the data ingestion pipeline, covering unit tests, integration tests, and end-to-end testing scenarios.

---

## Test Structure

```
tests/
├── unit/
│   ├── test_file_validation.py
│   ├── test_storage_upload.py
│   ├── test_database_operations.py
│   ├── test_xml_processing.py
│   ├── test_opensearch_indexing.py
│   └── test_rdf_processing.py
├── integration/
│   ├── test_pipeline_flow.py
│   ├── test_api_endpoints.py
│   └── test_error_handling.py
├── e2e/
│   ├── test_complete_pipeline.py
│   └── test_performance.py
├── fixtures/
│   ├── sample_documents/
│   │   ├── valid_xml.xml
│   │   ├── valid_docx.docx
│   │   ├── invalid_format.exe
│   │   └── large_file.xml
│   └── mock_data/
│       ├── domains.json
│       └── metadata.json
└── conftest.py
```

---

## Unit Testing

### 1. File Validation Tests

**File**: `tests/unit/test_file_validation.py`

```python
import pytest
from fastapi import UploadFile
from app.ingestion.processors.document_processor import DocumentProcessor
from app.core.exceptions import UnsupportedDocumentTypeError, DocumentTooLargeError

class TestFileValidation:
    
    @pytest.fixture
    def processor(self):
        return DocumentProcessor()
    
    def test_validate_supported_extensions(self, processor):
        """Test validation of supported file extensions"""
        valid_files = [
            ("document.xml", "application/xml"),
            ("document.docx", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"),
            ("document.html", "text/html"),
            ("archive.zip", "application/zip"),
            ("readme.md", "text/markdown")
        ]
        
        for filename, content_type in valid_files:
            file = UploadFile(filename=filename, content_type=content_type)
            result = processor.validate_file(file)
            assert result["file_type"] in processor.ALLOWED_EXTENSIONS
    
    def test_validate_unsupported_extensions(self, processor):
        """Test rejection of unsupported file extensions"""
        invalid_files = [
            ("malware.exe", "application/x-executable"),
            ("image.jpg", "image/jpeg"),
            ("video.mp4", "video/mp4")
        ]
        
        for filename, content_type in invalid_files:
            file = UploadFile(filename=filename, content_type=content_type)
            with pytest.raises(UnsupportedDocumentTypeError):
                processor.validate_file(file)
    
    @pytest.mark.asyncio
    async def test_file_size_validation(self, processor):
        """Test file size validation"""
        # Valid size file
        small_content = b"<xml>small content</xml>"
        small_file = UploadFile(filename="small.xml")
        small_file.file = io.BytesIO(small_content)
        
        content = await processor.validate_file_size(small_file)
        assert content == small_content
        
        # Oversized file
        large_content = b"x" * (500 * 1024 * 1024 + 1)  # 500MB + 1 byte
        large_file = UploadFile(filename="large.xml")
        large_file.file = io.BytesIO(large_content)
        
        with pytest.raises(DocumentTooLargeError):
            await processor.validate_file_size(large_file)
    
    def test_domain_validation(self, processor, mock_db_session):
        """Test domain validation"""
        # Valid domain
        result = processor.validate_file(
            UploadFile(filename="doc.xml"), 
            domain="SPACE"
        )
        assert result["domain"] == "SPACE"
        
        # Invalid domain
        with pytest.raises(HTTPException) as exc_info:
            processor.validate_file(
                UploadFile(filename="doc.xml"), 
                domain="INVALID"
            )
        assert exc_info.value.status_code == 400
```

### 2. Storage Upload Tests

**File**: `tests/unit/test_storage_upload.py`

```python
import pytest
from unittest.mock import Mock, patch
from app.ingestion.processors.document_processor import DocumentProcessor

class TestStorageUpload:
    
    @pytest.fixture
    def processor(self):
        return DocumentProcessor()
    
    @pytest.fixture
    def sample_file_data(self):
        return {
            "document_id": "test-doc-123",
            "filename": "test.xml",
            "content": b"<xml>test content</xml>",
            "size": 25,
            "content_type": "application/xml",
            "domain": "SPACE"
        }
    
    def test_classify_file(self, processor, sample_file_data):
        """Test file classification"""
        file = UploadFile(filename="test.xml")
        content = b"<xml>test content</xml>"
        
        result = processor.classify_file(file, content, domain="SPACE")
        
        assert "document_id" in result
        assert result["filename"] == "test.xml"
        assert result["content"] == content
        assert result["domain"] == "SPACE"
        assert result["file_extension"] == ".xml"
    
    @patch('app.ingestion.processors.document_processor.s3_client')
    def test_save_to_storage_success(self, mock_s3, processor, sample_file_data):
        """Test successful file upload to S3"""
        mock_s3.put_object.return_value = {"ETag": "test-etag"}
        
        prefix = "uploads/SPACE/test-doc-123/"
        storage_key = processor.save_to_storage(sample_file_data, prefix)
        
        expected_key = f"{prefix}test.xml"
        assert storage_key == expected_key
        
        mock_s3.put_object.assert_called_once_with(
            Bucket=processor.s3_bucket,
            Key=expected_key,
            Body=sample_file_data["content"],
            ContentType=sample_file_data["content_type"]
        )
    
    @patch('app.ingestion.processors.document_processor.s3_client')
    def test_save_to_storage_failure(self, mock_s3, processor, sample_file_data):
        """Test S3 upload failure handling"""
        mock_s3.put_object.side_effect = Exception("S3 upload failed")
        
        with pytest.raises(Exception) as exc_info:
            processor.save_to_storage(sample_file_data, "uploads/test/")
        
        assert "S3 upload failed" in str(exc_info.value)
```

### 3. Database Operations Tests

**File**: `tests/unit/test_database_operations.py`

```python
import pytest
from datetime import datetime
from app.ingestion.processors.document_processor import DocumentProcessor
from app.models.document import Document

class TestDatabaseOperations:
    
    @pytest.fixture
    def processor(self):
        return DocumentProcessor()
    
    @pytest.fixture
    def sample_document_data(self):
        return {
            "document_id": "test-doc-123",
            "filename": "test.xml",
            "s3_path": "uploads/SPACE/test-doc-123/test.xml",
            "file_size": 1024,
            "content_type": "application/xml",
            "domain": "SPACE",
            "file_metadata": {"author": "Test Author"},
            "parsed_metadata": {"title": "Test Document"}
        }
    
    def test_create_document_record(self, processor, sample_document_data, mock_db_session):
        """Test creating new document record"""
        result = processor.create_or_update_document_record(sample_document_data)
        
        assert result["action"] == "created"
        assert result["document_id"] == sample_document_data["document_id"]
        
        # Verify document was created in database
        doc = mock_db_session.query(Document).filter_by(
            document_id=sample_document_data["document_id"]
        ).first()
        
        assert doc is not None
        assert doc.filename == sample_document_data["filename"]
        assert doc.status == "pending"
    
    def test_update_existing_document(self, processor, sample_document_data, mock_db_session):
        """Test updating existing document record"""
        # Create initial document
        existing_doc = Document(
            document_id=sample_document_data["document_id"],
            filename="old_name.xml",
            status="pending"
        )
        mock_db_session.add(existing_doc)
        mock_db_session.commit()
        
        # Update document
        sample_document_data["filename"] = "new_name.xml"
        result = processor.create_or_update_document_record(sample_document_data)
        
        assert result["action"] == "updated"
        
        # Verify update
        updated_doc = mock_db_session.query(Document).filter_by(
            document_id=sample_document_data["document_id"]
        ).first()
        
        assert updated_doc.filename == "new_name.xml"
```

---

## Integration Testing

### 1. Pipeline Flow Tests

**File**: `tests/integration/test_pipeline_flow.py`

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

class TestPipelineFlow:
    
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    @pytest.fixture
    def valid_xml_file(self):
        return {
            "file": ("test.xml", b"<xml><title>Test Document</title></xml>", "application/xml")
        }
    
    def test_complete_upload_flow(self, client, valid_xml_file, mock_services):
        """Test complete document upload and processing flow"""
        # Step 1: Upload document
        response = client.post(
            "/api/v1/documents/upload",
            files=valid_xml_file,
            data={"domain": "SPACE", "format": "xml"}
        )
        
        assert response.status_code == 200
        result = response.json()
        document_id = result["document_id"]
        
        # Step 2: Verify document status
        status_response = client.get(f"/api/v1/documents/{document_id}/status")
        assert status_response.status_code == 200
        assert status_response.json()["status"] in ["pending", "processing"]
        
        # Step 3: Wait for processing completion (mock)
        mock_services.complete_processing(document_id)
        
        # Step 4: Verify final status
        final_status = client.get(f"/api/v1/documents/{document_id}/status")
        assert final_status.json()["status"] == "completed"
    
    def test_error_recovery_flow(self, client, mock_services):
        """Test error handling and recovery in pipeline"""
        # Simulate processing failure
        invalid_file = {
            "file": ("corrupt.xml", b"<xml><unclosed>", "application/xml")
        }
        
        response = client.post(
            "/api/v1/documents/upload",
            files=invalid_file,
            data={"domain": "SPACE"}
        )
        
        # Should still accept upload
        assert response.status_code == 200
        document_id = response.json()["document_id"]
        
        # Simulate processing failure
        mock_services.fail_processing(document_id, "XML parsing error")
        
        # Verify error status
        status_response = client.get(f"/api/v1/documents/{document_id}/status")
        status_data = status_response.json()
        
        assert status_data["status"] == "failed"
        assert "XML parsing error" in status_data["error_message"]
```

### 2. API Endpoint Tests

**File**: `tests/integration/test_api_endpoints.py`

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

class TestAPIEndpoints:
    
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    def test_upload_endpoint_validation(self, client):
        """Test upload endpoint input validation"""
        # Missing file
        response = client.post("/api/v1/documents/upload")
        assert response.status_code == 422
        
        # Invalid domain
        response = client.post(
            "/api/v1/documents/upload",
            files={"file": ("test.xml", b"<xml></xml>", "application/xml")},
            data={"domain": "INVALID"}
        )
        assert response.status_code == 400
        
        # Unsupported format
        response = client.post(
            "/api/v1/documents/upload",
            files={"file": ("test.exe", b"binary", "application/x-executable")},
            data={"domain": "SPACE"}
        )
        assert response.status_code == 400
    
    def test_status_endpoint(self, client, mock_document):
        """Test document status endpoint"""
        document_id = mock_document["document_id"]
        
        response = client.get(f"/api/v1/documents/{document_id}/status")
        assert response.status_code == 200
        
        data = response.json()
        assert "status" in data
        assert "created_at" in data
        assert "updated_at" in data
    
    def test_metadata_endpoint(self, client, mock_document):
        """Test document metadata endpoint"""
        document_id = mock_document["document_id"]
        
        response = client.get(f"/api/v1/documents/{document_id}/metadata")
        assert response.status_code == 200
        
        data = response.json()
        assert "file_metadata" in data
        assert "parsed_metadata" in data
```

---

## End-to-End Testing

### 1. Complete Pipeline Tests

**File**: `tests/e2e/test_complete_pipeline.py`

```python
import pytest
import time
from fastapi.testclient import TestClient
from app.main import app

class TestCompletePipeline:
    
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    @pytest.mark.e2e
    def test_xml_document_pipeline(self, client, real_services):
        """Test complete XML document processing pipeline"""
        # Prepare test document
        xml_content = """<?xml version="1.0"?>
        <document>
            <metadata>
                <title>Space Mission Report</title>
                <author>NASA</author>
                <date>2024-01-15</date>
            </metadata>
            <content>
                <section id="1">
                    <heading>Mission Overview</heading>
                    <text>This report details the space mission...</text>
                </section>
            </content>
        </document>"""
        
        # Upload document
        response = client.post(
            "/api/v1/documents/upload",
            files={"file": ("mission_report.xml", xml_content.encode(), "application/xml")},
            data={"domain": "SPACE", "format": "xml"}
        )
        
        assert response.status_code == 200
        document_id = response.json()["document_id"]
        
        # Wait for processing completion
        max_wait = 60  # seconds
        start_time = time.time()
        
        while time.time() - start_time < max_wait:
            status_response = client.get(f"/api/v1/documents/{document_id}/status")
            status = status_response.json()["status"]
            
            if status == "completed":
                break
            elif status == "failed":
                pytest.fail(f"Document processing failed: {status_response.json()}")
            
            time.sleep(2)
        else:
            pytest.fail("Document processing timed out")
        
        # Verify all pipeline stages completed
        self._verify_storage_upload(document_id)
        self._verify_database_record(document_id)
        self._verify_opensearch_indexing(document_id)
        self._verify_rdf_processing(document_id)
    
    def _verify_storage_upload(self, document_id):
        """Verify file was uploaded to S3"""
        # Check S3 storage
        pass
    
    def _verify_database_record(self, document_id):
        """Verify database record was created"""
        # Check database
        pass
    
    def _verify_opensearch_indexing(self, document_id):
        """Verify document was indexed in OpenSearch"""
        # Check OpenSearch index
        pass
    
    def _verify_rdf_processing(self, document_id):
        """Verify RDF processing completed"""
        # Check RDF graph
        pass
```

### 2. Performance Tests

**File**: `tests/e2e/test_performance.py`

```python
import pytest
import time
import concurrent.futures
from fastapi.testclient import TestClient
from app.main import app

class TestPerformance:
    
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    @pytest.mark.performance
    def test_concurrent_uploads(self, client):
        """Test system performance under concurrent uploads"""
        def upload_document(doc_num):
            xml_content = f"<xml><id>{doc_num}</id><content>Test content {doc_num}</content></xml>"
            
            start_time = time.time()
            response = client.post(
                "/api/v1/documents/upload",
                files={"file": (f"test_{doc_num}.xml", xml_content.encode(), "application/xml")},
                data={"domain": "SPACE"}
            )
            end_time = time.time()
            
            return {
                "doc_num": doc_num,
                "status_code": response.status_code,
                "response_time": end_time - start_time,
                "document_id": response.json().get("document_id") if response.status_code == 200 else None
            }
        
        # Test with 10 concurrent uploads
        num_uploads = 10
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            futures = [executor.submit(upload_document, i) for i in range(num_uploads)]
            results = [future.result() for future in concurrent.futures.as_completed(futures)]
        
        # Verify all uploads succeeded
        successful_uploads = [r for r in results if r["status_code"] == 200]
        assert len(successful_uploads) == num_uploads
        
        # Check response times
        avg_response_time = sum(r["response_time"] for r in results) / len(results)
        assert avg_response_time < 5.0  # Should complete within 5 seconds
    
    @pytest.mark.performance
    def test_large_file_processing(self, client):
        """Test processing of large files"""
        # Generate large XML content (10MB)
        large_content = "<xml>" + "<data>" + "x" * (10 * 1024 * 1024 - 20) + "</data>" + "</xml>"
        
        start_time = time.time()
        response = client.post(
            "/api/v1/documents/upload",
            files={"file": ("large_file.xml", large_content.encode(), "application/xml")},
            data={"domain": "SPACE"}
        )
        upload_time = time.time() - start_time
        
        assert response.status_code == 200
        assert upload_time < 30.0  # Should upload within 30 seconds
        
        document_id = response.json()["document_id"]
        
        # Monitor processing time
        processing_start = time.time()
        max_processing_time = 300  # 5 minutes
        
        while time.time() - processing_start < max_processing_time:
            status_response = client.get(f"/api/v1/documents/{document_id}/status")
            status = status_response.json()["status"]
            
            if status in ["completed", "failed"]:
                break
            
            time.sleep(5)
        
        processing_time = time.time() - processing_start
        assert processing_time < max_processing_time
```

---

## Test Configuration

### Fixtures and Mocks

**File**: `tests/conftest.py`

```python
import pytest
from unittest.mock import Mock, patch
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.database import Base
from app.models.document import Document
from app.models.domain import Domain

@pytest.fixture(scope="session")
def test_db():
    """Create test database"""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture
def mock_db_session(test_db):
    """Create mock database session"""
    Session = sessionmaker(bind=test_db)
    session = Session()
    
    # Add test domains
    domains = [
        Domain(name="SPACE", description="Space domain"),
        Domain(name="TIME", description="Time domain"),
        Domain(name="POWER", description="Power domain")
    ]
    session.add_all(domains)
    session.commit()
    
    yield session
    session.close()

@pytest.fixture
def mock_s3_client():
    """Mock S3 client"""
    with patch('app.ingestion.processors.document_processor.s3_client') as mock:
        mock.put_object.return_value = {"ETag": "test-etag"}
        mock.get_object.return_value = {"Body": Mock()}
        yield mock

@pytest.fixture
def mock_opensearch_client():
    """Mock OpenSearch client"""
    with patch('app.ingestion.processors.opensearch_processor.opensearch_client') as mock:
        mock.index.return_value = {"_id": "test-id", "result": "created"}
        mock.search.return_value = {"hits": {"hits": []}}
        yield mock

@pytest.fixture
def mock_sqs_client():
    """Mock SQS client"""
    with patch('app.ingestion.processors.rdf_processor.sqs_client') as mock:
        mock.send_message.return_value = {"MessageId": "test-message-id"}
        yield mock

@pytest.fixture
def mock_services(mock_s3_client, mock_opensearch_client, mock_sqs_client):
    """Combined mock services"""
    return {
        "s3": mock_s3_client,
        "opensearch": mock_opensearch_client,
        "sqs": mock_sqs_client
    }
```

---

## Running Tests

### Test Commands

```bash
# Run all tests
pytest

# Run specific test categories
pytest tests/unit/                    # Unit tests only
pytest tests/integration/             # Integration tests only
pytest tests/e2e/                     # End-to-end tests only

# Run with coverage
pytest --cov=app --cov-report=html

# Run performance tests
pytest -m performance

# Run with verbose output
pytest -v

# Run specific test file
pytest tests/unit/test_file_validation.py

# Run specific test method
pytest tests/unit/test_file_validation.py::TestFileValidation::test_validate_supported_extensions
```

### Test Environment Setup

```bash
# Install test dependencies
pip install pytest pytest-asyncio pytest-cov

# Set test environment variables
export TESTING=true
export DATABASE_URL=sqlite:///:memory:
export S3_BUCKET=test-bucket
export OPENSEARCH_HOST=localhost:9200

# Run tests
pytest
```

### CI/CD Integration

**File**: `.github/workflows/test.yml`

```yaml
name: Test Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.9
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-asyncio pytest-cov
    
    - name: Run unit tests
      run: pytest tests/unit/ --cov=app
    
    - name: Run integration tests
      run: pytest tests/integration/
    
    - name: Upload coverage
      uses: codecov/codecov-action@v1
```

---

## Test Data Management

### Sample Test Files

Create test files in `tests/fixtures/sample_documents/`:

1. **valid_xml.xml** - Valid XML document
2. **valid_docx.docx** - Valid DOCX document  
3. **invalid_format.exe** - Invalid file format
4. **large_file.xml** - File exceeding size limit
5. **corrupt_xml.xml** - Malformed XML

### Mock Data

**File**: `tests/fixtures/mock_data/domains.json`

```json
[
  {"name": "SPACE", "description": "Space domain"},
  {"name": "TIME", "description": "Time domain"},
  {"name": "POWER", "description": "Power domain"}
]
```

This testing guide provides comprehensive coverage for validating the data ingestion pipeline functionality, performance, and reliability.