# LibreChat Agent Patterns: Complete Usage Guide

## Table of Contents
1. [Overview](#overview)
2. [Pattern Types](#pattern-types)
3. [When to Use Each Pattern](#when-to-use-each-pattern)
4. [Sequential Agent Flow](#sequential-agent-flow)
5. [Orchestrator Pattern](#orchestrator-pattern)
6. [Advanced Configurations](#advanced-configurations)
7. [Best Practices](#best-practices)
8. [Common Pitfalls](#common-pitfalls)
9. [Performance Optimization](#performance-optimization)
10. [Examples](#examples)

---

## Overview

LibreChat implements a **graph-based multi-agent execution system** powered by LangGraph. It supports two primary agentic patterns:

1. **Sequential Flow (Direct Edges)**: Agents execute in a predefined sequence
2. **Orchestrator Pattern (Handoff Edges)**: A supervisor agent dynamically routes to specialist agents

Both patterns are configured using the `edges` property on the primary agent.

---

## Pattern Types

### 1. Sequential Flow (`edgeType: 'direct'`)

**What it does:**
- Agents execute in a strict, predefined order
- Each agent processes the output of the previous agent
- Control flows automatically from one agent to the next

**How it works:**
- Previous agent's output becomes context for the next agent
- Supports prompt templates to format handoff context
- Can exclude raw results and only pass formatted prompts
- Allows parallel execution when multiple destinations specified

**Configuration:**
```typescript
edges: [
  {
    from: "agent1",
    to: "agent2",
    edgeType: "direct",
    prompt: "Process this further: {convo}",  // Optional template
    excludeResults: true,  // Only pass formatted prompt, not raw output
    description: "Sequential handoff from agent1 to agent2"
  },
  {
    from: "agent2",
    to: "agent3",
    edgeType: "direct"
  }
]
```

### 2. Orchestrator Pattern (`edgeType: 'handoff'`)

**What it does:**
- A supervisor/orchestrator agent dynamically decides which specialist to use
- Creates tools that the orchestrator can invoke to transfer control
- Runtime decision-making based on task requirements

**How it works:**
- Each handoff edge creates a tool available to the orchestrator
- The `description` field helps the LLM understand when to use each handoff
- The orchestrator invokes the tool when it determines a specialist is needed
- The `prompt` parameter allows passing specific instructions to the specialist

**Configuration:**
```typescript
edges: [
  {
    from: "orchestrator",
    to: "code_specialist",
    edgeType: "handoff",
    description: "Use this agent for coding tasks, debugging, and technical implementation",
    prompt: "Implement the following requirement based on the discussion",
    promptKey: "instructions"  // Parameter name in handoff tool (default: "instructions")
  },
  {
    from: "orchestrator",
    to: "research_specialist",
    edgeType: "handoff",
    description: "Use this agent for research, information gathering, and analysis",
    prompt: "Research the following topic in detail"
  }
]
```

---

## When to Use Each Pattern

### Use Sequential Flow When:

✅ **The workflow is predictable and linear**
- Document review → summarization → recommendations
- Data collection → analysis → reporting
- Research → writing → editing

✅ **Each step builds on the previous one**
- You need structured, progressive refinement
- Output quality improves with each specialized stage

✅ **You want deterministic behavior**
- Same input should follow the same path
- Predictable cost and execution time

✅ **Task decomposition is clear**
- You know exactly which specialist handles which step
- No decision-making required between steps

### Use Orchestrator Pattern When:

✅ **Tasks require dynamic routing**
- User requests vary significantly
- Different specialists needed based on content
- Cannot predict the workflow in advance

✅ **You need flexible problem-solving**
- Complex tasks requiring multiple specialist consultations
- The orchestrator can determine best approach
- May need to consult multiple specialists for one task

✅ **Efficiency over predictability**
- Skip unnecessary steps
- Only invoke specialists when needed
- Adaptive workflows

✅ **Interactive or iterative work**
- User may ask follow-up questions requiring different specialists
- Multi-turn conversations with specialist switching

---

## Sequential Agent Flow

### Basic Configuration

```typescript
{
  "id": "document_pipeline",
  "name": "Document Processing Pipeline",
  "description": "Analyzes, summarizes, and generates recommendations",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "edges": [
    {
      "from": "analyzer",
      "to": "summarizer",
      "edgeType": "direct",
      "description": "Pass analysis to summarizer"
    },
    {
      "from": "summarizer",
      "to": "recommender",
      "edgeType": "direct",
      "description": "Pass summary to recommender"
    }
  ]
}
```

### Advanced: Custom Prompts for Context Shaping

The `prompt` field can be a template string or function that formats the previous agent's output:

```typescript
{
  "from": "researcher",
  "to": "writer",
  "edgeType": "direct",
  "prompt": `Based on the research findings below, write a comprehensive article:

{convo}

Guidelines:
- Use clear, accessible language
- Include specific examples from the research
- Structure with introduction, body, conclusion`,
  "excludeResults": true  // Only send the formatted prompt, not raw research output
}
```

**Key Parameters:**
- `prompt`: Template string where `{convo}` is replaced with previous agent's output
- `excludeResults`: When `true`, only the formatted prompt is passed (cleaner context)
- `description`: Human-readable description (good for documentation)

### Parallel Execution in Sequential Flow

You can fan out to multiple agents:

```typescript
{
  "from": "orchestrator",
  "to": ["agent1", "agent2", "agent3"],
  "edgeType": "direct",
  "description": "Execute all three agents in parallel"
}
```

This allows concurrent processing when agents don't depend on each other's outputs.

### Output Control

Control what the user sees with `hide_sequential_outputs`:

```typescript
{
  "id": "pipeline",
  "hide_sequential_outputs": true,  // Only show final output
  "edges": [
    // ... edges
  ]
}
```

**When `true`:**
- User only sees the final agent's output
- Intermediate agent outputs are hidden
- Tool calls are still visible
- Cleaner UX for multi-step pipelines

**When `false`:**
- All agent outputs visible
- User sees the full reasoning chain
- Better for transparency and debugging

---

## Orchestrator Pattern

### Basic Configuration

```typescript
{
  "id": "supervisor",
  "name": "AI Supervisor",
  "description": "Routes tasks to appropriate specialists",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "instructions": "You are a supervisor AI. Analyze user requests and delegate to the appropriate specialist agent. Use the handoff tools when you need specialized help.",
  "edges": [
    {
      "from": "supervisor",
      "to": "coder",
      "edgeType": "handoff",
      "description": "Use for programming, debugging, code review, and technical implementation tasks",
      "prompt": "Implement the following based on our discussion",
      "promptKey": "task"
    },
    {
      "from": "supervisor",
      "to": "researcher",
      "edgeType": "handoff",
      "description": "Use for research, fact-finding, information gathering, and analysis tasks",
      "prompt": "Research the following topic thoroughly",
      "promptKey": "topic"
    },
    {
      "from": "supervisor",
      "to": "writer",
      "edgeType": "handoff",
      "description": "Use for content creation, writing, editing, and creative tasks",
      "prompt": "Write content based on these requirements",
      "promptKey": "requirements"
    }
  ]
}
```

### How Handoffs Work

1. **Tool Creation**: Each handoff edge creates a tool available to the orchestrator
2. **Decision Making**: The orchestrator's LLM decides when to invoke handoff tools
3. **Context Passing**: The `prompt` field defines what context to pass to the specialist
4. **Execution**: The specialist agent processes the task and returns control
5. **Continuation**: The orchestrator receives the specialist's output and can continue

### Writing Effective Descriptions

The `description` field is **critical** for handoff edges because it helps the orchestrator decide when to use each specialist:

**❌ Poor descriptions:**
```typescript
description: "Code agent"  // Too vague
description: "Use this agent"  // Not helpful
description: "Agent 2"  // No context
```

**✅ Good descriptions:**
```typescript
description: "Use for writing, debugging, and reviewing Python, JavaScript, and TypeScript code. Handles implementation of features, bug fixes, and code optimization."

description: "Use for researching technical documentation, finding examples, analyzing codebases, and gathering information from external sources."

description: "Use for creative content writing, documentation, user guides, blog posts, and marketing copy. Specializes in clear, engaging prose."
```

**Best practices for descriptions:**
- Be specific about capabilities
- Include example use cases
- Mention tools/resources the specialist has access to
- Use action verbs (writing, debugging, researching, analyzing)
- Define the specialist's domain clearly

### Crafting Orchestrator Instructions

The orchestrator's `instructions` field guides its delegation behavior:

```typescript
{
  "instructions": `You are a supervisor AI that coordinates specialist agents.

Your responsibilities:
1. Analyze user requests to identify which specialist(s) are needed
2. Delegate tasks using handoff tools when specialized expertise is required
3. Synthesize specialist outputs into coherent responses
4. Handle simple queries yourself without delegation

Guidelines:
- Use the code specialist for implementation tasks
- Use the researcher for factual information and analysis
- Use the writer for content creation
- You can consult multiple specialists for complex tasks
- Always explain which specialist you're consulting and why

When delegating, provide clear, specific instructions to the specialist.`
}
```

### Multiple Handoffs Per Task

The orchestrator can consult multiple specialists:

```typescript
// User: "Research the best JavaScript frameworks and write a comparison article"

// Orchestrator flow:
1. Invokes researcher: "Research popular JavaScript frameworks, their features, and use cases"
2. Receives research results
3. Invokes writer: "Write a comprehensive comparison article about JavaScript frameworks based on this research: [research results]"
4. Synthesizes final response
```

### Handoff Limitations

**Maximum handoffs per agent: 10**

This is hardcoded in the implementation (`MAX_HANDOFFS = 10` in `AgentHandoffs.tsx`). If you need more than 10 specialists, consider:
- Grouping similar specialists
- Creating sub-orchestrators
- Using sequential chains for predictable portions

---

## Advanced Configurations

### 1. Recursion Limits

Control maximum execution steps to prevent infinite loops and manage costs:

```typescript
{
  "recursion_limit": 50,  // Agent-specific limit
}
```

**In global config (`librechat.yaml`):**
```yaml
endpoints:
  agents:
    maxRecursionLimit: 100  # System-wide cap
```

**What counts as a "step":**
- 1 LLM call = 1 step
- 1 tool call round = 1 step
- Example: Basic tool interaction = 3 steps (request, tool use, follow-up)

**Recommendations:**
- Simple agents: 10-25 steps
- Complex reasoning: 50-100 steps
- Multi-agent systems: 100+ steps
- Monitor actual usage and adjust

### 2. Tool Termination

Stop execution after tool calls complete:

```typescript
{
  "end_after_tools": true
}
```

**Use cases:**
- Action-oriented agents (e.g., "send email", "create ticket")
- When tool output is sufficient
- Preventing unnecessary LLM calls after tool execution

### 3. Conditional Routing

Add logic-based routing with the `condition` parameter:

```typescript
{
  "from": "analyzer",
  "to": ["positive_handler", "negative_handler"],
  "edgeType": "direct",
  "condition": (state) => {
    const sentiment = state.messages.slice(-1)[0].content;
    return sentiment.includes("positive") ? "positive_handler" : "negative_handler";
  }
}
```

**Condition return types:**
- `boolean`: Follow edge if true
- `string`: Route to specific destination
- `string[]`: Route to multiple destinations

### 4. Context Formatting

Shape the context passed between agents:

```typescript
{
  "from": "data_collector",
  "to": "analyst",
  "edgeType": "direct",
  "prompt": (messages, runStartIndex) => {
    const dataMessages = messages.slice(runStartIndex);
    const bufferString = getBufferString(dataMessages);
    return `Analyze this data:

${bufferString}

Focus on:
- Trends and patterns
- Anomalies
- Actionable insights`;
  },
  "excludeResults": true
}
```

### 5. Provider-Specific Considerations

**Google/Vertex AI:**
```typescript
// ❌ Cannot combine custom tools with structured tools
{
  "provider": "google",
  "tools": ["custom_tool", "tavily_search"],  // Will throw conflict error
}

// ✅ Use one type only
{
  "provider": "google",
  "tools": ["tavily_search", "web_browser"],  // All structured tools
}
```

**OpenAI/Anthropic/Azure:**
```typescript
// ✅ Can mix both types
{
  "provider": "anthropic",
  "tools": ["custom_tool", "tavily_search"],  // Works fine
}
```

---

## Best Practices

### 1. Agent Design

**Specialized Agents:**
- Give each agent a clear, specific role
- Define expertise boundaries in instructions
- Include relevant tools for their domain
- Use appropriate models for the task complexity

**Orchestrator Agent:**
- Use a strong reasoning model (GPT-4, Claude Sonnet, etc.)
- Provide clear delegation guidelines
- Explain when to use each specialist
- Include examples in instructions

### 2. Instructions Crafting

**For Sequential Agents:**
```typescript
{
  "instructions": "You are step 2 in a pipeline. You receive analyzed data and must summarize it. Focus on key insights and trends. Keep summaries concise but comprehensive."
}
```

**For Orchestrator:**
```typescript
{
  "instructions": "You coordinate specialists. Analyze requests, delegate to appropriate experts, and synthesize their outputs. Explain your delegation decisions to users."
}
```

**For Specialists:**
```typescript
{
  "instructions": "You are a Python coding specialist. You receive implementation requests from the supervisor. Write clean, well-documented code. Explain your approach and any trade-offs."
}
```

### 3. Edge Configuration

**Use `excludeResults: true` when:**
- Previous agent's raw output is verbose
- You want to reframe the context for the next agent
- The prompt template provides sufficient context

**Use `excludeResults: false` when:**
- Next agent needs full conversation history
- Context details are important
- Debugging or transparency is needed

### 4. Output Visibility

**Use `hide_sequential_outputs: true` when:**
- Creating a streamlined user experience
- Intermediate steps are not interesting to users
- Building a "black box" pipeline

**Use `hide_sequential_outputs: false` when:**
- Users want transparency
- Debugging the pipeline
- Each step provides valuable insights
- Building trust through visible reasoning

### 5. Model Selection

**Orchestrator:** Strong reasoning models
- GPT-4, GPT-4 Turbo
- Claude 3.5 Sonnet, Claude 3 Opus
- Gemini Pro/Ultra

**Specialists:** Task-appropriate models
- Complex tasks: GPT-4, Claude Sonnet
- Simple tasks: GPT-3.5 Turbo, Claude Haiku
- Coding: GPT-4, Claude Sonnet
- Creative writing: Claude Opus, GPT-4

**Sequential chains:** Balance cost and quality
- First agent (analysis): Stronger model
- Middle agents: Medium models
- Final agent (presentation): Stronger model

### 6. Testing Strategy

**Start simple:**
1. Test each agent individually
2. Test two-agent edges
3. Build up to full system
4. Monitor recursion usage

**Debugging tools:**
- Set `hide_sequential_outputs: false` initially
- Use lower `recursion_limit` during testing
- Check agent labels in conversation history
- Monitor token usage per agent

### 7. Cost Optimization

**Reduce costs by:**
- Using smaller models for simple agents
- Setting appropriate recursion limits
- Using `excludeResults: true` to reduce context size
- Hiding sequential outputs (fewer tokens streamed)
- Caching agent instructions when possible

**Monitor:**
- Token usage per agent (shown in UI)
- Total recursion steps
- Which agents are called most frequently

---

## Common Pitfalls

### 1. ❌ Circular Dependencies

**Problem:**
```typescript
edges: [
  { from: "agent1", to: "agent2", edgeType: "direct" },
  { from: "agent2", to: "agent1", edgeType: "direct" }  // Creates loop!
]
```

**Solution:**
- Design acyclic flows
- Use recursion limits as a safety net
- For iterative workflows, use a single agent with tools

### 2. ❌ Vague Handoff Descriptions

**Problem:**
```typescript
{
  description: "Use this agent",  // LLM won't know when to use it
}
```

**Solution:**
```typescript
{
  description: "Use for analyzing financial data, calculating metrics, and generating reports with charts and tables. Has access to data analysis tools and spreadsheet capabilities.",
}
```

### 3. ❌ Using Wrong Edge Type

**Problem:**
```typescript
// Want dynamic routing but using direct edges
edges: [
  { from: "supervisor", to: "specialist1", edgeType: "direct" },
  { from: "supervisor", to: "specialist2", edgeType: "direct" },
]
// This will execute BOTH specialists every time!
```

**Solution:**
```typescript
// Use handoff for dynamic routing
edges: [
  { from: "supervisor", to: "specialist1", edgeType: "handoff" },
  { from: "supervisor", to: "specialist2", edgeType: "handoff" },
]
```

### 4. ❌ Insufficient Recursion Limit

**Problem:**
```typescript
{
  "recursion_limit": 5,  // Too low for multi-agent system
  "edges": [/* multiple agents */]
}
// Execution stops prematurely
```

**Solution:**
- Start with higher limits (50-100)
- Monitor actual usage
- Adjust based on empirical data

### 5. ❌ Over-complicated Orchestration

**Problem:**
Creating 15 specialists for tasks that could be handled by 3-4.

**Solution:**
- Group related capabilities
- Use general agents with good instructions
- Reserve specialists for truly distinct domains

### 6. ❌ Missing Instructions for Orchestrator

**Problem:**
```typescript
{
  "id": "orchestrator",
  "instructions": "",  // No guidance on when to delegate
  "edges": [/* handoffs */]
}
```

**Solution:**
Always provide clear orchestration guidelines:
```typescript
{
  "instructions": "You coordinate specialists. Delegate to:\n- Code specialist for implementation\n- Researcher for information gathering\n- Writer for content creation"
}
```

### 7. ❌ Not Considering Context Size

**Problem:**
Long sequential chains with `excludeResults: false` create massive contexts.

**Solution:**
- Use `excludeResults: true` with targeted prompts
- Summarize at intermediate steps
- Monitor token usage

### 8. ❌ Google Provider Tool Conflicts

**Problem:**
```typescript
{
  "provider": "google",
  "tools": ["custom_tool", "tavily_search"],  // Error!
}
```

**Solution:**
```typescript
{
  "provider": "google",
  "tools": ["tavily_search", "web_browser"],  // Only structured tools
}
```

---

## Performance Optimization

### 1. Minimize Sequential Chain Length

**Inefficient:**
```
Input → Agent1 → Agent2 → Agent3 → Agent4 → Agent5 → Agent6 → Output
```

**Optimized:**
```
Input → Agent1 (does 1+2) → Agent2 (does 3+4+5) → Agent3 (does 6) → Output
```

Combine related steps into single capable agents.

### 2. Use Appropriate Models

**Don't do this:**
```typescript
// Using GPT-4 for simple formatting
{
  "id": "formatter",
  "model": "gpt-4",
  "instructions": "Format the text as markdown"
}
```

**Do this:**
```typescript
{
  "id": "formatter",
  "model": "gpt-3.5-turbo",  // Fast and cheap for simple tasks
  "instructions": "Format the text as markdown"
}
```

### 3. Optimize Context Passing

**Before:**
```typescript
{
  "from": "agent1",
  "to": "agent2",
  "edgeType": "direct",
  "excludeResults": false  // Passes 10,000 tokens of conversation
}
```

**After:**
```typescript
{
  "from": "agent1",
  "to": "agent2",
  "edgeType": "direct",
  "prompt": "Extract key points: {convo}",
  "excludeResults": true  // Passes 500 tokens of extracted points
}
```

### 4. Set Realistic Recursion Limits

Monitor actual usage:
- Check how many steps your workflow typically uses
- Set limit to 150-200% of typical usage
- Prevents runaway costs while allowing flexibility

### 5. Cache When Possible

Use consistent agent IDs and instructions:
- Many providers cache system messages
- Reusing same agents = faster subsequent calls
- Update instructions sparingly

---

## Examples

### Example 1: Document Processing Pipeline (Sequential)

**Use case:** Analyze uploaded documents, extract insights, generate summaries.

```json
{
  "id": "doc_pipeline",
  "name": "Document Processing Pipeline",
  "description": "Processes documents through analysis, insight extraction, and summarization",
  "provider": "openai",
  "model": "gpt-4-turbo-preview",
  "hide_sequential_outputs": true,
  "recursion_limit": 30,
  "instructions": "You are the document analyzer. Extract key information, entities, and themes from documents.",
  "tools": ["file_reader"],
  "edges": [
    {
      "from": "doc_pipeline",
      "to": "insight_extractor",
      "edgeType": "direct",
      "prompt": "Based on this document analysis:\n\n{convo}\n\nExtract actionable insights and key takeaways.",
      "excludeResults": true,
      "description": "Pass analysis to insight extractor"
    },
    {
      "from": "insight_extractor",
      "to": "summarizer",
      "edgeType": "direct",
      "prompt": "Create a concise summary including these insights:\n\n{convo}",
      "excludeResults": true,
      "description": "Pass insights to summarizer"
    }
  ]
}
```

**Additional agents:**
```json
{
  "id": "insight_extractor",
  "name": "Insight Extractor",
  "provider": "openai",
  "model": "gpt-4-turbo-preview",
  "instructions": "Extract actionable insights from analysis. Focus on practical takeaways."
}
```

```json
{
  "id": "summarizer",
  "name": "Summarizer",
  "provider": "openai",
  "model": "gpt-3.5-turbo",
  "instructions": "Create concise, well-structured summaries. Use bullet points and clear sections."
}
```

### Example 2: Customer Support Orchestrator (Handoff)

**Use case:** Route customer inquiries to appropriate specialists.

```json
{
  "id": "support_supervisor",
  "name": "Customer Support Supervisor",
  "description": "Routes customer inquiries to specialized agents",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "recursion_limit": 50,
  "instructions": "You are a customer support supervisor. Analyze customer inquiries and delegate to appropriate specialists:\n\n- Technical Support: For product issues, bugs, troubleshooting\n- Billing Support: For payment, subscription, invoice questions\n- Account Support: For login, settings, profile issues\n\nAlways explain which specialist you're consulting and why. Synthesize specialist responses into helpful customer replies.",
  "edges": [
    {
      "from": "support_supervisor",
      "to": "technical_support",
      "edgeType": "handoff",
      "description": "Use for technical issues, product bugs, troubleshooting, feature questions, and error messages. Has access to technical documentation and debugging tools.",
      "prompt": "Help this customer with their technical issue",
      "promptKey": "customer_issue"
    },
    {
      "from": "support_supervisor",
      "to": "billing_support",
      "edgeType": "handoff",
      "description": "Use for billing questions, payment issues, subscription management, refund requests, and invoice inquiries. Has access to billing system and payment tools.",
      "prompt": "Assist with this billing inquiry",
      "promptKey": "billing_question"
    },
    {
      "from": "support_supervisor",
      "to": "account_support",
      "edgeType": "handoff",
      "description": "Use for account access issues, password resets, profile settings, email changes, and account security. Has access to account management tools.",
      "prompt": "Help with this account issue",
      "promptKey": "account_issue"
    }
  ]
}
```

**Specialist agents:**
```json
{
  "id": "technical_support",
  "name": "Technical Support Specialist",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "instructions": "You are a technical support specialist. Troubleshoot issues, explain solutions clearly, and provide step-by-step guidance. Use technical documentation when available.",
  "tools": ["web_search", "documentation_search"]
}
```

```json
{
  "id": "billing_support",
  "name": "Billing Support Specialist",
  "provider": "openai",
  "model": "gpt-4",
  "instructions": "You are a billing support specialist. Help customers with payment issues, explain billing, and process refund requests. Be empathetic and clear about financial matters.",
  "tools": ["billing_system"]
}
```

```json
{
  "id": "account_support",
  "name": "Account Support Specialist",
  "provider": "openai",
  "model": "gpt-3.5-turbo",
  "instructions": "You are an account support specialist. Help with login issues, profile settings, and account security. Guide users through solutions step-by-step.",
  "tools": ["account_management"]
}
```

### Example 3: Research & Writing Pipeline (Hybrid)

**Use case:** Research a topic, then write content based on findings.

```json
{
  "id": "research_writer",
  "name": "Research & Writing Orchestrator",
  "description": "Coordinates research and writing for content creation",
  "provider": "openai",
  "model": "gpt-4",
  "recursion_limit": 75,
  "hide_sequential_outputs": false,
  "instructions": "You coordinate research and writing. First, delegate research to gather information. Then, delegate writing to create content based on research findings. Synthesize the final output.",
  "edges": [
    {
      "from": "research_writer",
      "to": "researcher",
      "edgeType": "handoff",
      "description": "Use for researching topics, gathering information, finding sources, and fact-checking. Has access to web search and can analyze multiple sources.",
      "prompt": "Research this topic thoroughly and provide comprehensive findings",
      "promptKey": "research_topic"
    },
    {
      "from": "researcher",
      "to": "writer",
      "edgeType": "direct",
      "prompt": "Write comprehensive content based on this research:\n\n{convo}\n\nCreate engaging, well-structured content with introduction, body, and conclusion.",
      "excludeResults": false,
      "description": "Pass research findings to writer"
    }
  ]
}
```

**Why hybrid?**
- Handoff from orchestrator to researcher (dynamic - only research if needed)
- Direct edge from researcher to writer (sequential - always write after research)

### Example 4: Code Review System (Sequential with Parallel)

**Use case:** Review code through multiple lenses simultaneously.

```json
{
  "id": "code_reviewer",
  "name": "Code Review Coordinator",
  "description": "Coordinates comprehensive code review",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "recursion_limit": 40,
  "hide_sequential_outputs": false,
  "instructions": "You coordinate code review. First, analyze the code for bugs and logic issues. Then trigger parallel reviews for security, performance, and style.",
  "edges": [
    {
      "from": "code_reviewer",
      "to": ["security_reviewer", "performance_reviewer", "style_reviewer"],
      "edgeType": "direct",
      "description": "Parallel review by all specialists",
      "excludeResults": false
    },
    {
      "from": ["security_reviewer", "performance_reviewer", "style_reviewer"],
      "to": "report_synthesizer",
      "edgeType": "direct",
      "prompt": "Synthesize these review findings into a comprehensive report:\n\n{convo}",
      "excludeResults": false,
      "description": "Combine all reviews into final report"
    }
  ]
}
```

### Example 5: Data Analysis Pipeline (Sequential)

**Use case:** Collect data, clean it, analyze it, visualize results.

```json
{
  "id": "data_pipeline",
  "name": "Data Analysis Pipeline",
  "description": "End-to-end data analysis workflow",
  "provider": "openai",
  "model": "gpt-4",
  "hide_sequential_outputs": true,
  "recursion_limit": 60,
  "edges": [
    {
      "from": "data_pipeline",
      "to": "data_cleaner",
      "edgeType": "direct",
      "prompt": "Clean and prepare this data for analysis:\n\n{convo}",
      "excludeResults": true
    },
    {
      "from": "data_cleaner",
      "to": "analyzer",
      "edgeType": "direct",
      "prompt": "Analyze this cleaned data and identify trends, patterns, and insights:\n\n{convo}",
      "excludeResults": true
    },
    {
      "from": "analyzer",
      "to": "visualizer",
      "edgeType": "direct",
      "prompt": "Create visualizations for these analysis results:\n\n{convo}\n\nGenerate charts, graphs, and visual summaries.",
      "excludeResults": false
    }
  ]
}
```

---

## Quick Reference

### Edge Types Comparison

| Feature | Sequential (`direct`) | Orchestrator (`handoff`) |
|---------|----------------------|--------------------------|
| Execution | Automatic, predefined | Dynamic, LLM-decided |
| Routing | Fixed path | Runtime decision |
| Context passing | Previous output | Custom prompt |
| Use case | Linear workflows | Adaptive workflows |
| Predictability | High | Lower |
| Flexibility | Low | High |
| Cost control | Easier | Harder |
| Setup complexity | Low | Medium |

### Configuration Checklist

**For any agent:**
- [ ] Clear, specific instructions
- [ ] Appropriate model for task complexity
- [ ] Reasonable recursion_limit
- [ ] Relevant tools assigned

**For sequential flows:**
- [ ] Logical progression of steps
- [ ] Appropriate use of excludeResults
- [ ] Context formatting via prompt templates
- [ ] Consider hide_sequential_outputs for UX

**For orchestrators:**
- [ ] Strong reasoning model
- [ ] Clear delegation guidelines in instructions
- [ ] Detailed handoff descriptions
- [ ] Appropriate promptKey naming
- [ ] Max 10 handoffs per agent

**For specialists (in orchestrator pattern):**
- [ ] Domain-specific instructions
- [ ] Relevant tools for their specialty
- [ ] Clear scope definition
- [ ] Appropriate model (can be smaller/cheaper)

---

## Additional Resources

**Key Implementation Files:**
- `/packages/data-schemas/src/schema/agent.ts` - Agent schema
- `/packages/data-provider/src/types/agents.ts` - GraphEdge types
- `/packages/api/src/agents/run.ts` - Run creation
- `/packages/api/src/agents/chain.ts` - Sequential helpers

**Configuration:**
- `librechat.yaml` - Global agent settings
- `maxRecursionLimit` - System-wide step limit
- `allowedProviders` - Which LLM providers to enable

**UI:**
- Agent Builder in LibreChat UI
- Advanced settings for edge configuration
- Handoff management interface

---

## Conclusion

LibreChat's agent system provides powerful, flexible patterns for building sophisticated multi-agent workflows:

- **Sequential flows** for predictable, linear processes
- **Orchestrator pattern** for adaptive, intelligent routing
- **Hybrid approaches** combining both patterns
- **Production-ready** with streaming, error handling, and cost controls

Start with simple configurations, test thoroughly, and scale up as you understand the patterns. Monitor token usage and recursion limits to optimize costs. Most importantly, choose the pattern that matches your workflow's natural structure.

Happy agent building! 🤖
