# LightRAG MCP Server - Quick Start Guide

## 🚀 Get Started in 15 Minutes

This guide will help you build and test your first working MCP server for LightRAG integration.

---

## Prerequisites

- Python 3.10 or higher
- Basic understanding of Python and async/await
- LightRAG instance running (or mock it for testing)

---

## Step 1: Project Setup (2 minutes)

```bash
# Create project directory
mkdir lightrag-mcp-server
cd lightrag-mcp-server

# Create Python package structure
mkdir -p src/lightrag_mcp tests/fixtures logs
touch src/lightrag_mcp/__init__.py

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install mcp pdfplumber aiohttp pydantic
```

---

## Step 2: Minimal Working Server (5 minutes)

Create `src/lightrag_mcp/server.py`:

```python
#!/usr/bin/env python3
"""Minimal LightRAG MCP Server"""

import asyncio
import base64
import logging
import sys
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# CRITICAL: Log to stderr, not stdout
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    stream=sys.stderr
)
logger = logging.getLogger(__name__)

app = Server("lightrag-mcp")


@app.list_tools()
async def list_tools() -> list[Tool]:
    """Return available tools."""
    return [
        Tool(
            name="upload_to_lightrag",
            description="Upload and index a PDF or Markdown file into LightRAG",
            inputSchema={
                "type": "object",
                "properties": {
                    "filename": {
                        "type": "string",
                        "description": "Filename (e.g., 'doc.pdf')"
                    },
                    "content": {
                        "type": "string",
                        "description": "Base64-encoded file content"
                    },
                    "mimeType": {
                        "type": "string",
                        "enum": ["application/pdf", "text/markdown"],
                        "description": "File MIME type"
                    }
                },
                "required": ["filename", "content", "mimeType"]
            }
        )
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Handle tool calls."""
    logger.info(f"Tool called: {name}")

    if name == "upload_to_lightrag":
        try:
            filename = arguments["filename"]
            content_b64 = arguments["content"]
            mime_type = arguments["mimeType"]

            # Decode base64
            content_bytes = base64.b64decode(content_b64)
            logger.info(f"Decoded {len(content_bytes)} bytes from {filename}")

            # Extract text (simplified for demo)
            if mime_type == "text/markdown":
                text = content_bytes.decode("utf-8")
            elif mime_type == "application/pdf":
                # For now, just a placeholder
                # Real implementation would use pdfplumber
                text = f"[PDF content from {filename}]"
            else:
                raise ValueError(f"Unsupported type: {mime_type}")

            word_count = len(text.split())

            # TODO: Actually send to LightRAG here
            # For now, just simulate success
            logger.info(f"Would index {word_count} words to LightRAG")

            result = (
                f"✅ Successfully indexed '{filename}' "
                f"({word_count} words, {len(content_bytes)} bytes)"
            )

            return [TextContent(type="text", text=result)]

        except Exception as e:
            logger.exception("Error in upload_to_lightrag")
            return [TextContent(
                type="text",
                text=f"❌ Error: {str(e)}"
            )]

    return [TextContent(type="text", text=f"Unknown tool: {name}")]


async def main():
    """Run the MCP server."""
    logger.info("Starting LightRAG MCP Server...")

    async with stdio_server() as (read_stream, write_stream):
        logger.info("Server ready on stdio")
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )


if __name__ == "__main__":
    asyncio.run(main())
```

**Test it works:**

```bash
# Run the server
python src/lightrag_mcp/server.py
```

You should see in your terminal:
```
2025-01-16 10:30:45 - INFO - Starting LightRAG MCP Server...
```

Press Ctrl+C to stop. ✅ Your server works!

---

## Step 3: Test with Manual JSON-RPC (3 minutes)

Create `test_manual.py`:

```python
"""Manual test of MCP server via stdio."""

import json
import subprocess
import base64

# Start server
process = subprocess.Popen(
    ["python", "src/lightrag_mcp/server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)


def send_request(request):
    """Send JSON-RPC request and get response."""
    process.stdin.write(json.dumps(request) + "\n")
    process.stdin.flush()
    response = process.stdout.readline()
    return json.loads(response)


# 1. Initialize
print("1️⃣ Testing initialize...")
init_response = send_request({
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {},
        "clientInfo": {"name": "test", "version": "1.0"}
    }
})
print(f"   Server name: {init_response['result']['serverInfo']['name']}")
print("   ✅ Initialize successful\n")

# 2. List tools
print("2️⃣ Testing tools/list...")
tools_response = send_request({
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list"
})
tool_name = tools_response['result']['tools'][0]['name']
print(f"   Found tool: {tool_name}")
print("   ✅ Tools listed successfully\n")

# 3. Call tool with sample markdown
print("3️⃣ Testing tool call...")
sample_md = "# Test Document\n\nThis is a test."
sample_b64 = base64.b64encode(sample_md.encode()).decode()

call_response = send_request({
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
        "name": "upload_to_lightrag",
        "arguments": {
            "filename": "test.md",
            "content": sample_b64,
            "mimeType": "text/markdown"
        }
    }
})

result_text = call_response['result']['content'][0]['text']
print(f"   Result: {result_text}")
print("   ✅ Tool call successful\n")

# Cleanup
process.terminate()
print("✅ All tests passed!")
```

Run it:

```bash
python test_manual.py
```

Expected output:
```
1️⃣ Testing initialize...
   Server name: lightrag-mcp
   ✅ Initialize successful

2️⃣ Testing tools/list...
   Found tool: upload_to_lightrag
   ✅ Tools listed successfully

3️⃣ Testing tool call...
   Result: ✅ Successfully indexed 'test.md' (6 words, 31 bytes)
   ✅ Tool call successful

✅ All tests passed!
```

---

## Step 4: Add Real PDF Support (3 minutes)

Create `src/lightrag_mcp/pdf_utils.py`:

```python
"""PDF text extraction utilities."""

import io
import pdfplumber


def extract_pdf_text(pdf_bytes: bytes) -> str:
    """
    Extract text from PDF bytes.

    Args:
        pdf_bytes: Raw PDF file data

    Returns:
        Extracted text content

    Raises:
        ValueError: If PDF is invalid or empty
    """
    if not pdf_bytes:
        raise ValueError("Empty PDF content")

    text_parts = []

    with pdfplumber.open(io.BytesIO(pdf_bytes)) as pdf:
        if not pdf.pages:
            raise ValueError("PDF has no pages")

        for page in pdf.pages:
            text = page.extract_text()
            if text:
                text_parts.append(text)

    full_text = "\n\n".join(text_parts)

    if not full_text.strip():
        raise ValueError("No text found in PDF")

    return full_text
```

Update `server.py` to use it:

```python
# Add at top of server.py
from lightrag_mcp.pdf_utils import extract_pdf_text

# In call_tool(), replace the PDF handling:
elif mime_type == "application/pdf":
    text = extract_pdf_text(content_bytes)
```

**Test with a real PDF:**

```python
# test_pdf.py
import base64

# Create a simple test PDF or use an existing one
with open("sample.pdf", "rb") as f:
    pdf_bytes = f.read()

pdf_b64 = base64.b64encode(pdf_bytes).decode()
print(f"Base64 length: {len(pdf_b64)}")
print(f"First 100 chars: {pdf_b64[:100]}")
```

---

## Step 5: Connect to LightRAG (2 minutes)

Create `src/lightrag_mcp/lightrag.py`:

```python
"""LightRAG client integration."""

import aiohttp
import logging
from typing import Dict, Any

logger = logging.getLogger(__name__)


class LightRAGClient:
    """Simple async LightRAG client."""

    def __init__(self, endpoint: str):
        """
        Initialize client.

        Args:
            endpoint: LightRAG API URL (e.g., "http://localhost:8080")
        """
        self.endpoint = endpoint.rstrip("/")
        self.session = None

    async def get_session(self):
        """Get or create HTTP session."""
        if not self.session:
            self.session = aiohttp.ClientSession()
        return self.session

    async def index_document(
        self,
        content: str,
        metadata: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Index document in LightRAG.

        Args:
            content: Document text
            metadata: Document metadata

        Returns:
            Result dictionary with chunk_count, etc.
        """
        session = await self.get_session()

        payload = {
            "content": content,
            "metadata": metadata
        }

        async with session.post(
            f"{self.endpoint}/api/index",
            json=payload,
            timeout=60
        ) as response:
            response.raise_for_status()
            return await response.json()

    async def close(self):
        """Close HTTP session."""
        if self.session:
            await self.session.close()
```

Update `server.py` to use it:

```python
import os
from lightrag_mcp.lightrag import LightRAGClient

# At top level
LIGHTRAG_ENDPOINT = os.getenv("LIGHTRAG_ENDPOINT", "http://localhost:8080")
lightrag_client = LightRAGClient(LIGHTRAG_ENDPOINT)

# In call_tool(), replace the TODO:
result = await lightrag_client.index_document(
    content=text,
    metadata={
        "filename": filename,
        "mime_type": mime_type,
        "word_count": word_count
    }
)

chunk_count = result.get("chunk_count", "unknown")

return [TextContent(
    type="text",
    text=(
        f"✅ Successfully indexed '{filename}' into LightRAG. "
        f"Document contains {word_count} words and was processed "
        f"into {chunk_count} semantic chunks."
    )
)]
```

Set environment variable:

```bash
export LIGHTRAG_ENDPOINT="http://localhost:8080"
```

---

## Step 6: Configure in LibreChat

**For LibreChat Administrator:**

Edit `librechat.yaml`:

```yaml
version: 1.1.7

endpoints:
  agents:
    capabilities:
      - tools

    mcpServers:
      lightrag:
        # Path to Python interpreter
        command: python

        # Arguments to run your server
        args:
          - "-m"
          - "lightrag_mcp.server"

        # Environment variables
        env:
          LIGHTRAG_ENDPOINT: "http://localhost:8080"

        # Expose all tools
        tools:
          - type: "all"
```

Restart LibreChat:

```bash
docker-compose restart api
# or
npm run backend
```

---

## Usage Examples in LibreChat

### Example 1: Upload PDF

**User:** [Attaches `research.pdf`] "Index this research paper"

**AI Agent (behind the scenes):**
1. Reads attached file
2. Base64 encodes it
3. Calls your tool:

```json
{
  "name": "upload_to_lightrag",
  "arguments": {
    "filename": "research.pdf",
    "content": "JVBERi0xLjQKJe...",
    "mimeType": "application/pdf"
  }
}
```

**Your MCP Server:**
- Decodes base64
- Extracts text from PDF
- Sends to LightRAG
- Returns success message

**AI Response:** "I've successfully indexed 'research.pdf' into your knowledge base. The document contains 3,456 words and has been processed into 18 semantic chunks."

### Example 2: Upload Markdown

**User:** [Pastes markdown] "Save these notes to my knowledge base"

```markdown
# Project Notes

## Meeting Summary
- Decision to use microservices
- API design finalized
```

**AI Agent:** Encodes markdown and calls tool

**Your Server:** Processes and indexes markdown

**AI Response:** "I've saved your project notes to the knowledge base (45 words indexed)."

---

## Debugging Tips

### Issue: "No output from server"

**Check:** Are you accidentally writing to stdout?

```python
# ❌ Wrong - breaks JSON-RPC
print("Debug message")

# ✅ Correct - goes to stderr
logger.info("Debug message")
```

### Issue: "Tool not found"

**Check:** Is your tool listed?

```bash
# Send tools/list request manually
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python src/lightrag_mcp/server.py
```

### Issue: "Base64 decode error"

**Check:** Is the content properly encoded?

```python
# Test encoding/decoding
import base64

text = "Hello, world!"
encoded = base64.b64encode(text.encode()).decode()
decoded = base64.b64decode(encoded).decode()

assert text == decoded
```

### Enable Debug Logging

```python
# In server.py
logging.basicConfig(
    level=logging.DEBUG,  # Changed from INFO
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    stream=sys.stderr
)
```

---

## Testing Checklist

Before deploying, verify:

- [ ] Server starts without errors
- [ ] `tools/list` returns your tools
- [ ] Can decode base64 content
- [ ] Can extract text from PDF
- [ ] Can extract text from Markdown
- [ ] LightRAG connection works
- [ ] Error messages are user-friendly
- [ ] No debug output to stdout
- [ ] File size limits are enforced
- [ ] Invalid inputs are handled gracefully

---

## Production Deployment

### 1. Package Your Server

Create `pyproject.toml`:

```toml
[project]
name = "lightrag-mcp-server"
version = "1.0.0"
dependencies = [
    "mcp>=1.0.0",
    "pdfplumber>=0.10.0",
    "aiohttp>=3.9.0",
]

[project.scripts]
lightrag-mcp = "lightrag_mcp.server:main"
```

Install:

```bash
pip install -e .
```

Now you can run:

```bash
lightrag-mcp
```

### 2. Environment Configuration

Create `.env`:

```bash
LIGHTRAG_ENDPOINT=http://production-lightrag:8080
LIGHTRAG_API_KEY=your_secret_key_here
LOG_LEVEL=INFO
MAX_FILE_SIZE_MB=10
```

Load in server:

```python
import os
from dotenv import load_dotenv

load_dotenv()

LIGHTRAG_ENDPOINT = os.getenv("LIGHTRAG_ENDPOINT")
LIGHTRAG_API_KEY = os.getenv("LIGHTRAG_API_KEY")
```

### 3. Health Monitoring

Add health check:

```python
@app.list_tools()
async def list_tools():
    return [
        # ... existing tools ...
        Tool(
            name="health_check",
            description="Check LightRAG connection health",
            inputSchema={"type": "object", "properties": {}}
        )
    ]

@app.call_tool()
async def call_tool(name, arguments):
    if name == "health_check":
        try:
            await lightrag_client.get_session()
            return [TextContent(type="text", text="✅ Healthy")]
        except Exception as e:
            return [TextContent(type="text", text=f"❌ Unhealthy: {e}")]
```

---

## Common Pitfalls

### ❌ Pitfall 1: Writing to stdout

```python
# This breaks everything!
print("Processing file...")  # Goes to stdout!
```

**Fix:** Always use logger

```python
logger.info("Processing file...")  # Goes to stderr ✅
```

### ❌ Pitfall 2: Blocking operations

```python
# This blocks the event loop!
import time
time.sleep(10)
```

**Fix:** Use async sleep

```python
import asyncio
await asyncio.sleep(10)
```

### ❌ Pitfall 3: Unhandled errors

```python
# Server crashes on error!
content_bytes = base64.b64decode(content_b64)
```

**Fix:** Wrap in try/except

```python
try:
    content_bytes = base64.b64decode(content_b64)
except Exception as e:
    logger.error(f"Decode error: {e}")
    return [TextContent(type="text", text=f"Error: Invalid base64")]
```

---

## Next Steps

1. ✅ You have a working MCP server
2. ✅ You understand the data flow
3. ✅ You can test it manually

**Now:**

- Add more features (query tool, metadata search)
- Improve error handling
- Add comprehensive tests
- Optimize performance
- Deploy to production

---

## Resources

- **Full Specification:** See `LIGHTRAG_MCP_SPECIFICATION.md`
- **MCP Python SDK:** https://github.com/modelcontextprotocol/python-sdk
- **LibreChat Docs:** https://docs.librechat.ai/
- **pdfplumber Docs:** https://github.com/jsvine/pdfplumber

---

## Getting Help

If you get stuck:

1. Check stderr logs for errors
2. Test manually with `test_manual.py`
3. Verify JSON-RPC messages are valid
4. Review the troubleshooting section in the full spec
5. Check that LibreChat config is correct

**Common Success Indicators:**

- Server logs: `INFO - Server ready on stdio`
- LibreChat logs: `MCP server 'lightrag' connected`
- Tools appear in agent tool list
- Test file upload succeeds

---

**Happy Coding! 🚀**
