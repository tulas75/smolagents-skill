---
name: smolagents
description: |
  Build autonomous AI agents with Hugging Face's smolagents library. Use when:
  (1) Creating custom tools with @tool decorator or Tool class
  (2) Setting up CodeAgent or ToolCallingAgent
  (3) Configuring models (OpenAI, Anthropic, HuggingFace, local)
  (4) Building multi-agent hierarchies
  (5) Setting up secure code execution (Docker, E2B, etc.)
  (6) Debugging agent behavior or memory
  Triggers: "smolagents", "create agent", "create tool", "CodeAgent", "ToolCallingAgent", "agent tool"
---

# Smolagents Development

Smolagents builds autonomous agents that write and execute Python code to solve tasks.

## Quick Start

```python
from smolagents import CodeAgent, InferenceClientModel, tool

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city.

    Args:
        city: City name to get weather for.
    """
    return f"Weather in {city}: 72°F, sunny"

model = InferenceClientModel(model_id="Qwen/Qwen2.5-Coder-32B-Instruct")
agent = CodeAgent(tools=[get_weather], model=model)
result = agent.run("What's the weather in Paris?")
```

## Agent Selection

| Agent | Use When |
|-------|----------|
| **CodeAgent** | Complex reasoning, loops, multi-step logic. Writes Python code. 30% fewer LLM calls. |
| **ToolCallingAgent** | Simple tool use, JSON-based. Better with models optimized for tool calling. |

Default to **CodeAgent** unless the model lacks code generation ability.

## Tool Creation

### Decorator (simple tools)

```python
from smolagents import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """Search the database for matching records.

    Args:
        query: Search query string.
        limit: Maximum results to return.
    """
    # Implementation
    return results
```

### Class (complex tools with state)

```python
from smolagents import Tool

class DatabaseTool(Tool):
    name = "database_search"
    description = "Search database for records. Returns matching entries as formatted string."
    inputs = {
        "query": {"type": "string", "description": "Search query"},
        "limit": {"type": "integer", "description": "Max results", "nullable": True},
    }
    output_type = "string"

    def __init__(self, connection_string: str, **kwargs):
        super().__init__(**kwargs)
        self.db = connect(connection_string)

    def forward(self, query: str, limit: int = 10) -> str:
        results = self.db.search(query, limit=limit)
        return "\n".join(str(r) for r in results)
```

**Tool requirements:**
- Docstring/description must explain what tool does AND what it returns
- All parameters need type hints and descriptions
- `output_type`: string, integer, number, boolean, image, audio, object, array, any

See [references/tools.md](references/tools.md) for advanced patterns.

## Model Configuration

```python
# HuggingFace Inference API (recommended start)
from smolagents import InferenceClientModel
model = InferenceClientModel(model_id="Qwen/Qwen2.5-Coder-32B-Instruct")

# OpenAI
from smolagents import OpenAIModel
model = OpenAIModel(model_id="gpt-4o")

# Anthropic (via LiteLLM)
from smolagents import LiteLLMModel
model = LiteLLMModel(model_id="anthropic/claude-sonnet-4-20250514")

# Local with Ollama
model = LiteLLMModel(model_id="ollama/llama3.2")
```

See [references/models.md](references/models.md) for all providers.

## Agent Configuration

```python
agent = CodeAgent(
    tools=[tool1, tool2],
    model=model,
    # Key options:
    max_steps=20,                    # Max reasoning steps (default: 20)
    verbosity_level=1,               # 0=errors, 1=info, 2=debug
    stream_outputs=True,             # Stream results live
    executor_type="local",           # Code execution: local, docker, e2b
    additional_authorized_imports=["pandas", "numpy"],  # Extra allowed imports
)

# Get detailed results
result = agent.run("task", return_full_result=True)
print(result.output)        # Final answer
print(result.token_usage)   # TokenUsage(input, output, total)
print(result.timing)        # Execution timing
```

See [references/agents.md](references/agents.md) for multi-agent patterns and memory.

## Secure Execution

For production, use sandboxed execution:

```python
# Docker (recommended for self-hosted)
agent = CodeAgent(tools=[...], model=model, executor_type="docker")

# E2B (cloud sandbox)
agent = CodeAgent(tools=[...], model=model, executor_type="e2b")
```

See [references/executors.md](references/executors.md) for all options.

## Common Patterns

### Web Research Agent

```python
from smolagents import CodeAgent, InferenceClientModel
from smolagents.default_tools import WebSearchTool, VisitWebpageTool

agent = CodeAgent(
    tools=[WebSearchTool(), VisitWebpageTool()],
    model=InferenceClientModel(),
)
result = agent.run("Research the latest developments in quantum computing")
```

### RAG Agent

```python
from smolagents import CodeAgent, Tool

class RetrieverTool(Tool):
    name = "search_docs"
    description = "Search documentation. Returns relevant excerpts."
    inputs = {"query": {"type": "string", "description": "Search query"}}
    output_type = "string"

    def __init__(self, retriever, **kwargs):
        super().__init__(**kwargs)
        self.retriever = retriever

    def forward(self, query: str) -> str:
        docs = self.retriever.invoke(query)
        return "\n\n".join(d.page_content for d in docs[:3])

agent = CodeAgent(tools=[RetrieverTool(my_retriever)], model=model)
```

### Multi-Agent System

```python
from smolagents import CodeAgent, ToolCallingAgent

# Specialist agent
researcher = ToolCallingAgent(
    tools=[WebSearchTool()],
    model=model,
    name="researcher",
    description="Searches the web for information",
)

# Manager coordinates specialists
manager = CodeAgent(
    tools=[],
    model=model,
    managed_agents=[researcher],
)

result = manager.run("Find and summarize recent AI safety papers")
```

## Debugging

```python
# Enable verbose logging
agent = CodeAgent(tools=[...], model=model, verbosity_level=2)

# Inspect execution steps
result = agent.run("task", return_full_result=True)
for step in result.steps:
    if hasattr(step, 'tool_calls'):
        print(f"Step {step.step_number}: {step.tool_calls}")
        print(f"Observation: {step.observations}")

# Check memory
for msg in agent.memory.get_messages():
    print(msg["role"], msg.get("content", "")[:100])
```
