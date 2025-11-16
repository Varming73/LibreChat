# LightRAG MCP Server - Architecture Guide

## Visual Reference for Understanding the Complete System

This document provides visual diagrams and architectural overviews to help you understand how all the pieces fit together.

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         LibreChat                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Frontend (React)                       │   │
│  │                                                            │   │
│  │  • File upload component                                  │   │
│  │  • Chat interface                                         │   │
│  │  • Agent tool selection                                   │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │ HTTP/WebSocket                             │
│  ┌──────────────────▼───────────────────────────────────────┐   │
│  │                 Backend (Node.js)                         │   │
│  │                                                            │   │
│  │  • File storage (S3/Local/Firebase)                       │   │
│  │  • Agent orchestration                                    │   │
│  │  • MCP connection manager                                 │   │
│  │  • Tool call handler                                      │   │
│  └──────────────────┬───────────────────────────────────────┘   │
└────────────────────┬┼───────────────────────────────────────────┘
                     ││
                     ││ stdio (JSON-RPC 2.0)
                     ││
        ┌────────────▼▼──────────────┐
        │  Child Process (Python)     │
        │                             │
        │  ┌───────────────────────┐  │
        │  │  MCP Server           │  │
        │  │  (Your Code)          │  │
        │  │                       │  │
        │  │  • Tool registration  │  │
        │  │  • Tool execution     │  │
        │  │  • Base64 handling    │  │
        │  │  • PDF processing     │  │
        │  └───────────┬───────────┘  │
        └──────────────┼──────────────┘
                       │ HTTP/gRPC
        ┌──────────────▼──────────────┐
        │       LightRAG Backend       │
        │                              │
        │  • Document indexing         │
        │  • Vector database           │
        │  • Semantic search           │
        │  • Query processing          │
        └──────────────────────────────┘
```

---

## Communication Flow

### Protocol Stack

```
Layer 5: User Interaction
         ↕
Layer 4: HTTP/WebSocket (Browser ↔ LibreChat Backend)
         ↕
Layer 3: JSON-RPC 2.0 (LibreChat ↔ MCP Server)
         ↕
Layer 2: stdio Transport (stdin/stdout)
         ↕
Layer 1: Operating System Process Management
```

### stdio Communication Detail

```
LibreChat Process                       MCP Server Process
┌─────────────────┐                    ┌──────────────────┐
│                 │                    │                  │
│  MCPManager     │                    │   server.py      │
│                 │                    │                  │
│  Write to ──────┼────── stdin ───────►  Read from       │
│  child stdin    │                    │  sys.stdin       │
│                 │                    │                  │
│  Read from  ◄───┼────── stdout ──────┼─  Write to       │
│  child stdout   │                    │  sys.stdout      │
│                 │                    │                  │
│  Read from  ◄───┼────── stderr ──────┼─  Write to       │
│  child stderr   │                    │  sys.stderr      │
│  (logs)         │                    │  (logs)          │
│                 │                    │                  │
└─────────────────┘                    └──────────────────┘
```

**Critical Rules:**
- ✅ stdout = JSON-RPC messages ONLY
- ✅ stderr = logs, debug info, errors
- ❌ Never print() to stdout (breaks JSON-RPC)
- ❌ Never write non-JSON to stdout

---

## Complete User Flow

### Upload Document Scenario

```
┌─────┐  ┌──────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐
│User │  │LibreChat │  │   AI   │  │   MCP   │  │LightRAG  │
│     │  │ Frontend │  │  Agent │  │  Server │  │          │
└──┬──┘  └────┬─────┘  └───┬────┘  └────┬────┘  └────┬─────┘
   │          │             │            │            │
   │ 1. Attach│PDF          │            │            │
   ├─────────►│             │            │            │
   │          │             │            │            │
   │          │ 2. Upload to storage     │            │
   │          ├────────┐    │            │            │
   │          │◄───────┘    │            │            │
   │          │ file_id     │            │            │
   │          │             │            │            │
   │ 3. "Index this doc"    │            │            │
   ├─────────►│             │            │            │
   │          │             │            │            │
   │          │ 4. User message + context│            │
   │          ├────────────►│            │            │
   │          │             │            │            │
   │          │             │ 5. Decide to use tool   │
   │          │             ├───────┐    │            │
   │          │             │◄──────┘    │            │
   │          │             │            │            │
   │          │             │ 6. Read file│           │
   │          │             ├────────┐   │            │
   │          │             │◄───────┘   │            │
   │          │             │ base64     │            │
   │          │             │            │            │
   │          │             │ 7. tools/call           │
   │          │             │ (JSON-RPC) │            │
   │          │             ├───────────►│            │
   │          │             │            │            │
   │          │             │            │ 8. Decode  │
   │          │             │            ├───────┐    │
   │          │             │            │◄──────┘    │
   │          │             │            │            │
   │          │             │            │ 9. Extract │
   │          │             │            │    text    │
   │          │             │            ├───────┐    │
   │          │             │            │◄──────┘    │
   │          │             │            │            │
   │          │             │            │10. Index   │
   │          │             │            ├───────────►│
   │          │             │            │            │
   │          │             │            │11. Indexed │
   │          │             │            │◄───────────┤
   │          │             │            │            │
   │          │             │12. Success │            │
   │          │             │ (JSON-RPC) │            │
   │          │             │◄───────────┤            │
   │          │             │            │            │
   │          │13. "Successfully indexed"│            │
   │          │◄────────────┤            │            │
   │          │             │            │            │
   │14. Success│message      │            │            │
   │◄─────────┤             │            │            │
   │          │             │            │            │
```

---

## JSON-RPC Message Format

### Request Format

```
stdin → MCP Server

{
  "jsonrpc": "2.0",           // Protocol version
  "id": 123,                  // Request ID (for matching response)
  "method": "tools/call",     // RPC method name
  "params": {                 // Method parameters
    "name": "upload_to_lightrag",
    "arguments": {
      "filename": "doc.pdf",
      "content": "JVBERi0x...",  // Base64
      "mimeType": "application/pdf"
    }
  }
}
```

### Response Format

```
stdout ← MCP Server

{
  "jsonrpc": "2.0",           // Protocol version
  "id": 123,                  // Same ID as request
  "result": {                 // Success result
    "content": [
      {
        "type": "text",
        "text": "Successfully indexed 'doc.pdf'..."
      }
    ]
  }
}
```

### Error Response Format

```
stdout ← MCP Server

{
  "jsonrpc": "2.0",
  "id": 123,
  "error": {                  // Error instead of result
    "code": -32602,           // JSON-RPC error code
    "message": "Invalid params",
    "data": {
      "details": "content must be base64 encoded"
    }
  }
}
```

---

## Data Transformation Pipeline

### PDF Upload Pipeline

```
Original PDF File
     │
     │ (User attaches in LibreChat)
     ▼
┌──────────────────────────┐
│ LibreChat File Storage   │
│                          │
│ • S3 bucket              │
│ • Local filesystem       │
│ • Firebase storage       │
└────────┬─────────────────┘
         │ (AI agent reads file)
         ▼
    Binary Data
    (PDF bytes)
         │
         │ base64.b64encode()
         ▼
    Base64 String
    "JVBERi0xLjQKJe..."
         │
         │ (Sent via JSON-RPC)
         ▼
┌──────────────────────────┐
│   MCP Server Receives    │
│                          │
│   arguments.content      │
└────────┬─────────────────┘
         │ base64.b64decode()
         ▼
    Binary Data
    (PDF bytes)
         │
         │ pdfplumber.open()
         ▼
┌──────────────────────────┐
│   PDF Text Extraction    │
│                          │
│   • Parse PDF structure  │
│   • Extract text         │
│   • Handle formatting    │
└────────┬─────────────────┘
         │
         ▼
    Plain Text
    "This is the document..."
         │
         │ (Send to LightRAG)
         ▼
┌──────────────────────────┐
│  LightRAG Processing     │
│                          │
│  • Chunking              │
│  • Embedding generation  │
│  • Vector indexing       │
└──────────────────────────┘
```

---

## Tool Schema Structure

### Anatomy of a Tool Definition

```json
{
  "name": "upload_to_lightrag",     // ← Unique tool identifier

  "description": "...",              // ← AI agent reads this to decide
                                     //    when to use the tool

  "inputSchema": {                   // ← JSON Schema definition
    "type": "object",
    "properties": {

      "filename": {                  // ← Parameter name
        "type": "string",            // ← Data type
        "description": "...",        // ← AI agent reads this
        "pattern": "^[\\w-. ]+\\.(pdf|md)$"  // ← Validation regex
      },

      "content": {
        "type": "string",
        "description": "Base64-encoded file content",
        "contentEncoding": "base64"  // ← Hint: this is base64
      },

      "mimeType": {
        "type": "string",
        "enum": [                    // ← Restrict to specific values
          "application/pdf",
          "text/markdown"
        ]
      }

    },
    "required": ["filename", "content", "mimeType"],  // ← Mandatory fields
    "additionalProperties": false    // ← Reject unknown fields
  }
}
```

### How AI Agents Use Tool Schemas

```
AI Agent Decision Process:

1. Read tool description
   ↓
   "Does this tool match user's intent?"

2. If yes, read inputSchema
   ↓
   "What parameters do I need to provide?"

3. Extract or generate parameter values
   ↓
   Example: filename from attached file
            content by reading and encoding file
            mimeType from file extension

4. Validate against schema
   ↓
   "Are all required fields present?"
   "Do values match the type constraints?"

5. Call tool with arguments
   ↓
   {
     "name": "upload_to_lightrag",
     "arguments": { ... }
   }
```

---

## File Handling Strategies

### Strategy Comparison

```
┌────────────────────┬──────────────┬──────────────┬─────────────┐
│ Strategy           │ File Size    │ Complexity   │ LibreChat   │
│                    │ Limit        │              │ Changes     │
├────────────────────┼──────────────┼──────────────┼─────────────┤
│ 1. Base64 Direct   │ ~10 MB       │ Simple       │ None        │
│   (Recommended)    │              │              │             │
├────────────────────┼──────────────┼──────────────┼─────────────┤
│ 2. File ID Ref     │ Unlimited    │ Medium       │ Major       │
│                    │              │              │ (Need API)  │
├────────────────────┼──────────────┼──────────────┼─────────────┤
│ 3. URL Reference   │ Unlimited    │ Medium       │ Medium      │
│                    │              │              │ (Need URLs) │
├────────────────────┼──────────────┼──────────────┼─────────────┤
│ 4. Shared Storage  │ Unlimited    │ Complex      │ None        │
│                    │              │              │ (Security!) │
└────────────────────┴──────────────┴──────────────┴─────────────┘
```

### Strategy 1: Base64 Direct (Recommended)

```
User File (5 MB PDF)
     │
     ▼
Base64 Encode (6.67 MB string)
     │
     ▼
JSON-RPC Parameter
     │
     ▼
MCP Server Decodes
     │
     ▼
Original File (5 MB)
```

**Pros:**
- Works immediately, no LibreChat changes
- Self-contained (no external dependencies)
- Simple implementation

**Cons:**
- 33% size overhead
- Not practical for >10 MB files
- All data in memory

### Strategy 2: File ID Reference (Future)

```
User File → LibreChat Storage → file_id: "abc123"
                                      │
                                      ▼
              ┌───────────────────────────────────┐
              │ MCP Server                        │
              │                                   │
              │ 1. Receive file_id: "abc123"      │
              │ 2. Call LibreChat API             │
              │    GET /api/files/abc123          │
              │ 3. Receive file stream            │
              │ 4. Process file                   │
              └───────────────────────────────────┘
```

**Pros:**
- No file size limit
- Efficient (no base64 overhead)
- Leverages existing storage

**Cons:**
- Requires LibreChat changes
- Need authentication/authorization
- More complex

---

## Error Handling Architecture

### Error Propagation

```
Error Source          Error Handler           User Feedback
─────────────────────────────────────────────────────────────

Invalid Base64
    │
    ├─► ValueError ──────► catch block ────► "Invalid file encoding"

Empty PDF
    │
    ├─► ValueError ──────► catch block ────► "PDF contains no text"

LightRAG API 500
    │
    ├─► RuntimeError ────► catch block ────► "Knowledge base temporarily unavailable"

Network Timeout
    │
    ├─► TimeoutError ────► catch block ────► "Upload timed out, please retry"

Unexpected Error
    │
    ├─► Exception ───────► catch block ────► "An error occurred: [details]"
                            │
                            └─► Log with stack trace to stderr
```

### Error Response Format

```python
# In your tool handler:

try:
    # Process file
    result = process_file(content)

    # Success response
    return [TextContent(
        type="text",
        text=f"✅ Success: {result}"
    )]

except ValueError as e:
    # User input error
    logger.warning(f"Validation error: {e}")
    return [TextContent(
        type="text",
        text=f"❌ Invalid input: {e}"
    )]

except RuntimeError as e:
    # System error
    logger.error(f"System error: {e}")
    return [TextContent(
        type="text",
        text=f"❌ System error: {e}"
    )]

except Exception as e:
    # Unexpected error
    logger.exception("Unexpected error")
    return [TextContent(
        type="text",
        text=f"❌ Unexpected error: {e}"
    )]
```

---

## Performance Considerations

### Memory Usage

```
Request Size Calculation:

PDF Size: 5 MB
    │
    ├─► Base64 Encoded: 5 MB × 1.33 = 6.67 MB
    │
    ├─► JSON Wrapper: ~7 MB (in JSON-RPC message)
    │
    ├─► Decoded in Memory: 5 MB
    │
    ├─► Text Extraction: ~0.5 MB (plain text)
    │
    └─► LightRAG Request: ~0.5 MB

Peak Memory Usage: ~13 MB per request
(base64 string + decoded bytes + extracted text)
```

### Async Benefits

```
Synchronous (Blocking):

Request 1 ──────────► [Process 5s] ──────────► Done
Request 2                              ──────────► [Process 5s] ──────────► Done
Request 3                                                        ──────────► [Process 5s]

Total Time: 15 seconds


Asynchronous (Non-blocking):

Request 1 ──────────► [Process 5s] ──────────► Done
Request 2 ──────────► [Process 5s] ──────────► Done
Request 3 ──────────► [Process 5s] ──────────► Done

Total Time: 5 seconds (concurrent processing)
```

---

## Security Architecture

### Trust Boundaries

```
┌──────────────────────────────────────────────────────┐
│ LibreChat (Trusted)                                  │
│                                                      │
│ • User authentication                                │
│ • File upload validation                             │
│ • Access control                                     │
└─────────────────────┬────────────────────────────────┘
                      │
        ═══════════════╪═══════════════
        Trust Boundary │
        ═══════════════╪═══════════════
                      │
┌─────────────────────▼────────────────────────────────┐
│ MCP Server (Semi-trusted)                            │
│                                                      │
│ • Validate all inputs                                │
│ • Sanitize filenames                                 │
│ • Enforce size limits                                │
│ • Type checking                                      │
└─────────────────────┬────────────────────────────────┘
                      │
        ═══════════════╪═══════════════
        Trust Boundary │
        ═══════════════╪═══════════════
                      │
┌─────────────────────▼────────────────────────────────┐
│ LightRAG (Untrusted / External)                      │
│                                                      │
│ • Don't trust responses                              │
│ • Validate returned data                             │
│ • Handle errors gracefully                           │
└──────────────────────────────────────────────────────┘
```

### Input Validation Layers

```
Layer 1: JSON Schema Validation
         ↓
    ✓ Correct types
    ✓ Required fields present
    ✓ Enum values valid
         ↓
Layer 2: Application Validation
         ↓
    ✓ Base64 format valid
    ✓ File size within limits
    ✓ Filename sanitized
         ↓
Layer 3: Content Validation
         ↓
    ✓ PDF structure valid
    ✓ MIME type matches content
    ✓ Text extraction successful
         ↓
    Process File
```

---

## Deployment Architecture

### Development Setup

```
Developer Machine
┌────────────────────────────────────────┐
│                                        │
│  Terminal 1: LibreChat                 │
│  $ npm run dev                         │
│  ├─► localhost:3080                    │
│  └─► Spawns MCP server on demand       │
│                                        │
│  Terminal 2: LightRAG (Mock)           │
│  $ python mock_lightrag.py             │
│  └─► localhost:8080                    │
│                                        │
│  Terminal 3: Logs                      │
│  $ tail -f mcp_server.log              │
│                                        │
└────────────────────────────────────────┘
```

### Production Setup

```
Production Environment
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  ┌──────────────────┐      ┌──────────────────┐           │
│  │  LibreChat       │      │  MCP Server      │           │
│  │  (Docker)        │◄────►│  (Process)       │           │
│  │                  │stdio │                  │           │
│  │  Port: 3080      │      │  No port         │           │
│  └──────────────────┘      └─────────┬────────┘           │
│           │                           │                    │
│           │                           │ HTTP               │
│           │                           ▼                    │
│           │                  ┌──────────────────┐          │
│           │                  │  LightRAG        │          │
│           │                  │  (Service)       │          │
│           │                  │  Port: 8080      │          │
│           │                  └──────────────────┘          │
│           │                                                │
│           ▼                                                │
│  ┌──────────────────┐                                      │
│  │  File Storage    │                                      │
│  │  (S3/Local)      │                                      │
│  └──────────────────┘                                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Monitoring and Observability

### Log Structure

```
stderr Output:

2025-01-16 10:30:45 - INFO - Starting LightRAG MCP Server
2025-01-16 10:30:45 - INFO - Loaded config: endpoint=http://localhost:8080
2025-01-16 10:30:45 - INFO - Server ready on stdio
2025-01-16 10:31:12 - INFO - Tool called: upload_to_lightrag
2025-01-16 10:31:12 - INFO - Processing: research.pdf (application/pdf)
2025-01-16 10:31:12 - INFO - Decoded 5242880 bytes
2025-01-16 10:31:14 - INFO - Extracted 3456 words from research.pdf
2025-01-16 10:31:15 - INFO - Successfully indexed document
2025-01-16 10:31:15 - INFO - LightRAG returned 18 chunks
```

### Metrics to Track

```
Performance Metrics:
┌─────────────────────┬──────────────────┐
│ Metric              │ Target           │
├─────────────────────┼──────────────────┤
│ Startup time        │ < 5 seconds      │
│ PDF decode time     │ < 1 second       │
│ Text extraction     │ < 3 seconds      │
│ LightRAG index time │ < 10 seconds     │
│ Total upload time   │ < 15 seconds     │
│ Memory usage        │ < 512 MB         │
└─────────────────────┴──────────────────┘

Error Rates:
- Base64 decode errors: < 1%
- PDF extraction errors: < 5%
- LightRAG API errors: < 2%
```

---

## Conclusion

This architecture guide provides visual reference for:

✅ How LibreChat communicates with your MCP server
✅ The data flow from file upload to indexing
✅ JSON-RPC message structure
✅ Error handling patterns
✅ Security boundaries
✅ Performance considerations

**Key Takeaways:**

1. **stdio is simple but strict** - Only JSON-RPC to stdout
2. **Base64 works for most files** - Up to 10MB efficiently
3. **Async is essential** - Handle multiple uploads concurrently
4. **Errors should be user-friendly** - Clear messages, not stack traces
5. **Security is layered** - Validate at every boundary

**Next Steps:**

- Review the Quick Start Guide for hands-on implementation
- Reference the Technical Specification for detailed requirements
- Build incrementally and test at each step

---

**Document Version:** 1.0
**Last Updated:** 2025-01-16
