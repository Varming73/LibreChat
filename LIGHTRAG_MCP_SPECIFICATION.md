# LightRAG MCP Server - Technical Specification

## Document Overview

This document provides complete technical specifications for building a Model Context Protocol (MCP) server that integrates LightRAG with LibreChat. This server will enable users to upload and index PDF and Markdown files into LightRAG through LibreChat's chat interface.

**Target Audience:** Developers with no prior knowledge of LibreChat's codebase
**Protocol:** MCP (Model Context Protocol) over stdio
**Primary Use Case:** Document indexing for PDF and Markdown files

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Functional Requirements](#functional-requirements)
3. [Non-Functional Requirements](#non-functional-requirements)
4. [MCP Protocol Basics](#mcp-protocol-basics)
5. [Data Flow](#data-flow)
6. [Tool Schema Design](#tool-schema-design)
7. [Implementation Guide](#implementation-guide)
8. [Testing Strategy](#testing-strategy)
9. [Error Handling](#error-handling)
10. [Security Considerations](#security-considerations)

---

## Architecture Overview

### System Context

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│             │         │              │         │             │
│  LibreChat  │◄───────►│  MCP Server  │◄───────►│  LightRAG   │
│  (Frontend) │  stdio  │  (Your Code) │   API   │  (Backend)  │
│             │  JSON   │              │         │             │
└─────────────┘  RPC    └──────────────┘         └─────────────┘
```

### Communication Protocol

- **Transport:** stdio (standard input/output)
- **Format:** JSON-RPC 2.0
- **Encoding:** UTF-8
- **File Transfer:** Base64 encoding within JSON

### Key Concept: How LibreChat Calls Your MCP Server

LibreChat spawns your MCP server as a **child process** and communicates via:
- **stdin:** LibreChat sends JSON-RPC requests
- **stdout:** Your server sends JSON-RPC responses
- **stderr:** Your server can log errors (optional)

**Critical:** Only write valid JSON-RPC to stdout. All debug logs must go to stderr or a file.

---

## Functional Requirements

### FR-1: File Upload Tool

**Requirement:** Provide an MCP tool that accepts document files and indexes them in LightRAG.

**Acceptance Criteria:**
- Tool accepts PDF files (binary content as base64)
- Tool accepts Markdown files (text content as base64)
- Tool extracts text content from PDFs
- Tool indexes content in LightRAG
- Tool returns success/failure status with meaningful messages

### FR-2: Tool Discovery

**Requirement:** Implement MCP's `tools/list` method to advertise available tools.

**Acceptance Criteria:**
- Returns list of available tools with proper JSON Schema
- Schema accurately describes input parameters
- Tool descriptions are clear and actionable for AI agents

### FR-3: Document Metadata Handling

**Requirement:** Preserve and utilize document metadata during indexing.

**Acceptance Criteria:**
- Extract filename from input parameters
- Store original filename in LightRAG
- Associate MIME type with indexed content
- Track upload timestamp

### FR-4: Query Tool (Optional but Recommended)

**Requirement:** Provide a tool to query indexed documents.

**Acceptance Criteria:**
- Accept natural language queries
- Return relevant document excerpts
- Include source document references

---

## Non-Functional Requirements

### NFR-1: Performance

- **File Size Support:** Handle files up to 10MB efficiently
- **Processing Time:** Index documents within 30 seconds for typical PDFs
- **Memory Usage:** Keep memory footprint under 512MB for concurrent operations
- **Startup Time:** Initialize MCP server within 5 seconds

### NFR-2: Reliability

- **Error Recovery:** Gracefully handle malformed PDFs and corrupted files
- **Connection Stability:** Maintain stdio connection without crashes
- **Idempotency:** Support re-indexing same document without duplication issues
- **Graceful Degradation:** Continue operation if LightRAG backend is temporarily unavailable

### NFR-3: Maintainability

- **Logging:** Comprehensive logging to stderr or file (not stdout)
- **Configuration:** Externalize LightRAG connection settings
- **Code Quality:** Follow Python best practices (PEP 8)
- **Documentation:** Inline code comments for complex logic

### NFR-4: Security

- **Input Validation:** Validate all base64 inputs before decoding
- **File Type Verification:** Verify MIME types match file content
- **Size Limits:** Reject files exceeding size limits
- **Path Traversal Prevention:** Sanitize filenames to prevent directory traversal attacks

---

## MCP Protocol Basics

### What You Need to Know

The MCP SDK handles most protocol complexity, but you need to understand:

1. **JSON-RPC 2.0:** Request/response format over stdio
2. **Tool Registration:** Declare tools with JSON Schema
3. **Tool Execution:** Receive arguments, process, return results

### Standard MCP Methods You Must Implement

#### 1. `initialize` (handled by SDK)

LibreChat sends:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "LibreChat",
      "version": "0.7.0"
    }
  }
}
```

Your server responds (SDK handles this):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {}
    },
    "serverInfo": {
      "name": "lightrag-mcp",
      "version": "1.0.0"
    }
  }
}
```

#### 2. `tools/list` (you implement)

LibreChat requests:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}
```

Your server responds:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "upload_to_lightrag",
        "description": "Upload and index a PDF or Markdown document into LightRAG knowledge base",
        "inputSchema": {
          "type": "object",
          "properties": {
            "filename": {
              "type": "string",
              "description": "The name of the file (e.g., 'document.pdf')"
            },
            "content": {
              "type": "string",
              "description": "Base64-encoded file content"
            },
            "mimeType": {
              "type": "string",
              "enum": ["application/pdf", "text/markdown"],
              "description": "MIME type of the file"
            }
          },
          "required": ["filename", "content", "mimeType"]
        }
      }
    ]
  }
}
```

#### 3. `tools/call` (you implement)

LibreChat calls your tool:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "upload_to_lightrag",
    "arguments": {
      "filename": "research-paper.pdf",
      "content": "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8P...",
      "mimeType": "application/pdf"
    }
  }
}
```

Your server responds:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Successfully indexed 'research-paper.pdf' into LightRAG. The document contains 2,456 words and has been processed into 12 semantic chunks."
      }
    ]
  }
}
```

---

## Data Flow

### Upload Sequence Diagram

```
User                LibreChat           MCP Server         LightRAG
 |                      |                    |                 |
 |  Attach PDF          |                    |                 |
 |--------------------->|                    |                 |
 |                      |                    |                 |
 |  "Index this doc"    |                    |                 |
 |--------------------->|                    |                 |
 |                      |                    |                 |
 |                      | Read file          |                 |
 |                      | Encode to base64   |                 |
 |                      |                    |                 |
 |                      | tools/call         |                 |
 |                      |------------------->|                 |
 |                      |                    |                 |
 |                      |                    | Decode base64   |
 |                      |                    | Validate file   |
 |                      |                    |                 |
 |                      |                    | Index document  |
 |                      |                    |---------------->|
 |                      |                    |                 |
 |                      |                    |    Success      |
 |                      |                    |<----------------|
 |                      |                    |                 |
 |                      |   Tool result      |                 |
 |                      |<-------------------|                 |
 |                      |                    |                 |
 |  Success message     |                    |                 |
 |<---------------------|                    |                 |
```

### Important: How Files Reach Your MCP Server

**Current State:** LibreChat does NOT automatically pass attached files to MCP tools.

**What This Means for Users:**

Users must currently use one of these workflows:

**Option A: Manual Base64 (Power Users)**
1. User attaches file in LibreChat
2. User provides file_id or path in their message
3. **Someone/something** (likely a LibreChat plugin or the AI agent) reads the file
4. The file content is base64-encoded
5. AI agent calls your tool with base64 content

**Option B: Direct Base64 Input (Copy/Paste)**
1. User manually base64-encodes file outside LibreChat
2. User pastes base64 string in chat
3. User asks AI to call upload tool with the base64 content

**Option C: Future Enhancement (Not Available Yet)**
LibreChat could be enhanced to automatically include attached files in tool parameters, but this requires LibreChat code changes.

**Recommendation for Your Implementation:**

Design your tool to accept base64 content directly. This is the most flexible approach and works regardless of how LibreChat evolves. Your tool schema should look like this:

```json
{
  "name": "upload_to_lightrag",
  "inputSchema": {
    "properties": {
      "content": {
        "type": "string",
        "description": "Base64-encoded PDF or Markdown file content"
      }
    }
  }
}
```

The AI agent in LibreChat will need to be instructed (via system prompt or user instruction) to read attached files and pass their base64-encoded content to your tool.

---

## Tool Schema Design

### Primary Tool: `upload_to_lightrag`

```json
{
  "name": "upload_to_lightrag",
  "description": "Upload and index a PDF or Markdown document into the LightRAG knowledge base for semantic search and retrieval. Use this when the user wants to add documents to their knowledge base.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "filename": {
        "type": "string",
        "description": "Original filename including extension (e.g., 'report.pdf', 'notes.md')",
        "pattern": "^[\\w\\-. ]+\\.(pdf|md)$"
      },
      "content": {
        "type": "string",
        "description": "Base64-encoded file content. For PDFs, this is the binary PDF data encoded as base64. For Markdown, this is the UTF-8 text encoded as base64.",
        "contentEncoding": "base64"
      },
      "mimeType": {
        "type": "string",
        "enum": ["application/pdf", "text/markdown", "text/x-markdown"],
        "description": "MIME type of the document"
      },
      "metadata": {
        "type": "object",
        "description": "Optional metadata to associate with the document",
        "properties": {
          "tags": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Tags to categorize the document"
          },
          "author": {
            "type": "string",
            "description": "Document author"
          },
          "source": {
            "type": "string",
            "description": "Source or origin of the document"
          }
        },
        "additionalProperties": true
      }
    },
    "required": ["filename", "content", "mimeType"],
    "additionalProperties": false
  }
}
```

### Secondary Tool: `query_lightrag` (Recommended)

```json
{
  "name": "query_lightrag",
  "description": "Query the LightRAG knowledge base with natural language. Returns relevant document excerpts and sources.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Natural language question or search query",
        "minLength": 3,
        "maxLength": 500
      },
      "max_results": {
        "type": "integer",
        "description": "Maximum number of results to return",
        "default": 5,
        "minimum": 1,
        "maximum": 20
      },
      "include_sources": {
        "type": "boolean",
        "description": "Include source document references in results",
        "default": true
      }
    },
    "required": ["query"],
    "additionalProperties": false
  }
}
```

---

## Implementation Guide

### Technology Stack

**Required:**
- Python 3.10+
- `mcp` package (MCP SDK for Python)
- `PyPDF2` or `pdfplumber` (PDF text extraction)
- LightRAG client library (your existing LightRAG integration)

**Recommended:**
- `pydantic` (data validation)
- `python-magic` (MIME type verification)

### Project Structure

```
lightrag-mcp-server/
├── pyproject.toml           # Python project config
├── README.md                # Setup instructions
├── .env.example             # Environment variable template
├── src/
│   └── lightrag_mcp/
│       ├── __init__.py
│       ├── server.py        # Main MCP server
│       ├── tools.py         # Tool implementations
│       ├── lightrag_client.py  # LightRAG API client
│       ├── pdf_processor.py    # PDF text extraction
│       └── config.py        # Configuration management
├── tests/
│   ├── test_server.py
│   ├── test_tools.py
│   └── fixtures/
│       ├── sample.pdf
│       └── sample.md
└── logs/                    # Log output directory
```

### Core Implementation: `server.py`

```python
#!/usr/bin/env python3
"""
LightRAG MCP Server
Provides document upload and query tools for LightRAG integration with LibreChat.
"""

import asyncio
import logging
import sys
from typing import Any

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

from .tools import upload_to_lightrag_handler, query_lightrag_handler
from .config import load_config

# Configure logging to stderr (NOT stdout - stdout is reserved for JSON-RPC)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    stream=sys.stderr  # Critical: use stderr for logs
)
logger = logging.getLogger(__name__)


# Initialize MCP server
app = Server("lightrag-mcp")


@app.list_tools()
async def list_tools() -> list[Tool]:
    """
    Return list of available tools.
    Called by LibreChat during tool discovery.
    """
    return [
        Tool(
            name="upload_to_lightrag",
            description=(
                "Upload and index a PDF or Markdown document into the LightRAG "
                "knowledge base for semantic search and retrieval. Use this when "
                "the user wants to add documents to their knowledge base."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "filename": {
                        "type": "string",
                        "description": "Original filename including extension (e.g., 'report.pdf', 'notes.md')",
                    },
                    "content": {
                        "type": "string",
                        "description": (
                            "Base64-encoded file content. For PDFs, this is the binary "
                            "PDF data encoded as base64. For Markdown, this is the UTF-8 "
                            "text encoded as base64."
                        ),
                    },
                    "mimeType": {
                        "type": "string",
                        "enum": ["application/pdf", "text/markdown", "text/x-markdown"],
                        "description": "MIME type of the document",
                    },
                    "metadata": {
                        "type": "object",
                        "description": "Optional metadata to associate with the document",
                        "properties": {
                            "tags": {
                                "type": "array",
                                "items": {"type": "string"},
                                "description": "Tags to categorize the document",
                            },
                            "author": {
                                "type": "string",
                                "description": "Document author",
                            },
                            "source": {
                                "type": "string",
                                "description": "Source or origin of the document",
                            },
                        },
                        "additionalProperties": True,
                    },
                },
                "required": ["filename", "content", "mimeType"],
            },
        ),
        Tool(
            name="query_lightrag",
            description=(
                "Query the LightRAG knowledge base with natural language. "
                "Returns relevant document excerpts and sources."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Natural language question or search query",
                    },
                    "max_results": {
                        "type": "integer",
                        "description": "Maximum number of results to return",
                        "default": 5,
                        "minimum": 1,
                        "maximum": 20,
                    },
                    "include_sources": {
                        "type": "boolean",
                        "description": "Include source document references in results",
                        "default": True,
                    },
                },
                "required": ["query"],
            },
        ),
    ]


@app.call_tool()
async def call_tool(name: str, arguments: Any) -> list[TextContent]:
    """
    Handle tool execution requests from LibreChat.

    Args:
        name: Tool name (e.g., "upload_to_lightrag")
        arguments: Dictionary of tool arguments matching inputSchema

    Returns:
        List of TextContent objects with results
    """
    logger.info(f"Tool called: {name}")
    logger.debug(f"Arguments: {arguments}")

    try:
        if name == "upload_to_lightrag":
            result = await upload_to_lightrag_handler(arguments)
            return [TextContent(type="text", text=result)]

        elif name == "query_lightrag":
            result = await query_lightrag_handler(arguments)
            return [TextContent(type="text", text=result)]

        else:
            error_msg = f"Unknown tool: {name}"
            logger.error(error_msg)
            return [TextContent(type="text", text=f"Error: {error_msg}")]

    except Exception as e:
        error_msg = f"Error executing {name}: {str(e)}"
        logger.exception(error_msg)
        return [TextContent(type="text", text=f"Error: {error_msg}")]


async def main():
    """Main entry point - run the MCP server over stdio."""
    logger.info("Starting LightRAG MCP Server...")

    # Load configuration
    config = load_config()
    logger.info(f"Loaded config: LightRAG endpoint = {config.lightrag_endpoint}")

    # Run the server using stdio transport
    async with stdio_server() as (read_stream, write_stream):
        logger.info("MCP Server ready and listening on stdio")
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Server stopped by user")
    except Exception as e:
        logger.exception(f"Fatal error: {e}")
        sys.exit(1)
```

### Tool Handlers: `tools.py`

```python
"""
Tool handler implementations for LightRAG MCP server.
"""

import base64
import logging
from typing import Dict, Any, Optional
from datetime import datetime

from .pdf_processor import extract_text_from_pdf
from .lightrag_client import LightRAGClient
from .config import get_config

logger = logging.getLogger(__name__)


async def upload_to_lightrag_handler(arguments: Dict[str, Any]) -> str:
    """
    Handle document upload to LightRAG.

    Args:
        arguments: Dictionary with keys:
            - filename: str
            - content: str (base64-encoded)
            - mimeType: str
            - metadata: Dict (optional)

    Returns:
        Success message with indexing details

    Raises:
        ValueError: If arguments are invalid
        RuntimeError: If LightRAG indexing fails
    """
    # Extract and validate arguments
    filename = arguments.get("filename")
    content_b64 = arguments.get("content")
    mime_type = arguments.get("mimeType")
    metadata = arguments.get("metadata", {})

    if not filename or not content_b64 or not mime_type:
        raise ValueError("Missing required arguments: filename, content, or mimeType")

    logger.info(f"Processing upload: {filename} ({mime_type})")

    # Decode base64 content
    try:
        content_bytes = base64.b64decode(content_b64)
        logger.info(f"Decoded {len(content_bytes)} bytes")
    except Exception as e:
        raise ValueError(f"Invalid base64 content: {e}")

    # Validate file size (10MB limit)
    max_size = 10 * 1024 * 1024  # 10MB
    if len(content_bytes) > max_size:
        raise ValueError(f"File too large: {len(content_bytes)} bytes (max {max_size})")

    # Extract text based on MIME type
    try:
        if mime_type == "application/pdf":
            text_content = extract_text_from_pdf(content_bytes)
        elif mime_type in ["text/markdown", "text/x-markdown"]:
            text_content = content_bytes.decode("utf-8")
        else:
            raise ValueError(f"Unsupported MIME type: {mime_type}")

        word_count = len(text_content.split())
        logger.info(f"Extracted {word_count} words from {filename}")

    except Exception as e:
        raise RuntimeError(f"Failed to extract text: {e}")

    # Prepare document metadata
    doc_metadata = {
        "filename": filename,
        "mime_type": mime_type,
        "upload_time": datetime.utcnow().isoformat(),
        "word_count": word_count,
        **metadata  # Merge user-provided metadata
    }

    # Index in LightRAG
    config = get_config()
    client = LightRAGClient(config.lightrag_endpoint, config.lightrag_api_key)

    try:
        result = await client.index_document(
            content=text_content,
            metadata=doc_metadata
        )

        chunk_count = result.get("chunk_count", "unknown")

        success_msg = (
            f"Successfully indexed '{filename}' into LightRAG. "
            f"The document contains {word_count} words and has been "
            f"processed into {chunk_count} semantic chunks."
        )

        logger.info(success_msg)
        return success_msg

    except Exception as e:
        error_msg = f"LightRAG indexing failed: {e}"
        logger.error(error_msg)
        raise RuntimeError(error_msg)


async def query_lightrag_handler(arguments: Dict[str, Any]) -> str:
    """
    Handle query to LightRAG knowledge base.

    Args:
        arguments: Dictionary with keys:
            - query: str
            - max_results: int (optional)
            - include_sources: bool (optional)

    Returns:
        Query results formatted as text
    """
    query = arguments.get("query")
    max_results = arguments.get("max_results", 5)
    include_sources = arguments.get("include_sources", True)

    if not query:
        raise ValueError("Missing required argument: query")

    logger.info(f"Querying LightRAG: '{query}' (max_results={max_results})")

    config = get_config()
    client = LightRAGClient(config.lightrag_endpoint, config.lightrag_api_key)

    try:
        results = await client.query(
            query=query,
            max_results=max_results
        )

        # Format results
        if not results:
            return "No relevant documents found for your query."

        response_parts = [f"Found {len(results)} relevant results:\n"]

        for i, result in enumerate(results, 1):
            response_parts.append(f"\n**Result {i}:**")
            response_parts.append(result.get("text", ""))

            if include_sources and "metadata" in result:
                metadata = result["metadata"]
                source = metadata.get("filename", "Unknown source")
                response_parts.append(f"\n*Source: {source}*")

        return "\n".join(response_parts)

    except Exception as e:
        error_msg = f"LightRAG query failed: {e}"
        logger.error(error_msg)
        raise RuntimeError(error_msg)
```

### PDF Processing: `pdf_processor.py`

```python
"""
PDF text extraction utilities.
"""

import io
import logging
from typing import Union

try:
    import pdfplumber
    PDF_LIBRARY = "pdfplumber"
except ImportError:
    try:
        import PyPDF2
        PDF_LIBRARY = "PyPDF2"
    except ImportError:
        raise ImportError("No PDF library found. Install pdfplumber or PyPDF2")

logger = logging.getLogger(__name__)


def extract_text_from_pdf(pdf_bytes: bytes) -> str:
    """
    Extract text content from PDF bytes.

    Args:
        pdf_bytes: Raw PDF file content

    Returns:
        Extracted text as string

    Raises:
        ValueError: If PDF is invalid or empty
        RuntimeError: If extraction fails
    """
    if not pdf_bytes:
        raise ValueError("PDF content is empty")

    logger.info(f"Extracting text from PDF using {PDF_LIBRARY}")

    try:
        if PDF_LIBRARY == "pdfplumber":
            return _extract_with_pdfplumber(pdf_bytes)
        else:
            return _extract_with_pypdf2(pdf_bytes)
    except Exception as e:
        raise RuntimeError(f"PDF extraction failed: {e}")


def _extract_with_pdfplumber(pdf_bytes: bytes) -> str:
    """Extract text using pdfplumber (more accurate)."""
    text_parts = []

    with pdfplumber.open(io.BytesIO(pdf_bytes)) as pdf:
        if len(pdf.pages) == 0:
            raise ValueError("PDF has no pages")

        for page_num, page in enumerate(pdf.pages, 1):
            try:
                text = page.extract_text()
                if text:
                    text_parts.append(text)
                logger.debug(f"Extracted page {page_num}")
            except Exception as e:
                logger.warning(f"Failed to extract page {page_num}: {e}")

    full_text = "\n\n".join(text_parts)

    if not full_text.strip():
        raise ValueError("No text content found in PDF")

    return full_text


def _extract_with_pypdf2(pdf_bytes: bytes) -> str:
    """Extract text using PyPDF2 (fallback)."""
    text_parts = []

    reader = PyPDF2.PdfReader(io.BytesIO(pdf_bytes))

    if len(reader.pages) == 0:
        raise ValueError("PDF has no pages")

    for page_num, page in enumerate(reader.pages, 1):
        try:
            text = page.extract_text()
            if text:
                text_parts.append(text)
            logger.debug(f"Extracted page {page_num}")
        except Exception as e:
            logger.warning(f"Failed to extract page {page_num}: {e}")

    full_text = "\n\n".join(text_parts)

    if not full_text.strip():
        raise ValueError("No text content found in PDF")

    return full_text
```

### LightRAG Client: `lightrag_client.py`

```python
"""
LightRAG API client.
"""

import logging
from typing import Dict, Any, List, Optional
import aiohttp

logger = logging.getLogger(__name__)


class LightRAGClient:
    """Async client for LightRAG API."""

    def __init__(self, endpoint: str, api_key: Optional[str] = None):
        """
        Initialize LightRAG client.

        Args:
            endpoint: LightRAG API endpoint URL
            api_key: Optional API key for authentication
        """
        self.endpoint = endpoint.rstrip("/")
        self.api_key = api_key
        self.session: Optional[aiohttp.ClientSession] = None

    async def _get_session(self) -> aiohttp.ClientSession:
        """Get or create aiohttp session."""
        if self.session is None or self.session.closed:
            headers = {}
            if self.api_key:
                headers["Authorization"] = f"Bearer {self.api_key}"

            self.session = aiohttp.ClientSession(headers=headers)

        return self.session

    async def close(self):
        """Close the HTTP session."""
        if self.session and not self.session.closed:
            await self.session.close()

    async def index_document(
        self,
        content: str,
        metadata: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Index a document in LightRAG.

        Args:
            content: Document text content
            metadata: Document metadata

        Returns:
            Indexing result with chunk count

        Raises:
            RuntimeError: If indexing fails
        """
        session = await self._get_session()

        payload = {
            "content": content,
            "metadata": metadata
        }

        try:
            async with session.post(
                f"{self.endpoint}/api/index",
                json=payload,
                timeout=aiohttp.ClientTimeout(total=60)
            ) as response:
                response.raise_for_status()
                result = await response.json()
                logger.info(f"Indexed document: {result}")
                return result

        except aiohttp.ClientError as e:
            raise RuntimeError(f"LightRAG API error: {e}")

    async def query(
        self,
        query: str,
        max_results: int = 5
    ) -> List[Dict[str, Any]]:
        """
        Query LightRAG knowledge base.

        Args:
            query: Natural language query
            max_results: Maximum number of results

        Returns:
            List of result dictionaries
        """
        session = await self._get_session()

        payload = {
            "query": query,
            "max_results": max_results
        }

        try:
            async with session.post(
                f"{self.endpoint}/api/query",
                json=payload,
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                response.raise_for_status()
                results = await response.json()
                logger.info(f"Query returned {len(results)} results")
                return results

        except aiohttp.ClientError as e:
            raise RuntimeError(f"LightRAG API error: {e}")
```

### Configuration: `config.py`

```python
"""
Configuration management for LightRAG MCP server.
"""

import os
from dataclasses import dataclass
from typing import Optional


@dataclass
class Config:
    """Server configuration."""
    lightrag_endpoint: str
    lightrag_api_key: Optional[str]
    log_level: str = "INFO"
    max_file_size_mb: int = 10


_config: Optional[Config] = None


def load_config() -> Config:
    """
    Load configuration from environment variables.

    Environment variables:
        LIGHTRAG_ENDPOINT: LightRAG API endpoint URL (required)
        LIGHTRAG_API_KEY: API key for authentication (optional)
        LOG_LEVEL: Logging level (default: INFO)
        MAX_FILE_SIZE_MB: Maximum file size in MB (default: 10)

    Returns:
        Config object
    """
    global _config

    if _config is not None:
        return _config

    endpoint = os.getenv("LIGHTRAG_ENDPOINT")
    if not endpoint:
        raise ValueError("LIGHTRAG_ENDPOINT environment variable is required")

    _config = Config(
        lightrag_endpoint=endpoint,
        lightrag_api_key=os.getenv("LIGHTRAG_API_KEY"),
        log_level=os.getenv("LOG_LEVEL", "INFO"),
        max_file_size_mb=int(os.getenv("MAX_FILE_SIZE_MB", "10"))
    )

    return _config


def get_config() -> Config:
    """Get current configuration (must be loaded first)."""
    if _config is None:
        raise RuntimeError("Configuration not loaded. Call load_config() first.")
    return _config
```

### `pyproject.toml`

```toml
[project]
name = "lightrag-mcp-server"
version = "1.0.0"
description = "MCP server for LightRAG integration with LibreChat"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
    "pdfplumber>=0.10.0",
    "aiohttp>=3.9.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.0.0",
    "mypy>=1.0.0",
]

[build-system]
requires = ["setuptools>=68.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools]
packages = ["lightrag_mcp"]
package-dir = {"" = "src"}

[project.scripts]
lightrag-mcp = "lightrag_mcp.server:main"
```

### `.env.example`

```bash
# LightRAG Configuration
LIGHTRAG_ENDPOINT=http://localhost:8080
LIGHTRAG_API_KEY=your_api_key_here

# Logging
LOG_LEVEL=INFO

# File Upload Limits
MAX_FILE_SIZE_MB=10
```

---

## Testing Strategy

### Unit Tests: `tests/test_tools.py`

```python
"""
Unit tests for tool handlers.
"""

import pytest
import base64
from lightrag_mcp.tools import upload_to_lightrag_handler


@pytest.mark.asyncio
async def test_upload_pdf_success():
    """Test successful PDF upload."""
    # Read sample PDF
    with open("tests/fixtures/sample.pdf", "rb") as f:
        pdf_bytes = f.read()

    # Prepare arguments
    arguments = {
        "filename": "sample.pdf",
        "content": base64.b64encode(pdf_bytes).decode("utf-8"),
        "mimeType": "application/pdf",
        "metadata": {
            "tags": ["test"],
            "author": "Test User"
        }
    }

    # Call handler
    result = await upload_to_lightrag_handler(arguments)

    # Assertions
    assert "Successfully indexed" in result
    assert "sample.pdf" in result


@pytest.mark.asyncio
async def test_upload_invalid_base64():
    """Test upload with invalid base64 content."""
    arguments = {
        "filename": "test.pdf",
        "content": "not-valid-base64!!!",
        "mimeType": "application/pdf"
    }

    with pytest.raises(ValueError, match="Invalid base64"):
        await upload_to_lightrag_handler(arguments)


@pytest.mark.asyncio
async def test_upload_missing_arguments():
    """Test upload with missing required arguments."""
    arguments = {
        "filename": "test.pdf"
        # Missing content and mimeType
    }

    with pytest.raises(ValueError, match="Missing required arguments"):
        await upload_to_lightrag_handler(arguments)
```

### Integration Testing

Create a test script to verify stdio communication:

```python
# tests/test_stdio.py
"""
Test stdio communication with MCP server.
"""

import json
import subprocess
import sys


def test_mcp_server_initialization():
    """Test server responds to initialize request."""

    # Start server process
    process = subprocess.Popen(
        ["python", "-m", "lightrag_mcp.server"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )

    # Send initialize request
    initialize_request = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "test-client",
                "version": "1.0.0"
            }
        }
    }

    process.stdin.write(json.dumps(initialize_request) + "\n")
    process.stdin.flush()

    # Read response
    response_line = process.stdout.readline()
    response = json.loads(response_line)

    # Verify response
    assert response["jsonrpc"] == "2.0"
    assert response["id"] == 1
    assert "result" in response
    assert response["result"]["serverInfo"]["name"] == "lightrag-mcp"

    # Clean up
    process.terminate()
    process.wait()


if __name__ == "__main__":
    test_mcp_server_initialization()
    print("✅ MCP server stdio communication test passed")
```

---

## Error Handling

### Error Response Format

When tools encounter errors, return helpful error messages:

```python
# Good error message
return [TextContent(
    type="text",
    text=(
        "Error: Unable to extract text from PDF. The file may be "
        "corrupted, encrypted, or image-based. Please ensure the PDF "
        "contains selectable text."
    )
)]

# Bad error message
return [TextContent(type="text", text="Error: NoneType object has no attribute 'split'")]
```

### Common Error Scenarios

| Error Scenario | HTTP/Error Code | User Message |
|----------------|-----------------|--------------|
| Invalid base64 | ValueError | "The file content is not properly encoded. Please try re-uploading the file." |
| File too large | ValueError | "File size exceeds 10MB limit. Please upload a smaller file." |
| Unsupported file type | ValueError | "Only PDF and Markdown files are supported. Please upload a .pdf or .md file." |
| Empty PDF | ValueError | "The PDF appears to be empty or contains no extractable text." |
| LightRAG API down | RuntimeError | "The knowledge base is temporarily unavailable. Please try again in a few moments." |
| Network timeout | TimeoutError | "The upload operation timed out. Please check your connection and try again." |

### Logging Best Practices

```python
# Log levels
logger.debug("Decoded 524288 bytes")           # Detailed debugging info
logger.info("Processing upload: report.pdf")   # Normal operations
logger.warning("Failed to extract page 5")     # Recoverable issues
logger.error("LightRAG API returned 500")      # Errors that affect functionality
logger.exception("Unexpected error")           # Exceptions with stack trace
```

---

## Security Considerations

### Input Validation Checklist

- [ ] Validate base64 encoding before decoding
- [ ] Check file size limits before processing
- [ ] Verify MIME type matches file content
- [ ] Sanitize filenames (remove path separators, special characters)
- [ ] Limit maximum query length to prevent abuse
- [ ] Validate metadata fields don't contain executable code

### Filename Sanitization

```python
import re
from pathlib import Path


def sanitize_filename(filename: str) -> str:
    """
    Sanitize filename to prevent path traversal and other attacks.

    Examples:
        "../../etc/passwd" -> "etc_passwd"
        "report<script>.pdf" -> "report_script.pdf"
    """
    # Remove path components
    filename = Path(filename).name

    # Remove or replace dangerous characters
    filename = re.sub(r'[<>:"|?*]', '_', filename)

    # Remove leading/trailing dots and spaces
    filename = filename.strip('. ')

    # Ensure filename is not empty
    if not filename:
        filename = "unnamed_file"

    return filename
```

### Environment Variable Security

Never hardcode credentials. Always use environment variables:

```python
# ❌ Bad
LIGHTRAG_API_KEY = "sk-1234567890abcdef"

# ✅ Good
LIGHTRAG_API_KEY = os.getenv("LIGHTRAG_API_KEY")
if not LIGHTRAG_API_KEY:
    raise ValueError("LIGHTRAG_API_KEY environment variable is required")
```

---

## LibreChat Configuration

### How to Register Your MCP Server in LibreChat

LibreChat administrators configure MCP servers in their `librechat.yaml` file:

```yaml
# librechat.yaml
endpoints:
  agents:
    capabilities:
      - tools  # Enable tool calling

    # MCP server configurations
    mcpServers:
      lightrag:
        command: python
        args:
          - "-m"
          - "lightrag_mcp.server"
        env:
          LIGHTRAG_ENDPOINT: "http://localhost:8080"
          LIGHTRAG_API_KEY: "${LIGHTRAG_API_KEY}"  # From environment

        # Tool exposure settings
        tools:
          - type: "all"  # Expose all tools from this server
```

**Note to developers:** You don't need to configure this yourself. Provide these instructions to the LibreChat administrator.

### Alternative: Per-User MCP Servers

LibreChat also supports user-specific MCP servers (requires user authentication):

```yaml
# User-specific configuration
mcpServers:
  lightrag:
    userScoped: true  # Each user has their own instance
    command: python
    args: ["-m", "lightrag_mcp.server"]
    env:
      LIGHTRAG_ENDPOINT: "${USER_LIGHTRAG_ENDPOINT}"
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] All tests passing (`pytest tests/`)
- [ ] Logging configured to stderr (not stdout)
- [ ] Environment variables documented in `.env.example`
- [ ] Dependencies specified in `pyproject.toml`
- [ ] Error messages are user-friendly
- [ ] File size limits configured appropriately

### Deployment Steps

1. **Install dependencies:**
   ```bash
   pip install -e .
   ```

2. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with actual values
   ```

3. **Test server manually:**
   ```bash
   python -m lightrag_mcp.server
   # Send test JSON-RPC request
   ```

4. **Add to LibreChat configuration:**
   Edit `librechat.yaml` with MCP server config (see above)

5. **Restart LibreChat:**
   ```bash
   docker-compose restart api
   ```

6. **Verify in LibreChat:**
   - Open LibreChat
   - Start agent conversation
   - Verify tools appear in tool list
   - Test file upload

### Post-Deployment Monitoring

Monitor these log patterns (in stderr):

```
# Success patterns
INFO - Processing upload: document.pdf (application/pdf)
INFO - Extracted 1234 words from document.pdf
INFO - Successfully indexed 'document.pdf' into LightRAG

# Warning patterns (investigate if frequent)
WARNING - Failed to extract page 3: ...
WARNING - LightRAG API slow response: 5.2s

# Error patterns (requires immediate attention)
ERROR - LightRAG API returned 500
ERROR - Connection to LightRAG failed
EXCEPTION - Unexpected error
```

---

## Troubleshooting Guide

### Common Issues

#### Issue: LibreChat can't connect to MCP server

**Symptoms:** Tools don't appear in LibreChat, connection errors in logs

**Checklist:**
1. Verify `command` and `args` in `librechat.yaml` are correct
2. Check `python -m lightrag_mcp.server` runs without errors
3. Ensure no syntax errors in `server.py`
4. Verify environment variables are set
5. Check LibreChat has permission to execute Python

#### Issue: Tool calls fail with "Invalid JSON"

**Symptoms:** Tool executes but returns error about JSON parsing

**Cause:** Writing non-JSON data to stdout

**Solution:** Ensure all logging goes to stderr:
```python
# Wrong
print("Debug info")  # Goes to stdout!

# Correct
logger.info("Debug info")  # Goes to stderr
```

#### Issue: PDF extraction returns empty text

**Symptoms:** "No text content found in PDF" error

**Possible Causes:**
1. PDF is image-based (scanned document) - needs OCR
2. PDF is encrypted
3. PDF uses non-standard encoding

**Solution:** Add OCR support for image-based PDFs:
```python
# Install: pip install pdf2image pytesseract
from pdf2image import convert_from_bytes
import pytesseract

def extract_with_ocr(pdf_bytes):
    images = convert_from_bytes(pdf_bytes)
    text_parts = []
    for image in images:
        text = pytesseract.image_to_string(image)
        text_parts.append(text)
    return "\n\n".join(text_parts)
```

#### Issue: File size limit errors

**Symptoms:** "File too large" errors for valid files

**Solution:** Adjust `MAX_FILE_SIZE_MB` environment variable:
```bash
MAX_FILE_SIZE_MB=20  # Increase to 20MB
```

---

## Performance Optimization

### Recommended Optimizations

1. **Connection Pooling:** Reuse HTTP connections to LightRAG
   ```python
   # Singleton session
   _session = None

   def get_session():
       global _session
       if _session is None:
           _session = aiohttp.ClientSession()
       return _session
   ```

2. **Chunk Large PDFs:** Process large PDFs in smaller chunks
   ```python
   CHUNK_SIZE = 10000  # words per chunk

   if word_count > CHUNK_SIZE:
       chunks = split_into_chunks(text_content, CHUNK_SIZE)
       for i, chunk in enumerate(chunks):
           await client.index_document(chunk, {..., "chunk_id": i})
   ```

3. **Async File Processing:** Use async I/O for file operations
   ```python
   import aiofiles

   async def save_temp_file(content: bytes, filename: str):
       async with aiofiles.open(f"/tmp/{filename}", "wb") as f:
           await f.write(content)
   ```

---

## Appendix: Complete Example Usage

### End-to-End User Flow

**User Workflow:**

1. User opens LibreChat
2. User attaches `research-paper.pdf` in chat
3. User types: *"Index this research paper in my knowledge base"*
4. AI agent (Claude/GPT-4) recognizes the intent
5. AI agent reads the attached file
6. AI agent base64-encodes the file content
7. AI agent calls `upload_to_lightrag` tool with:
   ```json
   {
     "filename": "research-paper.pdf",
     "content": "JVBERi0xLjQKJe...",
     "mimeType": "application/pdf"
   }
   ```
8. Your MCP server processes the request
9. LightRAG indexes the document
10. AI agent responds: *"I've successfully indexed 'research-paper.pdf' into your knowledge base. The document contains 3,456 words and has been processed into 18 semantic chunks."*

**Later Query:**

1. User types: *"What did the research paper say about climate change?"*
2. AI agent calls `query_lightrag` with:
   ```json
   {
     "query": "climate change findings",
     "max_results": 5
   }
   ```
3. Your MCP server queries LightRAG
4. LightRAG returns relevant excerpts
5. AI agent synthesizes answer with sources

---

## Support and Maintenance

### Logging Location

By default, logs go to stderr. To save logs to file:

```python
# In server.py
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stderr),
        logging.FileHandler('logs/mcp_server.log')
    ]
)
```

### Health Check Endpoint (Optional)

Add a simple health check tool:

```python
@app.list_tools()
async def list_tools():
    return [
        # ... existing tools ...
        Tool(
            name="health_check",
            description="Check if LightRAG connection is healthy",
            inputSchema={
                "type": "object",
                "properties": {},
                "additionalProperties": False
            }
        )
    ]

@app.call_tool()
async def call_tool(name, arguments):
    if name == "health_check":
        try:
            config = get_config()
            client = LightRAGClient(config.lightrag_endpoint, config.lightrag_api_key)
            # Simple ping
            await client.query("test", max_results=1)
            return [TextContent(type="text", text="✅ LightRAG connection healthy")]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ LightRAG connection failed: {e}")]
```

---

## Glossary

- **MCP (Model Context Protocol):** Standard protocol for connecting AI assistants to external tools and data sources
- **stdio:** Standard input/output - a communication method using stdin and stdout streams
- **JSON-RPC:** Remote procedure call protocol using JSON
- **Base64:** Binary-to-text encoding scheme for transmitting binary data as ASCII text
- **LightRAG:** Retrieval-Augmented Generation system for document indexing and semantic search
- **Tool Schema:** JSON Schema definition describing a tool's input parameters
- **LibreChat:** Open-source AI chat interface supporting multiple LLM providers

---

## Conclusion

This specification provides everything needed to build a production-ready MCP server for LightRAG integration with LibreChat. The key design decisions are:

1. **Base64 encoding** for file transfer (works within JSON-RPC constraints)
2. **Stdio transport** for LibreChat communication (standard MCP approach)
3. **PDF and Markdown support** (covers 90% of document indexing use cases)
4. **Async implementation** (handles concurrent requests efficiently)
5. **Comprehensive error handling** (user-friendly messages, proper logging)

**Next Steps:**

1. Set up development environment
2. Implement core `server.py` with tool listing
3. Implement `upload_to_lightrag_handler` with PDF processing
4. Test with sample PDFs and Markdown files
5. Configure in LibreChat
6. Deploy and monitor

For questions or issues, refer to:
- MCP SDK Documentation: https://github.com/modelcontextprotocol/python-sdk
- LibreChat MCP Guide: https://docs.librechat.ai/features/mcp
- This specification document

---

**Document Version:** 1.0
**Last Updated:** 2025-01-16
**Author:** Technical Specification for LightRAG MCP Integration
