# LightRAG MCP Server - Complete Documentation Package

> **Build an MCP server to integrate LightRAG document indexing with LibreChat**

## 📚 Documentation Overview

This package contains everything you need to build a production-ready MCP (Model Context Protocol) server that enables file uploads from LibreChat to LightRAG, even if you have no prior knowledge of the LibreChat codebase.

### What You're Building

An **MCP server** that allows users to:
1. Attach PDF or Markdown files in LibreChat's chat interface
2. Ask the AI to index those files
3. Have the files processed and stored in LightRAG for semantic search
4. Query the indexed knowledge base later

### What You Get

✅ **Complete technical specification** with all requirements
✅ **Step-by-step implementation guide** to get started in 15 minutes
✅ **Visual architecture diagrams** to understand the system
✅ **Production-ready code examples** in Python
✅ **Testing strategies** and debugging tips
✅ **Security best practices** and error handling patterns

---

## 🗂️ Document Guide

### **Start Here: Which Document to Read First?**

```
┌─────────────────────────────────────────────────────┐
│  Are you new to MCP and want to get started fast?   │
│                                                     │
│  → Read: LIGHTRAG_MCP_QUICKSTART.md                 │
│                                                     │
│  This guide gets you from zero to working MCP       │
│  server in 15 minutes with hands-on examples.       │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Do you want to understand the system architecture? │
│                                                     │
│  → Read: LIGHTRAG_MCP_ARCHITECTURE.md               │
│                                                     │
│  This guide has visual diagrams showing how all     │
│  the pieces fit together and communicate.           │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Are you ready to build the production version?     │
│                                                     │
│  → Read: LIGHTRAG_MCP_SPECIFICATION.md              │
│                                                     │
│  This comprehensive spec has all functional and     │
│  non-functional requirements, security, testing,    │
│  and deployment guidelines.                         │
└─────────────────────────────────────────────────────┘
```

---

## 📖 Document Descriptions

### 1. **[LIGHTRAG_MCP_QUICKSTART.md](./LIGHTRAG_MCP_QUICKSTART.md)**

**Purpose:** Get started quickly with hands-on examples

**Contents:**
- 15-minute setup guide
- Minimal working MCP server code
- Step-by-step testing instructions
- Common pitfalls and how to avoid them
- Quick troubleshooting guide

**Best for:**
- First-time MCP developers
- Learning by doing
- Rapid prototyping
- Testing concepts

**Time to complete:** 15-30 minutes

---

### 2. **[LIGHTRAG_MCP_ARCHITECTURE.md](./LIGHTRAG_MCP_ARCHITECTURE.md)**

**Purpose:** Understand the system architecture and data flow

**Contents:**
- System architecture diagrams
- Communication protocol details (stdio, JSON-RPC)
- Complete user flow diagrams
- Data transformation pipeline
- Security architecture
- Performance considerations

**Best for:**
- Visual learners
- Understanding system design
- Planning implementation
- Debugging complex issues

**Time to read:** 20-30 minutes

---

### 3. **[LIGHTRAG_MCP_SPECIFICATION.md](./LIGHTRAG_MCP_SPECIFICATION.md)**

**Purpose:** Complete technical specification for production implementation

**Contents:**
- **Functional Requirements** - What the server must do
- **Non-Functional Requirements** - Performance, reliability, security
- **MCP Protocol Basics** - How JSON-RPC works over stdio
- **Implementation Guide** - Production-ready code patterns
- **Tool Schema Design** - JSON Schema for tool definitions
- **Error Handling** - Comprehensive error strategies
- **Testing Strategy** - Unit and integration tests
- **Security Considerations** - Input validation, sanitization
- **Deployment Checklist** - Pre/post-deployment steps

**Best for:**
- Building production systems
- Complete implementation reference
- Security and performance requirements
- Deployment planning

**Time to implement:** 4-8 hours for full production version

---

## 🎯 Recommended Reading Path

### For Rapid Learning (1 hour total):

1. **Quick Start** (15 min) → Build minimal working server
2. **Architecture** (20 min) → Understand how it works
3. **Specification** (25 min) → Skim sections relevant to your needs

### For Production Implementation (1 day):

1. **Architecture** (30 min) → Full understanding of system
2. **Specification** (1 hour) → Read completely, take notes
3. **Quick Start** (30 min) → Build and test minimal version
4. **Implementation** (5 hours) → Build production version following spec
5. **Testing** (2 hours) → Write and run tests

---

## 🚀 Quick Reference

### System Requirements

```bash
# Python 3.10+
python --version

# Required packages
pip install mcp pdfplumber aiohttp pydantic
```

### LibreChat Configuration

Add to `librechat.yaml`:

```yaml
endpoints:
  agents:
    mcpServers:
      lightrag:
        command: python
        args: ["-m", "lightrag_mcp.server"]
        env:
          LIGHTRAG_ENDPOINT: "http://localhost:8080"
        tools:
          - type: "all"
```

### Minimal Server Structure

```
lightrag-mcp-server/
├── src/
│   └── lightrag_mcp/
│       ├── __init__.py
│       ├── server.py          # Main MCP server
│       ├── tools.py           # Tool handlers
│       ├── pdf_processor.py   # PDF text extraction
│       └── lightrag_client.py # LightRAG API client
├── tests/
│   └── test_server.py
├── pyproject.toml
└── .env
```

---

## 🔑 Key Concepts

### What is MCP?

**Model Context Protocol (MCP)** is a standard protocol that connects AI assistants to external tools and data sources. Think of it like a universal plugin system for AI applications.

### How Does This Work?

```
User uploads PDF → LibreChat → MCP Server → LightRAG
                                   ↓
User asks question ← AI with context ← Query results
```

1. **User** attaches a PDF in LibreChat
2. **AI agent** (Claude/GPT-4) detects the intent to index
3. **LibreChat** encodes the file as base64
4. **MCP Server** (your code) receives the file
5. **Your code** extracts text and sends to LightRAG
6. **LightRAG** indexes the document
7. **User** can now query the indexed content

### Why Base64?

MCP uses **JSON-RPC** over **stdio** (stdin/stdout). JSON can't directly contain binary data, so we encode PDFs as base64 text strings.

```python
# PDF bytes → base64 string
pdf_bytes = b"\x25\x50\x44\x46..."  # Binary
encoded = "JVBERi0xLjQK..."          # Base64 text ✓ (JSON-safe)
```

---

## ✅ Success Criteria

Your implementation is complete when:

- [ ] MCP server starts without errors
- [ ] Tools appear in LibreChat's tool list
- [ ] Can upload PDF files (< 10 MB)
- [ ] Can upload Markdown files
- [ ] Text is correctly extracted from PDFs
- [ ] Documents are indexed in LightRAG
- [ ] Error messages are user-friendly
- [ ] No debug output to stdout (only stderr)
- [ ] Tests pass
- [ ] Deployed and monitored in production

---

## 🐛 Troubleshooting Quick Links

### Common Issues

| Problem | Document | Section |
|---------|----------|---------|
| Server won't start | Quick Start | "Debugging Tips" |
| Tools don't appear | Architecture | "Communication Flow" |
| JSON-RPC errors | Specification | "MCP Protocol Basics" |
| PDF extraction fails | Specification | "PDF Processing" |
| Base64 decode errors | Quick Start | "Testing Checklist" |
| LibreChat can't connect | Specification | "Troubleshooting Guide" |

---

## 📊 Expected Timeline

### Phase 1: Learning & Prototyping (2-4 hours)
- Read Quick Start
- Build minimal server
- Test with sample files
- Understand architecture

### Phase 2: Implementation (4-6 hours)
- Read full specification
- Implement production code
- Add error handling
- Write tests

### Phase 3: Integration (2-3 hours)
- Configure LibreChat
- Test end-to-end flow
- Debug issues
- Refine UX

### Phase 4: Deployment (2-3 hours)
- Set up production environment
- Deploy and monitor
- Performance tuning
- Documentation

**Total:** 10-16 hours for complete implementation

---

## 🎓 Learning Path

### If You're New to MCP:

1. Start with **Quick Start** → Build something working
2. Read **Architecture** → Understand what you built
3. Reference **Specification** → Fill in the gaps

### If You're Experienced with MCP:

1. Skim **Architecture** → Understand specifics
2. Read **Specification** → Get requirements
3. Reference **Quick Start** → Code examples

### If You're Building for Production:

1. Read **Specification** completely
2. Review **Architecture** for system design
3. Use **Quick Start** for code templates

---

## 💡 Pro Tips

### Development

```bash
# Always test manually first
python src/lightrag_mcp/server.py

# Use verbose logging during development
export LOG_LEVEL=DEBUG

# Test JSON-RPC communication
python test_manual.py
```

### Debugging

```python
# NEVER print to stdout (breaks JSON-RPC)
print("Debug")  # ❌ Wrong

# ALWAYS use logger (goes to stderr)
logger.info("Debug")  # ✅ Correct
```

### Performance

```python
# Don't block the event loop
time.sleep(5)  # ❌ Wrong

# Use async sleep
await asyncio.sleep(5)  # ✅ Correct
```

---

## 📞 Support

### When You Get Stuck:

1. **Check the troubleshooting section** in the Specification
2. **Review diagrams** in the Architecture guide
3. **Test manually** with the Quick Start examples
4. **Check logs** in stderr output

### Resources:

- **MCP SDK Docs:** https://github.com/modelcontextprotocol/python-sdk
- **LibreChat Docs:** https://docs.librechat.ai/
- **pdfplumber:** https://github.com/jsvine/pdfplumber
- **JSON-RPC 2.0 Spec:** https://www.jsonrpc.org/specification

---

## 🏁 Next Steps

**Ready to start?**

1. **Clone or download** this documentation package
2. **Read the Quick Start** → Get a working prototype in 15 minutes
3. **Review the Architecture** → Understand the system design
4. **Follow the Specification** → Build the production version

**Your journey:**

```
Quick Start → Working Prototype (15 min)
    ↓
Architecture Guide → Understanding (30 min)
    ↓
Technical Specification → Production Build (8 hours)
    ↓
Testing & Deployment → Live System (4 hours)
```

**Questions to ask yourself:**

- [ ] Do I understand what MCP is?
- [ ] Do I know how stdio communication works?
- [ ] Have I tested the minimal server?
- [ ] Do I understand the base64 workflow?
- [ ] Am I ready to implement the production version?

---

## 📝 Document Versions

| Document | Version | Last Updated |
|----------|---------|--------------|
| Quick Start | 1.0 | 2025-01-16 |
| Architecture | 1.0 | 2025-01-16 |
| Specification | 1.0 | 2025-01-16 |
| This README | 1.0 | 2025-01-16 |

---

## 🙏 Acknowledgments

This documentation was created for developers building LightRAG MCP servers for LibreChat, with no prior knowledge of the LibreChat codebase assumed.

**Technologies Used:**
- MCP (Model Context Protocol)
- Python 3.10+
- LibreChat
- LightRAG
- JSON-RPC 2.0

---

## 📄 License

This documentation is provided as-is for educational and development purposes.

---

**Happy Building! 🚀**

Start with [LIGHTRAG_MCP_QUICKSTART.md](./LIGHTRAG_MCP_QUICKSTART.md) to get your first MCP server running in 15 minutes.
